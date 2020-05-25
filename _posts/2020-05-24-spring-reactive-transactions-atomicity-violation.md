---
layout: post
title: Spring reactive transactions atomicity violation
date: '2020-05-25T21:42:00.000+04:00'
author: rpuchkovskiy
tags:
- Spring
- reactive
- transaction
- atomicity
- violation
excerpt_separator: <!--more-->
---

Spring supports transactions in reactive flows since version 5.2 M2, so this combination
(transactivity + reactive streams) is still pretty new. Transactions in imperative mode are here for years, they are
well known and no serious caveats are expected. But it turns out that, in the current Spring versions (namely,
version 5.2.6), reactive transactions may sometimes violate the atomicity requirement!

<!--more-->

TLDR; skip to <a href="#pragmatic-recommendations">Pragmatic recommendations</a>.

Further I explain why the atomicity violation
can happen, but first a few words about the differences between the classic (imperative) case and the reactive one.

## Imperative transaction management flow

In the classic (imperative) world, here is how wrapping of a business code in a transaction looks like:

```java
startTransaction();
try {
    businessLogic();
    commitTransaction();
} catch (Throwable e) {
    handleException(e);
}
```

It can be seen that a classic transactional code has only two terminal states:

1. it either 'completes normally', in which case the transaction gets committed;
2. or an exception is thrown, in which case the outcome actually depends on an exception, as some of them cause
transaction commit, but most of the time a rollback happens.

## Reactive transaction management flow

In the reactive case, schematically, it looks like this:

```java
Publisher<T> businessLogicPublisher = businessLogic();
Publisher<T> transactionalPublisher = new Transactionally(businessPublisher, transactionManager);
transactionalPublisher.subscribe(...);
```

When the flow is activated (i.e. a subscription is made on the final publisher), a `Subscription` object is created.
The subscription has three terminal states:

1. *complete* signalled via `Subscriber#onComplete()`. This is the analogue of 'completes normally' state from the
imperative flow; in this case the transaction gets committed.
2. *error* signalled via `Subscriber#onError(Throwable)`. This is the analogue of 'an exception is thrown' state
from the imperative flow; in this case the same handling logic is executed, so the transaction may be committed or
rolled back, it all depends on the `Throwable` that is available when handling this signal.
3. *cancelled* signalled via `Subscription#onCancel()`. This is issued when the subscription gets cancelled, and here
the problem has its roots.

## Why is cancellation a problem?

The thing is that a cancellation can happen due to different reasons.

