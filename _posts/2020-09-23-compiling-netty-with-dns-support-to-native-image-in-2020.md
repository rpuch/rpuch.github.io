---
layout: post
title: Compiling Netty with DNS support to native image in 2020
date: '2020-09-23T21:08:00.000+04:00'
author: rpuchkovskiy
tags:
- netty
- netty-dns
- native-image
- java
- InetAddress
- Inet4Address
- Inet6Address
excerpt_separator: <!--more-->
---

In 2018, an excellent article was written
[Instant Netty Startup using GraalVM Native Image Generation](https://medium.com/graalvm/instant-netty-startup-using-graalvm-native-image-generation-ed6f14ff7692)
that explained
how an HTTP server using [Netty](https://netty.io/) could be compiled to a
[Native Image](https://www.graalvm.org/reference-manual/native-image/) for improved startup time and memory consumption.

It is 2020 now, the fixes for the problems highlighted by the article are fixed in Netty itself (you can see
`native-image.properties` files in many of its modules). But [GraalVM project](https://www.graalvm.org/) itself
is being actively developed, and,
all of a sudden, this can make the compilation fail again if you use `netty-dns` module with a recent GraalVM version.

Here, I'm showing what needs to be done to compile an application using Netty as a client, not a server, using its
DNS resolver feature.

<!--more-->

# The test application

The demo application is available here: https://github.com/rpuch/netty-InetAddress-native-image-diagnosing

Here's the main class:

```java
public class Main {
    public static void main(String[] args) throws Exception {
        NioEventLoopGroup group = new NioEventLoopGroup(1, new DefaultThreadFactory("netty"));

        DnsAddressResolverGroup resolverGroup = new DnsAddressResolverGroup(NioDatagramChannel.class,
                DnsServerAddressStreamProviders.platformDefault());
        AddressResolver<InetSocketAddress> resolver = resolverGroup.getResolver(group.next());
        System.out.println(resolver);

        resolver.close();
        group.shutdownGracefully().get();
    }
}
```

It just creates a event loop group and a DNS resolver group (this is what a client like Redisson would do
at initialization).

The corresponding dependendies in the `pom.xml`:

```xml
<properties>
    <netty.version>4.1.52.Final</netty.version>

    <graalvm.version>20.2.0</graalvm.version>
</properties>

<dependencies>
    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-transport</artifactId>
        <version>${netty.version}</version>
    </dependency>
    <dependency>
        <groupId>io.netty</groupId>
        <artifactId>netty-resolver-dns</artifactId>
        <version>${netty.version}</version>
    </dependency>

    <dependency>
        <groupId>org.graalvm.nativeimage</groupId>
        <artifactId>svm</artifactId>
        <version>${graalvm.version}</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```

We are using Netty 4.1.52; at the time of writing, this is the most recent released version.

## GraalVM version

I'm using version 20.2.0 here (the most recent released version as of the moment of writing).

# An attempt to build

Let's try to build the project. I'm omitting the usual `mvn clean package` and the collection of dynamic accesses
information (like reflection, JNI and proxy usage).

```
native-image --no-server --no-fallback -H:+TraceClassInitialization -H:ConfigurationFileDirectories=target/config \
--allow-incomplete-classpath \
--initialize-at-build-time=io.netty \
-cp `find target/deps | tr '\n' ':'`target/netty-InetAddress-native-image-diagnosing-1.0-SNAPSHOT.jar Main target/app
```

I request the whole `io.netty` package to be initialized at build time because I want to do as much work as possible
at build time.

Alas, `native-image` yells at me really hard:

```
Error: Unsupported features in 4 methods
Detailed message:
Error: No instances of java.net.Inet4Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
Trace: Object was reached by
	reading field io.netty.channel.socket.InternetProtocolFamily.localHost of
		constant io.netty.channel.socket.InternetProtocolFamily@8c1c1f3 reached by
	scanning method io.netty.resolver.dns.DnsNameResolver.preferredAddressType(DnsNameResolver.java:484)
Call path from entry point to io.netty.resolver.dns.DnsNameResolver.preferredAddressType(ResolvedAddressTypes):
	at io.netty.resolver.dns.DnsNameResolver.preferredAddressType(DnsNameResolver.java:481)
	at io.netty.resolver.dns.DnsNameResolver.<init>(DnsNameResolver.java:439)
	at io.netty.resolver.dns.DnsNameResolverBuilder.build(DnsNameResolverBuilder.java:476)
	at io.netty.resolver.dns.DnsAddressResolverGroup.newNameResolver(DnsAddressResolverGroup.java:114)
	at io.netty.resolver.dns.DnsAddressResolverGroup.newResolver(DnsAddressResolverGroup.java:92)
	at io.netty.resolver.dns.DnsAddressResolverGroup.newResolver(DnsAddressResolverGroup.java:77)
	at io.netty.resolver.AddressResolverGroup.getResolver(AddressResolverGroup.java:70)
	at Main.main(Main.java:19)
	at com.oracle.svm.core.JavaMainWrapper.runCore(JavaMainWrapper.java:149)
	at com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:184)
	at com.oracle.svm.core.code.IsolateEnterStub.JavaMainWrapper_run_5087f5482cc9a6abc971913ece43acb471d2631b(generated:0)
Error: No instances of java.net.Inet4Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
Trace: Object was reached by
	reading field java.net.InetSocketAddress$InetSocketAddressHolder.addr of
		constant java.net.InetSocketAddress$InetSocketAddressHolder@12c7ee50 reached by
	reading field java.net.InetSocketAddress.holder of
		constant java.net.InetSocketAddress@1f443271 reached by
	reading field io.netty.resolver.dns.SingletonDnsServerAddresses.address of
		constant io.netty.resolver.dns.SingletonDnsServerAddresses@41f58f1b reached by
	scanning method io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider.nameServerAddressStream(DefaultDnsServerAddressStreamProvider.java:115)
Call path from entry point to io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider.nameServerAddressStream(String):
	at io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider.nameServerAddressStream(DefaultDnsServerAddressStreamProvider.java:115)
	at io.netty.resolver.dns.DnsServerAddressStreamProviders$DefaultProviderHolder$1.nameServerAddressStream(DnsServerAddressStreamProviders.java:131)
	at io.netty.resolver.dns.DnsNameResolver.doResolveAllUncached0(DnsNameResolver.java:1070)
	at io.netty.resolver.dns.DnsNameResolver.access$600(DnsNameResolver.java:90)
	at io.netty.resolver.dns.DnsNameResolver$6.run(DnsNameResolver.java:1053)
	at com.oracle.svm.core.jdk.RuntimeSupport.executeHooks(RuntimeSupport.java:125)
	at com.oracle.svm.core.jdk.RuntimeSupport.executeStartupHooks(RuntimeSupport.java:75)
	at com.oracle.svm.core.JavaMainWrapper.runCore(JavaMainWrapper.java:141)
	at com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:184)
	at com.oracle.svm.core.code.IsolateEnterStub.JavaMainWrapper_run_5087f5482cc9a6abc971913ece43acb471d2631b(generated:0)
Error: com.oracle.graal.pointsto.constraints.UnsupportedFeatureException: No instances of java.net.Inet4Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
Trace:
	at parsing io.netty.resolver.dns.DnsNameResolver.resolveHostsFileEntry(DnsNameResolver.java:659)
Call path from entry point to io.netty.resolver.dns.DnsNameResolver.resolveHostsFileEntry(String):
	at io.netty.resolver.dns.DnsNameResolver.resolveHostsFileEntry(DnsNameResolver.java:651)
	at io.netty.resolver.dns.DnsNameResolver.doResolve(DnsNameResolver.java:884)
	at io.netty.resolver.dns.DnsNameResolver.doResolve(DnsNameResolver.java:733)
	at io.netty.resolver.SimpleNameResolver.resolve(SimpleNameResolver.java:61)
	at io.netty.resolver.SimpleNameResolver.resolve(SimpleNameResolver.java:53)
	at io.netty.resolver.InetSocketAddressResolver.doResolve(InetSocketAddressResolver.java:55)
	at io.netty.resolver.InetSocketAddressResolver.doResolve(InetSocketAddressResolver.java:31)
	at io.netty.resolver.AbstractAddressResolver.resolve(AbstractAddressResolver.java:106)
	at io.netty.bootstrap.Bootstrap.doResolveAndConnect0(Bootstrap.java:206)
	at io.netty.bootstrap.Bootstrap.doResolveAndConnect(Bootstrap.java:162)
	at io.netty.bootstrap.Bootstrap.connect(Bootstrap.java:139)
	at io.netty.resolver.dns.DnsNameResolver$DnsResponseHandler.channelRead(DnsNameResolver.java:1239)
	at io.netty.channel.AbstractChannelHandlerContext.invokeChannelRead(AbstractChannelHandlerContext.java:379)
	at io.netty.channel.AbstractChannelHandlerContext.access$600(AbstractChannelHandlerContext.java:61)
	at io.netty.channel.AbstractChannelHandlerContext$7.run(AbstractChannelHandlerContext.java:370)
	at com.oracle.svm.core.jdk.RuntimeSupport.executeHooks(RuntimeSupport.java:125)
	at com.oracle.svm.core.jdk.RuntimeSupport.executeStartupHooks(RuntimeSupport.java:75)
	at com.oracle.svm.core.JavaMainWrapper.runCore(JavaMainWrapper.java:141)
	at com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:184)
	at com.oracle.svm.core.code.IsolateEnterStub.JavaMainWrapper_run_5087f5482cc9a6abc971913ece43acb471d2631b(generated:0)
Error: com.oracle.graal.pointsto.constraints.UnsupportedFeatureException: No instances of java.net.Inet6Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
Trace:
	at parsing io.netty.resolver.dns.DnsQueryContextManager.getOrCreateContextMap(DnsQueryContextManager.java:111)
Call path from entry point to io.netty.resolver.dns.DnsQueryContextManager.getOrCreateContextMap(InetSocketAddress):
	at io.netty.resolver.dns.DnsQueryContextManager.getOrCreateContextMap(DnsQueryContextManager.java:96)
	at io.netty.resolver.dns.DnsQueryContextManager.add(DnsQueryContextManager.java:42)
	at io.netty.resolver.dns.DnsQueryContext.<init>(DnsQueryContext.java:69)
	at io.netty.resolver.dns.TcpDnsQueryContext.<init>(TcpDnsQueryContext.java:35)
	at io.netty.resolver.dns.DnsNameResolver$DnsResponseHandler$1.operationComplete(DnsNameResolver.java:1257)
	at io.netty.resolver.dns.DnsNameResolver$DnsResponseHandler$1.operationComplete(DnsNameResolver.java:1239)
	at io.netty.util.concurrent.DefaultPromise.notifyListener0(DefaultPromise.java:577)
	at io.netty.util.concurrent.DefaultPromise.access$300(DefaultPromise.java:35)
	at io.netty.util.concurrent.DefaultPromise$2.run(DefaultPromise.java:531)
	at com.oracle.svm.core.jdk.RuntimeSupport.executeHooks(RuntimeSupport.java:125)
	at com.oracle.svm.core.jdk.RuntimeSupport.executeStartupHooks(RuntimeSupport.java:75)
	at com.oracle.svm.core.JavaMainWrapper.runCore(JavaMainWrapper.java:141)
	at com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:184)
	at com.oracle.svm.core.code.IsolateEnterStub.JavaMainWrapper_run_5087f5482cc9a6abc971913ece43acb471d2631b(generated:0)

Error: Use -H:+ReportExceptionStackTraces to print stacktrace of underlying exception
Error: Image build request failed with exit status 1
```

This is a lot of output, but it can easily be seen that the theme is the same: *No instances of XYZ are allowed in the
image heap as this class should be initialized at image runtime* where XYZ is either `java.net.Inet4Address` or
`java.net.Inet6Address`. Let's see what it means.

# A bit of theory

But first a bit of theory might be useful.

In Java, every class must be initialized before usage. Initialization means executing static field initializers
and static initialization blocks. In a standard JVM (like HotSpot), this happens at runtime, of course.

But with a native image, you have two alternatives (well, actually, there is a third one, but I'll omit it for the sake
of simplicity). A class may still be initialized at runtime, or its initialization
may be carried out at build time. The latter has an obvious benefit of avoiding this work at native image startup,
which makes the image start faster. But for some classes it does not make sense to initialize them at build time.
Such an example could be a class that makes some decisions in its initialization (create an instance of this or that
class, for example) based on environment (env variables, config files, etc).

There are some restrictions in regard to choosing between build/run time initialization alternatives:

* If a class is initialized at build time, all of its superclasses must be initialized at build time
* If a class is initialized at run time, all of its subclasses must be initialized at runtime
* Some classes must always be initialized at runtime (see below)
* An instance of a class that is initialized at runtime cannot exist in *image heap* (this means that **no
build-time initialized class or its instance can (directly or indirectly) reference such a runtime-initialized
class instance**)

## InetAddress issue

Since version 19.3.0, `native-image` tool requires that `java.net.InetAddress` class is always initialized at runtime.
This (by restriction 2) means that its subclasses, `java.net.Inet4Address` and `java.net.Inet6Address` must also be
initialized at runtime, which in turn (by restriction 4) means that you cannot have any `InetAddress` referenced
by a build-time initialized class.

All the build failures we encounter here are caused by this same problem: either `Inet4Address` or `Inet6Address`
in image heap.

# But why could Netty-related classes try to be initialized at build time in real projects?

For example, `netty-codec-http` contains the following

    Args = --initialize-at-build-time=io.netty \

in its `native-image.properties` below `META-INF`, and micronaut has `netty-codec-http` as a dependency, so all
`io.netty` classes are by default initialized at build time (as `native-image` tool respects such
`native-image.properties` files) if your project uses Micronaut (at least, version 2.0.0).

# Diagnosing and fixing

## 1

Build fails with the following error:

    Error: com.oracle.graal.pointsto.constraints.UnsupportedFeatureException: No instances of java.net.Inet4Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
    Trace:
        at parsing io.netty.resolver.dns.DnsNameResolver.resolveHostsFileEntry(DnsNameResolver.java:659)
    Call path from entry point to io.netty.resolver.dns.DnsNameResolver.resolveHostsFileEntry(String):
        at io.netty.resolver.dns.DnsNameResolver.resolveHostsFileEntry(DnsNameResolver.java:651)
        at io.netty.resolver.dns.DnsNameResolver.doResolve(DnsNameResolver.java:884)
        at io.netty.resolver.dns.DnsNameResolver.doResolve(DnsNameResolver.java:733)
        at io.netty.resolver.SimpleNameResolver.resolve(SimpleNameResolver.java:61)
        at io.netty.resolver.SimpleNameResolver.resolve(SimpleNameResolver.java:53)
        at io.netty.resolver.InetSocketAddressResolver.doResolve(InetSocketAddressResolver.java:55)
        at io.netty.resolver.InetSocketAddressResolver.doResolve(InetSocketAddressResolver.java:31)
        at io.netty.resolver.AbstractAddressResolver.resolve(AbstractAddressResolver.java:106)
        at io.netty.bootstrap.Bootstrap.doResolveAndConnect0(Bootstrap.java:206)
        at io.netty.bootstrap.Bootstrap.access$000(Bootstrap.java:46)
        at io.netty.bootstrap.Bootstrap$1.operationComplete(Bootstrap.java:180)

`DnsNameResolver:659` is

    return LOCALHOST_ADDRESS;

and it references the static field named `LOCALHOST_ADDRESS` of type `InetAddress`. Let's avoid its initialization
at build time by adding the following to the `native-image` command`:

    --initialize-at-run-time=io.netty.resolver.dns.DnsNameResolver

The error goes away.

## 2

Now there is another one:

    Error: No instances of java.net.Inet6Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
    Trace: Object was reached by
        reading field java.util.HashMap$Node.value of
            constant java.util.HashMap$Node@26eb0f30 reached by
        indexing into array
            constant java.util.HashMap$Node[]@63e95621 reached by
        reading field java.util.HashMap.table of
            constant java.util.HashMap@563992d1 reached by
        reading field java.util.Collections$UnmodifiableMap.m of
            constant java.util.Collections$UnmodifiableMap@38a9945c reached by
        reading field io.netty.resolver.DefaultHostsFileEntriesResolver.inet6Entries of
            constant io.netty.resolver.DefaultHostsFileEntriesResolver@7ef4ba7e reached by
        scanning method io.netty.resolver.dns.DnsNameResolverBuilder.<init>(DnsNameResolverBuilder.java:56)
    Call path from entry point to io.netty.resolver.dns.DnsNameResolverBuilder.<init>():
        at io.netty.resolver.dns.DnsNameResolverBuilder.<init>(DnsNameResolverBuilder.java:68)
        at io.netty.resolver.dns.DnsAddressResolverGroup.<init>(DnsAddressResolverGroup.java:54)
        at Main.main(Main.java:18)

`DnsNameResolverBuilder:56` is

    private HostsFileEntriesResolver hostsFileEntriesResolver = HostsFileEntriesResolver.DEFAULT;

Let's postpone `HostsFileEntriesResolver` initialization:

    --initialize-at-run-time=io.netty.resolver.HostsFileEntriesResolver

## 3

Now, there is another error:

    Error: com.oracle.graal.pointsto.constraints.UnsupportedFeatureException: No instances of java.net.Inet6Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
    Trace:
        at parsing io.netty.resolver.dns.DnsQueryContextManager.getOrCreateContextMap(DnsQueryContextManager.java:111)
    Call path from entry point to io.netty.resolver.dns.DnsQueryContextManager.getOrCreateContextMap(InetSocketAddress):
        at io.netty.resolver.dns.DnsQueryContextManager.getOrCreateContextMap(DnsQueryContextManager.java:96)

`DnsQueryContextManager:111` references `NetUtil.LOCALHOST6`. `NetUtil` is initialized at build time, but its fields
`LOCALHOST4` and `LOCALHOST6` contain instances of `Inet4Address` and `Inet6Address`, respectively. This can be solved
by a *substitution*. We just add the following class to our project:

    @TargetClass(NetUtil.class)
    final class NetUtilSubstitutions {
        @Alias
        @InjectAccessors(NetUtilLocalhost4Accessor.class)
        public static Inet4Address LOCALHOST4;

        @Alias
        @InjectAccessors(NetUtilLocalhost6Accessor.class)
        public static Inet6Address LOCALHOST6;

        private static class NetUtilLocalhost4Accessor {
            static Inet4Address get() {
                // using https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom
                return NetUtilLocalhost4LazyHolder.LOCALHOST4;
            }
        }

        private static class NetUtilLocalhost4LazyHolder {
            private static final Inet4Address LOCALHOST4;

            static {
                byte[] LOCALHOST4_BYTES = {127, 0, 0, 1};
                // Create IPv4 loopback address.
                try {
                    LOCALHOST4 = (Inet4Address) InetAddress.getByAddress("localhost", LOCALHOST4_BYTES);
                } catch (Exception e) {
                    // We should not get here as long as the length of the address is correct.
                    PlatformDependent.throwException(e);
                    throw new IllegalStateException("Should not reach here");
                }
            }
        }

        private static class NetUtilLocalhost6Accessor {
            static Inet6Address get() {
                // using https://en.wikipedia.org/wiki/Initialization-on-demand_holder_idiom
                return NetUtilLocalhost6LazyHolder.LOCALHOST6;
            }
        }

        private static class NetUtilLocalhost6LazyHolder {
            private static final Inet6Address LOCALHOST6;

            static {
                byte[] LOCALHOST6_BYTES = {0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1};
                // Create IPv6 loopback address.
                try {
                    LOCALHOST6 = (Inet6Address) InetAddress.getByAddress("localhost", LOCALHOST6_BYTES);
                } catch (Exception e) {
                    // We should not get here as long as the length of the address is correct.
                    PlatformDependent.throwException(e);
                    throw new IllegalStateException("Should not reach here");
                }
            }
        }
    }

The idea is to replace loads of the problematic fields with method invocations controlled by us. This is achieved
via a substitution (note `@TargetClass`, `@Alias` and `@InjectAccessors`). As a result, the `InetAddress` values
are not stored in image heap anymore. The error goes away.

## 4

We now have another one:

    Error: No instances of java.net.Inet6Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
    Trace: Object was reached by
        reading field io.netty.channel.socket.InternetProtocolFamily.localHost of
            constant io.netty.channel.socket.InternetProtocolFamily@5dc39065 reached by
        scanning method io.netty.resolver.dns.DnsNameResolver.preferredAddressType(DnsNameResolver.java:487)
    Call path from entry point to io.netty.resolver.dns.DnsNameResolver.preferredAddressType(ResolvedAddressTypes):
        at io.netty.resolver.dns.DnsNameResolver.preferredAddressType(DnsNameResolver.java:481)

As it can be seen from the code of `InternetProtocolFamily`, each enum constant stores an instance of `InetAddress`,
so if any class initialized at build time initializes `InternetProtocolFamily`, the image heap gets polluted with
`InetAddress` instance. This can also be solved with a substitution:

    @TargetClass(InternetProtocolFamily.class)
    final class InternetProtocolFamilySubstitutions {
        @Alias
        @InjectAccessors(InternetProtocolFamilyLocalhostAccessor.class)
        private InetAddress localHost;

        private static class InternetProtocolFamilyLocalhostAccessor {
            static InetAddress get(InternetProtocolFamily family) {
                switch (family) {
                    case IPv4:
                        return NetUtil.LOCALHOST4;
                    case IPv6:
                        return NetUtil.LOCALHOST6;
                    default:
                        throw new IllegalStateException("Unsupported internet protocol family: " + family);
                }
            }

            static void set(InternetProtocolFamily family, InetAddress address) {
                // storing nothing as the getter derives all it needs from its argument
            }
        }
    }

The error goes away.

## 5

There is another one this time:

    Error: No instances of java.net.Inet4Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
    Detailed message:
    Trace: Object was reached by
        reading field java.net.InetSocketAddress$InetSocketAddressHolder.addr of
            constant java.net.InetSocketAddress$InetSocketAddressHolder@34913c36 reached by
        reading field java.net.InetSocketAddress.holder of
            constant java.net.InetSocketAddress@ad1fe10 reached by
        reading field io.netty.resolver.dns.SingletonDnsServerAddresses.address of
            constant io.netty.resolver.dns.SingletonDnsServerAddresses@79fd599 reached by
        scanning method io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider.nameServerAddressStream(DefaultDnsServerAddressStreamProvider.java:115)
    Call path from entry point to io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider.nameServerAddressStream(String):
        at io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider.nameServerAddressStream(DefaultDnsServerAddressStreamProvider.java:115)
        at io.netty.resolver.dns.DnsServerAddressStreamProviders$DefaultProviderHolder$1.nameServerAddressStream(DnsServerAddressStreamProviders.java:131)
        at io.netty.resolver.dns.DnsNameResolver.doResolveAllUncached0(DnsNameResolver.java:1070)

First, let's move initialization of `io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider` to
run-time:

    --initialize-at-run-time=io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider

Now, the error is similar, but still slightly different:

    Error: No instances of java.net.Inet4Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
    Trace: Object was reached by
        reading field java.net.InetSocketAddress$InetSocketAddressHolder.addr of
            constant java.net.InetSocketAddress$InetSocketAddressHolder@5537c5de reached by
        reading field java.net.InetSocketAddress.holder of
            constant java.net.InetSocketAddress@fb954f8 reached by
        reading field io.netty.resolver.dns.SingletonDnsServerAddresses.address of
            constant io.netty.resolver.dns.SingletonDnsServerAddresses@3ec9baab reached by
        reading field io.netty.resolver.dns.UnixResolverDnsServerAddressStreamProvider.defaultNameServerAddresses of
            constant io.netty.resolver.dns.UnixResolverDnsServerAddressStreamProvider@1b7f0339 reached by
        reading field io.netty.resolver.dns.DnsServerAddressStreamProviders$DefaultProviderHolder$1.currentProvider of
            constant io.netty.resolver.dns.DnsServerAddressStreamProviders$DefaultProviderHolder$1@2d249be7 reached by
        scanning method io.netty.resolver.dns.DnsServerAddressStreamProviders.unixDefault(DnsServerAddressStreamProviders.java:104)
    Call path from entry point to io.netty.resolver.dns.DnsServerAddressStreamProviders.unixDefault():
        at io.netty.resolver.dns.DnsServerAddressStreamProviders.unixDefault(DnsServerAddressStreamProviders.java:104)
        at io.netty.resolver.dns.DnsServerAddressStreamProviders.platformDefault(DnsServerAddressStreamProviders.java:100)
        at Main.main(Main.java:18)

Ok, let's move initialization of `io.netty.resolver.dns.DnsServerAddressStreamProviders$DefaultProviderHolder` to
run-time as well:

    '--initialize-at-run-time=io.netty.resolver.dns.DnsServerAddressStreamProviders$DefaultProviderHolder'

(note the single quotes: without them `$` and characters following it will be interpreted by `sh` and replaced with an
empty string).

The error goes away.

>Please note that the order turned out to be important here. When I first moved
`io.netty.resolver.dns.DnsServerAddressStreamProviders$DefaultProviderHolder` initialization to run-time but did not
touch `io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider` initialization, the error report did not change
a bit. So it takes a little patience and experimenting (or some knowledge which I do not have, alas).

Now we have this:

    Error: com.oracle.graal.pointsto.constraints.UnsupportedFeatureException: No instances of java.net.Inet6Address are allowed in the image heap as this class should be initialized at image runtime. Object has been initialized without the native-image initialization instrumentation and the stack trace can't be tracked.
    Detailed message:
    Trace:
        at parsing io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider.<clinit>(DefaultDnsServerAddressStreamProvider.java:87)
    Call path from entry point to io.netty.resolver.dns.DefaultDnsServerAddressStreamProvider.<clinit>():
        no path found from entry point to target method

Ok, it's `NetUtil.LOCALHOST` being referenced, so let's add a substitution for it as well (to `NetUtilSubstitutions`):

    @Alias
    @InjectAccessors(NetUtilLocalhostAccessor.class)
    public static InetAddress LOCALHOST;

    // NOTE: this is the simpliest implementation I could invent to just demonstrate the idea; it is probably not
    // too efficient. An efficient implementation would only have getter and it would compute the InetAddress
    // there; but the post is already very long, and NetUtil.LOCALHOST computation logic in Netty is rather cumbersome.
    private static class NetUtilLocalhostAccessor {
        private static volatile InetAddress ADDR;

        static InetAddress get() {
            return ADDR;
        }

        static void set(InetAddress addr) {
            ADDR = addr;
        }
    }

This makes the final error go away.

>Thanks to [Nicolas Filotto](https://stackoverflow.com/users/1997376/nicolas-filotto) for his suggestions on item 5.

## Now it's fixed!

The application builds successfully and can be run using `target/app`.

# Summary of techniques

1. First of all, you could find a class that, being moved to runtime initialization phase, makes a failure to go away.
To do this, you could trace references in the provided stack trace. Best candidates to consider are static fields.
2. If item 1 does not help, you could try to create a substitution
3. There is also a theoretical possibility to go with a lighter variant: `@RecomputeFieldValue`, but it's more
restricted, and I could not make it work in this Netty-related task.

# Links

1. This post is a slight adaptation of a [StackOverflow answer](https://stackoverflow.com/questions/63328298/how-do-you-debug-a-no-instances-of-are-allowed-in-the-image-heap-when-buil)
2. [Instant Netty Startup using GraalVM Native Image Generation](https://medium.com/graalvm/instant-netty-startup-using-graalvm-native-image-generation-ed6f14ff7692)
3. [Native Image](https://www.graalvm.org/reference-manual/native-image/)
4. Substitution-related code is inspired by https://github.com/quarkusio/quarkus/pull/5353/files
