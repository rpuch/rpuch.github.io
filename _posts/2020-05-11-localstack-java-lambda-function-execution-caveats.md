---
layout: post
title: Localstack Java Lambda Function Execution Caveats
date: '2020-05-11T15:43:00.000+04:00'
author: rpuchkovskiy
tags:
- localstack
- lambda function execution model
- lambda function cleanup
excerpt_separator: <!--more-->
---

[Localstack](https://github.com/localstack/localstack) is an implementation of AWS cloud stack which makes it very
convenient if you want to develop and debug your AWS components locally. Also, it enables the creation of integration
tests of your Lambda Functions: you just start Localstack (available as
[Docker containers](https://hub.docker.com/r/localstack/localstack/), so
[Testcontainers](https://www.testcontainers.org/modules/localstack/) can be used). Such tests can then be run without
a necessity to communicate with real AWS (which would make the tests more fragile, slower and add complexity).

But it turns out that Localstack's Lambda Function execution model (at least when it comes to Java functions) is
different from AWS's, which may lead you to a problematic behavior of your Lambdas.

<!--more-->

## Localstack Lambda Function Execution Model

Localstack currently offers three lambda function executors:

* **local** - executes the function in the same machine in which Localstack is run
* **docker** (default) - each function invocation is made in a fresh Docker container; it is removed after the
invocation returns
* **docker-reuse** - a Docker container is created per function; each invocation is processed inside the corresponding
container

Despite their differences, all the executors execute a lambda function in the following way:

1. Lambda jar is copied/uploaded to the executor (if needed) along with a special 'runner' jar
2. The 'runner' jar is run passing it the Lambda jar as a parameter
3. Lambda function arguments, env and input are passed in as command line arguments
4. The 'runner' jar invokes the Lambda function
5. When the function invocation terminates (successfully or not), it dumps the function result to STDERR and exits
6. Localstack waits for the 'runner' process to terminate
7. When it is terminated, Localstack analyzes the exit code (and errors out if it's not 0) and obtains the function
result from the STDERR of the 'runner' process as long as the Function STDOUT/STDERR output
8. Localstack logs the output of the function to its own STDOUT (this does not happen for **docker-reuse** executor
for some reason)

Note two things:

1. in Localstack, *every Lambda function instance is effectively one-off*
2. function stdout/stderr output will not be seen by Localstack until the invocation is finished

## The problem

AWS Lambda documentation encourages us to initialize heavy resources once so that, when AWS sends consequtive
invocations to the same Lambda Function instance, we could save on initialization.

Imagine that in our function we create some connection that includes a worker (non-demonic) thread. Let's look what
happens when we invoke such a function in Localstack.

A Java process gets terminated when all non-demonic threads terminate. So, on step 5, the 'runner' process will never
terminate if we don't terminate the worker thread! And we should not
terminate it as it will require its resource reinitialization on next invocation in AWS! So Localstack at step 6
will never see the 'runner' process termination (only timeout will kill it), which means that synchronous invocations
will be blocked forever.

What makes the situation even more frustrating is that you will not see any Lambda logging output in the Localstack
stdout which makes you think that nothing happens, although you can observe side effects (DB writes, for example),
which convince you that the function has been executed.

## Ok, how about Lambda Function graceful shutdown?

Usually, when you stumble upon something like this, you expect that the component has some means to be
cleaned up/shut down. But for AWS Lambda Functions will not work. Remember that the Lambda Functions are meant to be
initialized once and then reused? There is no need in shutting them down gracefully, so Lambda interfaces do provide us
any means to do it.

## The solution

As non-terminated threads cause problems in Localstack only (not in AWS itself), *and* Function instances are
effectively one-off in Localstack, we can just shut down the lambda function after each invocation using an env flag:

```java
public String handleRequest(Map<String, String> event, Context context) {
    try {
        return doHandleRequest(event, context);
    } finally {
        if (Boolean.parseBoolean(System.getenv("SHUTDOWN_AFTER_INVOCATION"))) {
            // stop all the threads, pools, release resources, etc
            shutdown();
        }
    }
}
```

Having this code and environment variable `SHUTDOWN_AFTER_INVOCATION` set to `true`, the Lambda Function will be
successfully called (and the control will be returned to the caller right away after it finished its execution)
in Localstack.