1. It can be a part of the 'normal flow'. For example, some [Reactor](https://projectreactor.io/)'s operators, like
`Flux.take(n)`, `Flux.next()`, `Mono.from(Publisher)` cancel the upstream when they are satisfied with the number of
elements they got from there and do not need it anymore. Such cancellations are 'benign' and should be treated in the
same way as a successful completion.
2. It can be due to an exception thrown somewhere inside the reactive pipe. For example, `Flux.concatMap()` cancels
the upstream when it gets an 'onError()` signal.
3. A cancellation may be requested by some code external to the reactive pipeline (for example, it may be a consequence
of an application shutdown).

In the first case, the current transaction must be committed on cancel. In the second and third, it must be rolled back.

The worse thing is that a cancellation signal does not allow any information to be carried along. According to
[Reactor developers](https://github.com/spring-projects/spring-framework/issues/25091#issuecomment-630095597),
[Reactor context](https://projectreactor.io/docs/core/release/reference/#context) does not allow to pass
such information as well. So it does not seem possible to distinguish these cases and pick the right decision:
whether the current transaction needs to be committed or rolled back on cancel.

## How it works now in Spring

Spring 5.2.6 commits on cancel. The choice was made
[deliberately](https://github.com/spring-projects/spring-framework/issues/25091#issuecomment-632791093) to make
operators like `Flux.take(n)` usable with transactions.

**This creates a possibility of a partial (i.e. non-atomic) commit** if a cancel arrives at bad time!

### Demonstrating the problem

Having the following 'business method' (very artificial, crafted specifically to get 100% reproduction rate)

```java
    @Transactional
    public Mono<Void> savePair(String collection, CountDownLatch latch) {
        return Mono.defer(() -> {
            Boot left = new Boot();
            left.setKind("left");
            Boot right = new Boot();
            right.setKind("right");

            return mongoOperations.insert(left, collection)
                    // signaling to the test that the first insert has been done and the subscription can be cancelled
                    .then(Mono.fromRunnable(latch::countDown))
                    // do not proceed to the second insert ever
                    .then(Mono.fromRunnable(this::blockForever))
                    .then(mongoOperations.insert(right, collection))
                    .then();
        });
    }
```

the following test

```java
    @Test
    void cancelShouldNotLeadToPartialCommit() throws InterruptedException {
        // latch is used to make sure that we cancel the subscription only after the first insert has been done
        CountDownLatch latch = new CountDownLatch(1);

        Disposable disposable = bootService.savePair(collection, latch).subscribe();

        // wait for the first insert to be executed
        latch.await();

        // now cancel the reactive pipeline
        disposable.dispose();

        // Now see what we have in the DB. Atomicity requires that we either see 0 or 2 documents.
        List<Boot> boots = mongoOperations.findAll(Boot.class, collection).collectList().block();

        assertEquals(0, boots.size());
    }
```

fails because it sees exactly 1 record in the Mongo collection.

## So what's the big deal?

Well, if a transaction may be unreliable under 'some rare circumstances', it means that the transactions are
not reliable. We usually use transactions when we need a 100% guarantee that our commits will be atomic so that we can
count on data staying consistent. Spring reactive transactions currently cannot provide such a guarantee.

## What can be done?

The Spring team is [working on it](https://github.com/spring-projects/spring-framework/issues/25091), but right now
the safest approach is to patch `spring-tx` by flipping the logic from 'commit-on-cancel' to 'rollback-on-cancel'.

Here is a (straight-forward) example of how it can be done:
[https://github.com/rpuch/spring-framework/commit/95c2872c0c3a8bebec06b413001148b28bc78f2a](https://github.com/rpuch/spring-framework/commit/95c2872c0c3a8bebec06b413001148b28bc78f2a)  
This fixes `TransactionalOperator` and the declarative `@Transactional`-based cases.

If you use spring-data module's specific transactivity mechanisms, you need to address them as well. For example,
if `ReactiveMongoOperations.inTransaction()` is in use, you need to change the following code in
`ReactiveMongoTemplate.inTransaction(Publisher)`

```java
    return Flux.usingWhen(Mono.just(session), //
            s -> ReactiveMongoTemplate.this.withSession(action, s), //
            ClientSession::commitTransaction, //
            (sess, err) -> sess.abortTransaction(), //
            ClientSession::commitTransaction) //
            .doFinally(signalType -> doFinally.accept(session));
```

to use `ClientSession::abortTransaction` instead of `ClientSession::commitTransaction` in the
`asyncCancel` parameter.

### The unpleasant consequences

Unfortunately, the fix is not free. It will restore the transactional guarantees for the cases of unexpected
cancellations. But, in return, it will make expected cancellations to *also* roll transactions back. Here is an example:

```java
    @Transactional
    public Flux<Shoe> savePairAndReturnFlux(String collection) {
        return Flux.defer(() -> {
            Shoe left = new Shoe("left");
            Shoe right = new Shoe("right");

            return mongoOperations.insert(List.of(left, right), collection)
                    .thenMany(Flux.just(left, right));
        });
    }
```

The following code

```java
    shoeService.savePairAndReturnFlux(collection)
            .take(1)
            .blockLast();
```

will roll the transaction back, so nothing will be stored in the database.

## Pragmatic recommendations

1. If you absolutely need operations like `Flux.take(n)` downstream from your transactional publishers, **and**
you never
make more than one write in your transactional code, it is ok to proceed with the current unpatched Spring versions as
the partial commits are never going to bite you. *But please note that (at least, currently) Spring team is going to
flip the logic to 'rollback-on-cancel' in Spring 5.3*, so it is better to be prepared anyway.
2. If you absolutely need operations like `Flux.take(n)` downstream from your transactional publishers, **and**
you sometimes have more than one write in your transactional code, you are in trouble. You are forced to use the
currently standard 'commit-on-cancel', but you are amenable to atomicity violations (partial commits) on cancel.
3. If you do not need 'routinely cancelling' operations like `Flux.take(n)` downstream from transactional publishers,
switch to a patched Spring version (with 'rollback-on-cancel' policy) and then switch to Spring 5.3 when it is available
(that will have the same 'rollback-on-cancel' policy).

## Conclusion

Reactive transactions in the current Spring version (5.2.6) leave a possibility for non-atomic commits, but
in many cases it is possible to fix the problem by using a patched `spring-tx` jar.

## References

1. [Spring Reactive Transactions introduction](https://spring.io/blog/2019/05/16/reactive-transactions-with-spring)
2. [https://github.com/spring-projects/spring-framework/issues/25091](https://github.com/spring-projects/spring-framework/issues/25091)
The bug report at Spring Framework's Github page
3. A [Stackoverflow question](https://stackoverflow.com/questions/61822249/spring-reactive-transaction-gets-committed-on-cancel-producing-partial-commits?noredirect=1#comment109381171_61822249)
about the problem
4. [https://github.com/rpuch/spring-commit-on-cancel-problems](https://github.com/rpuch/spring-commit-on-cancel-problems)
A github repository with a test that demonstrates the problem
5. [https://github.com/rpuch/spring-framework/commit/95c2872c0c3a8bebec06b413001148b28bc78f2a](https://github.com/rpuch/spring-framework/commit/95c2872c0c3a8bebec06b413001148b28bc78f2a)
A commit switching to 'rollback-on-cancel'
