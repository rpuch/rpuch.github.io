---
layout: post
title: A clean, safe and concise read-only Wicket Model with Java 8 Lambdas
date: '2014-11-26T21:42:00.000+03:00'
author: rpuchkovskiy
tags:
- Java 8
- method reference
- type-safe
- concise
- Wicket model
- lambda
modified_time: '2014-11-26T21:42:23.901+03:00'
blogger_id: tag:blogger.com,1999:blog-4160252863216482620.post-8700908848124778459
blogger_orig_url: https://rpuchkovskiy.blogspot.com/2014/11/clean-safe-and-concise-read-only-wicket.html
---

<a href="http://wicket.apache.org/" target="_blank">Wicket framework</a> uses models
(<a href="https://wicket.apache.org/apidocs/1.4/org/apache/wicket/model/IModel.html" target="_blank">IModel</a>
implementations) to bind data to components. Let's say you want to display properties of some object. Here is the data class:

```java
public class User implements Serializable {
    private final String name;
    private final int age;
    
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    public String getName() {
        return name;
    }
    
    public int getAge() {
        return age;     
    }
}
```

You can do the following to display both its properties using Label components:

```java
public class AbstractReadOnlyModelPanel extends Panel {
    public AbstractReadOnlyModelPanel(String id, IModel<User> model) {
        super(id, model);
        
        add(new Label("name", new AbstractReadOnlyModel<String>() {
            @Override
            public String getObject() {
                return model.getObject().getName();
            }
        }));
        add(new Label("age", new AbstractReadOnlyModel<Integer>() {
            @Override
            public Integer getObject() {
                return model.getObject().getAge();
            }
        }));
    }
}
```
 
Straight-forward, type-safe, but not too concise: each label requires 6 lines of code! Of course, we can reduce this
count using some optimized coding conventions and so on, but anyway, anonymous classes are very verbose.

A more economical way (in terms of lines and characters to type and read) is
<a href="https://wicket.apache.org/apidocs/1.4/org/apache/wicket/model/PropertyModel.html" target="_blank">PropertyModel</a>.

```java
public class PropertyModelPanel extends Panel {
    public PropertyModelPanel(String id, IModel<User> model) {
        super(id, model);
        
        add(new Label("name", PropertyModel.of(model, "name")));
        add(new Label("age", PropertyModel.of(model, "age")));
    }
}
```

It is way shorter and still pretty intuitive. But it has drawbacks:

* First of all, it is not safe as the compiler does not check whether property named "age" exists at all!
* And it uses reflection which does not make your web-application faster. This does not seem to be critical,
but it is still a little drawback. 

Luckily, Java 8 introduced <a href="https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html" target="_blank">lambdas</a>
and <a href="https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html" target="_blank">method references</a>
which allow us to create another model implementation. Here it is:

```java
public class GetterModel<E, P> extends AbstractReadOnlyModel<P> {
    private final E entity;
    private final IModel<E> entityModel;
    private final IPropertyGetter<E, P> getter;
    
    private GetterModel(E entity, IModel<E> entityModel, IPropertyGetter<E, P> getter) {
        this.entity = entity;
        this.entityModel = entityModel;
        this.getter = getter;
    }
    
    public static <E, P> GetterModel<E, P> ofObject(E entity, IPropertyGetter<E, P> getter) {
        Objects.requireNonNull(entity, "Entity cannot be null");
        Objects.requireNonNull(getter, "Getter cannot be null");
        
        return new GetterModel<>(entity, null, getter);
    }
    
    public static <E, P> GetterModel<E, P> ofModel(IModel<E> entityModel, IPropertyGetter<E, P> getter) {
        Objects.requireNonNull(entityModel, "Entity model cannot be null");
        Objects.requireNonNull(getter, "Getter cannot be null");
        
        return new GetterModel<>(null, entityModel, getter);
    }
    
    @Override
    public P getObject() {
        return getter.getPropertyValue(getEntity());
    }
    
    private E getEntity() {
        return entityModel != null ? entityModel.getObject() : entity;
    }
}
``` 

... along with its support interface:

```java
public interface IPropertyGetter<E, P> {
    P getPropertyValue(E entity);
}
```
 
And here is the same panel example rewritten using the new model class:

```java
public class GetterModelPanel extends Panel {
    public GetterModelPanel(String id, IModel<User> model) {
        super(id, model);
        
        add(new Label("name", GetterModel.ofModel(model, User::getName)));
        add(new Label("age", GetterModel.ofModel(model, User::getAge)));
    }
}
```
 
The code is almost as concise as the version using PropertyModel, but it is:

* type-safe: the compiler will check the actual getter type
* defends you from typos better, because the compiler will check that the getter actually exists
* fast as it just uses regular method calls (2 per `getObject()` call in this case) instead of parsing property
expression and using reflection

Here are the drawbacks of the described approach in comparison with `PropertyModel`:

* It's read-only while `PropertyModel` allows to write to the property. It's easy to add ability to write using
a setter, but it will make code pretty clumsy, and we'll have to be careful and not use getter from one property
and setter from another one.
* `PropertyModel` allows to reference nested properties using the dot operator, for instance using
`outerObject.itsProperty.propertyOfProperty` property expression.

But anyway, when you just need read-only models, `GetterModel` seems to be an interesting alternative to
`PropertyModel`.

And here is a little bonus: this model implementation allows to use both models and plain data objects as sources.
We just need two factory methods: `ofModel()` and `ofObject()`, and we mimic the magical universality of
`PropertyModel` (which accepts both models and POJOs as first argument) with no magic tricks at all.
