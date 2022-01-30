---
layout: post
title: Instantiating Uninstantiatable
date: '2022-01-30T17:43:00.000+04:00'
author: rpuchkovskiy
tags:
- java
- instantiation
- constructor
- serialization
- ReflectionFactory
- ObjectStreamClass
- Unsafe
excerpt_separator: <!--more-->
---

Let's imagine that you are writing a Java object serialization framework. You serialized your object
(turned it to an array of bytes) and now you need to deserialize it back. To do it, you need to create an empty
instance of the class first (this is called 'instantiation'). How would you do it?

The most straight-forward and obvious way is to use the no-arg constructor of the class via reflection.
Alas, there is a tiny catch: not every class has such a constructor.

But let's start from the beginning.

<!--more-->

# The interface

Here is the interface that we'll implement.

```java
interface Instantiation {
    boolean supports(Class<?> objectClass);

    Object newInstance(Class<?> objectClass) throws InstantiationException;
}
```

# No-arg constructor

Here is the promised solution using the no-arg constructor:

```java
class NoArgConstructorInstantiation implements Instantiation {
    @Override
    public boolean supports(Class<?> objectClass) {
        // this implementation is not perfect; in production code, you would probably want to cache the constructor
        for (Constructor<?> constructor : objectClass.getDeclaredConstructors()) {
            if (constructor.getParameterCount() == 0) {
                return true;
            }
        }

        return false;
    }

    @Override
    public Object newInstance(Class<?> objectClass) throws InstantiationException {
        try {
            Constructor<?> constructor = objectClass.getDeclaredConstructor();
            constructor.setAccessible(true);
            return constructor.newInstance();
        } catch (ReflectiveOperationException e) {
            throw new InstantiationException("Cannot instantiate " + objectClass, e);
        }
    }
}
```

