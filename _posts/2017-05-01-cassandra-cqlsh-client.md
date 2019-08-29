---
layout: post
title: Cassandra cqlsh client, OperationTimedOut and request timeouts
date: '2017-05-01T21:55:00.001+03:00'
author: rpuchkovskiy
tags:
- Cassandra
- OperationTimedOut
- cqlsh
- request timeout
modified_time: '2017-05-01T21:55:32.907+03:00'
blogger_id: tag:blogger.com,1999:blog-4160252863216482620.post-7761701281272675461
blogger_orig_url: https://rpuchkovskiy.blogspot.com/2017/05/cassandra-cqlsh-client.html
---

I'm used to mysql command line client. If a query runs for a long time, it just keeps running.
This is very useful if you feed a script (i.e. a sequence of queries) to mysql client for execution:
no matter how long each query executes, the queries are run serially. If nothing breaks in the middle,
they all execute successfully.

It turned out that with the default settings Cassandra's `cqlsh` (command line client) behaves differently.
All of a sudden, my script (a sequence of DDL queries run in the beginning of an integration test to prepare
database) has failed. The first error was **OperationTimedOut**, but the following ones were caused by the fact
that the first query did not yet finish. For example, in my case the first query was
`DROP KEYSPACE`, while the second was `CREATE KEYSPACE` with the same name. Of course, it failed, and
the following `CREATE TABLE` queries failed as well.

Why does this happen? Because `cqlsh` has a limit (by default it is 10 seconds, according to
[documentation](https://docs.datastax.com/en/cql/3.3/cql/cql_reference/cqlsh.html)). If your query runs
more than this limit, the client just fails with **OperationTimedOut** error message, but the query is still
running on the server.

OK, how do we disable this limit, or at least configure it to be long enough?

Good news: `cqlsh` in Cassandra 2.1.16 has `--request-timeout` command line parameter and you can specify
the limit there (in seconds). `--request-timeout 3600` would be a good start.

Bad news: `cqlsh` in Cassandra 2.1.12 does NOT have that parameter yet, so this parameter is not that universal.

>By the way, version reported by cqlsh (with the usual `--version`) is strange. I tried it with cqlsh included
>in Cassandra distribution for Cassandra 2.1.8, 2.1.12, 2.1.16, and in all these cases the version was reported
>as 5.0.1, even though 2.1.16 reports support for `--request-timeout` (and really supports it) and the other
>two versions don't.

But let's return to our limit.

Good news: `~/.cassandra/cqlshrc` file allows to define this timeout in <a href="http://docs.datastax.com/en/cql/3.1/cql/cql_reference/cqlshrc.html" target="_blank">[connection] section</a>.

Bad news: the documentation is not accurate. Although it says that the option was added in version 2.1.1 and
is called `request_timeout`, and this is true for 2.1.16, it is NOT true for 2.1.12. In it, you have to call
the option `client_timeout`. Moreover: in 2.1.12, according to
[this article](https://playwithcassandra.wordpress.com/2015/11/05/cqlsh-increase-timeout-limit/),
you could completely disable the timeout by assigning `None`. Alas, in 2.1.16
(with `request_timeout`) this does not work.

It is not possible to (reliably) completely disable the timeout. If you set
`request_timeout` to 0, this will mean that **any** request will timeout. Negative values cause errors.
So the only option is to set it to some large value (like the abovementioned 3600 seconds).

So, a kinda universal way to make sure your integration tests don't stumble upon this, is to put the following
in your `~/.cassandra/cqlshrc`:

```
[connection]
request_timeout = 3600
client_timeout = 3600
```

BTW, how come that `DROP KEYSPACE` for a keyspace with a few tables with no data in them where the cluster
contains just one node could not fit into the default timeout (presumably 10 seconds) on a machine with
a decent HDD which was not overloaded? It's a different story...