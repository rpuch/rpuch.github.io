---
layout: post
title: Extensible Enum Design Pattern
date: '2022-02-19T20:12:00.000+04:00'
author: rpuchkovskiy
tags:
- extensible
- enum
- design pattern
- constant
excerpt_separator: <!--more-->
---

It's 2022 already, but still there is a lot of Java code where integers (or strings) are used to represent
enums. Like this:

```java
public static final int COLOR_RED = 1;
public static final int COLOR_BLUE = 2;
public static final int COLOR_PURPLE = 3;
```

This has drawbacks, the most serious is that such values are weakly typed. The compiler has no idea that
`COLOR_RED` and `2*2` represent values from different domains: one represents a color, another one is just some
number (maybe, number of beds in an appartment with two bedrooms, each of which has two beds).

<!--more-->

# Enums

Of course, since Java version 1.5 we have a specific tool to solve exactly this problem: enums.

```java
public enum Color {
    RED, BLUE, PURPLE
}
```

If color IDs must be specified expliticly, it's easy:

```java
public enum Color {
    RED(1), BLUE(2), PURPLE(3);

    private final int id;

    Color(int id) {
        this.id = id;
    }

    public int id() {
        return id;
    }
}
```

So why do people still resort to weakly-typed constants?

# Extensibility

The main reason seems to be *extensibility*. Let's consider an example to make it clear what I mean.

There is a system that works with vehicles of different types. Each type has an ID.
The system core supports a limited set of vehicle types
(car, truck, motorbike). But it also has a plugin system so that plugins can be added to provide support for
other vehicle types.

So the system core could define its vehicle types like this:

```java
public enum VehicleType {
    CAR(1), TRUCK(2), MOTORBIKE(3);

    private final int id;

    VehicleType(int id) {
        this.id = id;
    }

    public int id() {
        return id;
    }
}
```

Now someone wants to create their own plugin to support another vehicle type, namely `BOAT(1001)`. But it's not
possible to extend an enum in Java; they would have to either edit the source of the core `VehicleType`
(this scales badly), or add their own enum type (but the core would have no way to interoperate with it).

One way to solve this is to return to the ancient technique of weakly-typed constants (as they can be added
easily in any namespace, just make sure there is no overlap with other plugins and the core, and you are ok!).

It is worth noting that in the context of an extensible system, the constant-based approach causes
another inconvenience: as the constants inhabiting the same virtual enum are now defined by different modules,
it becomes more difficult to gather all the possible values of such a virtual enum. The language will not help you,
you'd have to implement some convention (and adhere to it).

Is there an alternative? Yes!

# Extensible enums

Let's turn `VehicleType` into an interface

```java
public interface VehicleType {
    int id();
}
```

And make our core enum implement it:

```java
public enum CoreVehicleType implements VehicleType {
    CAR(1), TRUCK(2), MOTORBIKE(3);

    private final int id;

    CoreVehicleType(int id) {
        this.id = id;
    }

    @Override
    public int id() {
        return id;
    }
}
```

Now, the Boats plugin could define its own enum to add its own types:

```java
public enum BoatsVehicleType implements VehicleType {
    BOAT(1001), SUBMARINE(1002);

    private final int id;

    BoatsVehicleType(int id) {
        this.id = id;
    }

    @Override
    public int id() {
        return id;
    }
}
```

The system core would know nothing about the plugin types, it would just use `VehicleType` interface everywhere.
Of course, the plugin would have to return a list of its types to the core somehow, but this is out of scope
of this post.

# Summary

1. The suggested pattern allows to use strong typing
2. It allows to extend the original enumerated type
3. Inside each 'module' (core/plugin), the typing is even stronger (as it's provided by an enum)
4. It is pretty easy to find all the possible values: just search for all central interface implementations