We find the no-arg constructor, make it accessible (for the case when it's private, for example) and just invoke it.
This provides us with a fresh object that is a well-behaved Java object, it's constructor is called, everything is cool.

The only problem, as stated above, is that not every class has a no-arg constructor.
`java.util.UUID` serves as an example.

But how do we instantiate a class that does not have a no-arg constructor? We could find another constructor and
use it, but we would not know what parameters to pass to it.

Wait, [Java Serialization](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/serialTOC.html)
is doing the same thing, right? They have some magical tools to instantiate an object that
has does not have a no-arg constructor, so let's use these superpowers!

# Riding Java Serialization

Straight to the point:

```java
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.ObjectStreamClass;
import java.io.ObjectStreamConstants;
import java.io.Serializable;

class SerializableInstantiation implements Instantiation {

    private static final int STREAM_VERSION = 5;

    @Override
    public boolean supports(Class<?> objectClass) {
        if (!Serializable.class.isAssignableFrom(objectClass)) {
            return false;
        }

        // again, in production code you'd want to cache the fact that a class has this or that property
        if (hasReadObject(objectClass)) {
            return false;
        }
        if (hasReadResolve(objectClass)) {
            return false;
        }

        return true;
    }

    private boolean hasReadObject(Class<?> objectClass) {
        // a simplified approach: if there is readObject() method, we also need to check that it's private
        return hasMethod(objectClass, void.class, "readObject", ObjectInputStream.class);
    }

    private boolean hasReadResolve(Class<?> objectClass) {
        return hasMethod(objectClass, Object.class, "readResolve");
    }

    private boolean hasMethod(Class<?> objectClass, Class<?> returnType, String name, Class<?>... argTypes) {
        try {
            Method method = objectClass.getDeclaredMethod(name, argTypes);
            return method.getReturnType() == returnType;
        } catch (NoSuchMethodException e) {
            return false;
        }
    }

    @Override
    public Object newInstance(Class<?> objectClass) throws InstantiationException {
        byte[] jdkSerialization = jdkSerializationOfEmptyInstanceOf(objectClass);

        try (ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(jdkSerialization))) {
            return ois.readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new InstantiationException("Cannot deserialize JDK serialization of an empty instance", e);
        }
    }

    private byte[] jdkSerializationOfEmptyInstanceOf(Class<?> objectClass) throws InstantiationException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();

        try (DataOutputStream dos = new DataOutputStream(baos)) {
            writeSignature(dos);

            dos.writeByte(ObjectStreamConstants.TC_OBJECT);
            dos.writeByte(ObjectStreamConstants.TC_CLASSDESC);
            dos.writeUTF(objectClass.getName());

            dos.writeLong(serialVersionUid(objectClass));

            writeFlags(dos);

            writeZeroFields(dos);

            dos.writeByte(ObjectStreamConstants.TC_ENDBLOCKDATA);
            writeNullForNoParentDescriptor(dos);
        } catch (IOException e) {
            throw new InstantiationException("Cannot create JDK serialization of an empty instance", e);
        }

        return baos.toByteArray();
    }

    private void writeSignature(DataOutputStream dos) throws IOException {
        dos.writeShort(ObjectStreamConstants.STREAM_MAGIC);
        dos.writeShort(STREAM_VERSION);
    }

    private long serialVersionUid(Class<?> objectClass) {
        ObjectStreamClass descriptor = ObjectStreamClass.lookup(objectClass);
        return descriptor.getSerialVersionUID();
    }

    private void writeFlags(DataOutputStream dos) throws IOException {
        dos.writeByte(ObjectStreamConstants.SC_SERIALIZABLE);
    }

    private void writeZeroFields(DataOutputStream dos) throws IOException {
        dos.writeShort(0);
    }

    private void writeNullForNoParentDescriptor(DataOutputStream dos) throws IOException {
        dos.writeByte(ObjectStreamConstants.TC_NULL);
    }
}
```

That's quite a bulk of code, but the idea is simple: we construct a byte array that would be produced by Java
Serialization if it serialized an instance of the given class, but with a slight modification: no fields
data remains here (because we want to get an empty object).

Looks like a hack, and it is a hack, but, surprisingly, it works. Well... at least *sometimes*.

Look at the `supports()` method that became pretty bulky. As you can see:

1. Only classes marked as `Serializable` can be instantiated using this approach
2. If a class has `readObject()` method, the instantiation will most certainly fail because the Java Serialization
will call the method, which will try to read something from the stream we constructed, but we put there no data
3. If a class has `readResolve()` method, the result of instantiation may be an instance of a class different
from the one you wanted to be instantiated (as it gets replaced by the method); also, something might fail during
the `readResolve()` invocation as we probably broke some invariant while deserializing an empty object
4. One thing that is not listed here is that the deepest non-serializable superclass of the given class still must
have a no-arg constructor for this to work (this is a restriction of Java Serialization)

All this means that this approach will fail to instantiate many classes, alas.

Another catch here is that only
[constructors of non-serializable superclasses](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/serial-arch.html#5929)
will be invoked (as per the Java Serialization Specification). That might be or not be what you want.

Still, is there a better approach?

# A tale of Reflections and a Factory

But how does Java Serialization achieve its goal of instantiating something that cannot be instantiated?

The answer is easy: it generates a constructor.

Could we use the tools Java Serialization uses? Yes, we could. Just look at `jdk.internal.reflect.ReflectionFactory`.
It has `newConstructorForSerialization()`` method: you just need to find the no-arg constructor of the deepest
non-serializable superclass of your class and give it to the method, then cache the resulting constructor and
use it to instantiate your objects.

I leave this as an excercise for the reader as there seems to be a better approach.

# Java Serialization, give me your powers!

It turns out that we can use the same magic tools that Java Serialization uses without luring it to eat our tasty
but poisoned bait, and, also, avoid generating constructors ourselves.

```java
import java.io.ObjectStreamClass;
import java.io.Serializable;
import java.lang.invoke.MethodHandle;
import java.lang.invoke.MethodHandles;
import java.lang.invoke.MethodType;

class ObjectStreamClassInstantiation implements Instantiation {

    private static final MethodHandle STREAM_CLASS_NEW_INSTANCE;

    static {
        try {
            STREAM_CLASS_NEW_INSTANCE = streamClassNewInstanceMethodHandle();
        } catch (ReflectiveOperationException e) {
            throw new ExceptionInInitializerError(e);
        }
    }

    private static MethodHandle streamClassNewInstanceMethodHandle() throws NoSuchMethodException, IllegalAccessException {
        MethodHandles.Lookup lookup = MethodHandles.privateLookupIn(ObjectStreamClass.class, MethodHandles.lookup());
        return lookup.findVirtual(ObjectStreamClass.class, "newInstance", MethodType.methodType(Object.class));
    }

    @Override
    public boolean supports(Class<?> objectClass) {
        return Serializable.isAssignableFrom(objectClass);
    }

    @Override
    public Object newInstance(Class<?> objectClass) throws InstantiationException {
        // Using the standard machinery (ObjectStreamClass) to instantiate an object
        // to avoid generating excessive constructors
        // (as the standard machinery caches the constructors effectively).

        ObjectStreamClass desc = ObjectStreamClass.lookup(objectClass);

        // But as ObjectStreamClass#newInstance() is package-local, we have to resort to reflection/method handles magic.

        try {
            return STREAM_CLASS_NEW_INSTANCE.invokeExact(desc);
        } catch (Error e) {
            throw e;
        } catch (Throwable e) {
            throw new InstantiationException("Cannot instantiate", e);
        }
    }
}
```

Java Serialization uses `ObjectStreamClass` instances to represent serializable classes. So we can just construct/find
such an instance and then invoke its `newInstance()` method. The method is package local, but that's not a big deal
for guys who want to instantiate something that cannot be instantiated legally!

This mechanism does not invoke the pesky `readObject()` and `readResolve()` methods, so it has less restrictions than
the 'crafty stream' approach. Also, the constructor is generated by Java Serialization machinery, so it gets reused
if Java Serialization is instantiating the class, not just you.

But, as always with Java Serialization, there are same basic restrictions:

1. The class must be `Serializable`
2. Its deepest non-serializable superclass must have a no-arg constructor

So, are we doomed to obey the Java Serialization restrictions (or forget about instantiation if there
no-arg constructor is not present)?

Fortunately, no. We still haven't tried the most powerful tool of all.

# Behold: the Unsafe

```java
import sun.misc.Unsafe;

import java.lang.reflect.Field;
import java.security.AccessController;
import java.security.PrivilegedActionException;
import java.security.PrivilegedExceptionAction;

class UnsafeInstantiation implements Instantiation {
    private static final Unsafe UNSAFE = unsafe();

    private static Unsafe unsafe() {
        try {
            return Unsafe.getUnsafe();
        } catch (SecurityException ignored) {
            try {
                return AccessController.doPrivileged(
                        new PrivilegedExceptionAction<Unsafe>() {
                            @Override
                            public Unsafe run() throws Exception {
                                Field f = Unsafe.class.getDeclaredField("theUnsafe");

                                f.setAccessible(true);

                                return (Unsafe) f.get(null);
                            }
                        });
            } catch (PrivilegedActionException e) {
                throw new RuntimeException("Could not get Unsafe instance", e.getCause());
            }
        }
    }

    @Override
    public boolean supports(Class<?> objectClass) {
        return true;
    }

    @Override
    public Object newInstance(Class<?> objectClass) throws InstantiationException {
        try {
            return UNSAFE.allocateInstance(objectClass);
        } catch (java.lang.InstantiationException e) {
            throw new InstantiationException("Cannot instantiate " + objectClass, e);
        }
    }
}
```

The most difficult part is how you get access to the `Unsafe` instance. Having this instance, you just
allocate an instance, that's it.

On the bright side, in this way you can instantiate any class (note the `supports()` that just returns `true`),
although it would probably be wise to disallow instantiation of some classes (like `Class`).

On the other hand, this approach has some disadvantages:

1. You need to be careful with the power you get. It's unsafe, after all!
2. Java developers make it harder and harder to obtain and use `Unsafe` instances with time. At some moment
the current solution could break and will need to be fixed (or maybe it will become impossible to use at all).
3. No constructor is invoked **at all**, so any invariant it establishes can be broken now (although this happens
for Java Serialization, too, but there the author of the class has ability to intervene using special methods).

# Conclusion

Sometimes you need to instantiate a class. The easiest way is to use a no-arg constructor, but if such constructor
does not exist, you need to resort to some hack: ride Java Serialization, use its underlying machinery
(like `ReflectionFactory` or `ObjectStreamClass`), or allocate objects using `Unsafe`.

Other alternatives are possible (like using code generation directly, this is what `ReflectionFactory` does
internally), but they are left as an excercise for the reader.
