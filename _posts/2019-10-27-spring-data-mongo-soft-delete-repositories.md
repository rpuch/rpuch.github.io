---
layout: post
title: Spring Data Mongo Soft-delete Repositories
date: '2019-10-27T19:51:00.000+04:00'
author: rpuchkovskiy
tags:
- spring-data-mongo
- soft-delete
- repository
- java
excerpt_separator: <!--more-->
---

Everybody loves [Spring Data](https://spring.io/projects/spring-data)
[Query Methods](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#repositories.query-methods).
Let's assume that you need to find all Persons by email. You could do

```java
@Repository
public interface PersonRepository extends CrudRepository<Person, String> {
    List<Person> findPersonsByEmail(String email);
}
```

Spring Data will automatically create for you an implementation that will look for all `Person` instances in the
corresponding table/collection that have `email` property equal to the given string.

## Soft-deletion problem

Now let's imagine that your repository needs to implement 'soft-delete' logic. That means that, when an entity is 'deleted',
it is actually just labeled as such, so this is actually an update and not a real deletion. In such a storage, you need
all the retriever methods to respect that 'soft-delete' property: that is, they should not return entities marked as deleted.

How do we achieve this with a Spring Data repository?

<!--more-->

Well, we could do something like this:

```java
    List<Person> findPersonsByEmailAndDeletedIsFalse(String email);
```

But this does not seem very elegant. There are two problems:

* duplication (we must do it in each and every Query Method)
* this is error-prone (we could forget to do it somewhere)

## Solution

It turns out that for Mongo (at least, for spring-data-mongo 2.1.6) we can hack into
the standard `QueryLookupStrategy` implementation to add the desired
'soft-deleted documents are not visible by finders' behavior:

```java
    public class SoftDeleteMongoQueryLookupStrategy implements QueryLookupStrategy {
        private final QueryLookupStrategy strategy;
        private final MongoOperations mongoOperations;
    
        public SoftDeleteMongoQueryLookupStrategy(QueryLookupStrategy strategy,
                MongoOperations mongoOperations) {
            this.strategy = strategy;
            this.mongoOperations = mongoOperations;
        }
    
        @Override
        public RepositoryQuery resolveQuery(Method method, RepositoryMetadata metadata, ProjectionFactory factory,
                NamedQueries namedQueries) {
            RepositoryQuery repositoryQuery = strategy.resolveQuery(method, metadata, factory, namedQueries);
    
            // revert to the standard behavior if requested
            if (method.getAnnotation(SeesSoftlyDeletedRecords.class) != null) {
                return repositoryQuery;
            }
    
            if (!(repositoryQuery instanceof PartTreeMongoQuery)) {
                return repositoryQuery;
            }
            PartTreeMongoQuery partTreeQuery = (PartTreeMongoQuery) repositoryQuery;
    
            return new SoftDeletePartTreeMongoQuery(partTreeQuery);
        }
    
        private Criteria notDeleted() {
            return new Criteria().orOperator(
                    where("deleted").exists(false),
                    where("deleted").is(false)
            );
        }
    
        private class SoftDeletePartTreeMongoQuery extends PartTreeMongoQuery {
            SoftDeletePartTreeMongoQuery(PartTreeMongoQuery partTreeQuery) {
                super(partTreeQuery.getQueryMethod(), mongoOperations);
            }
    
            @Override
            protected Query createQuery(ConvertingParameterAccessor accessor) {
                Query query = super.createQuery(accessor);
                return withNotDeleted(query);
            }
    
            @Override
            protected Query createCountQuery(ConvertingParameterAccessor accessor) {
                Query query = super.createCountQuery(accessor);
                return withNotDeleted(query);
            }
    
            private Query withNotDeleted(Query query) {
                return query.addCriteria(notDeleted());
            }
        }
    }

    @Retention(RetentionPolicy.RUNTIME)
    @Target(ElementType.METHOD)
    public @interface SeesSoftlyDeletedRecords {
    }
```

We just add 'and not deleted' condition to all the queries unless `@SeesSoftlyDeletedRecords` on the method
asks to avoid it.

Then, we need the following infrastructure to plug our `QueryLiikupStrategy` implementation:

```java
    public class SoftDeleteMongoRepositoryFactory extends MongoRepositoryFactory {
        private final MongoOperations mongoOperations;
    
        public SoftDeleteMongoRepositoryFactory(MongoOperations mongoOperations) {
            super(mongoOperations);
            this.mongoOperations = mongoOperations;
        }
    
        @Override
        protected Optional<QueryLookupStrategy> getQueryLookupStrategy(QueryLookupStrategy.Key key,
                QueryMethodEvaluationContextProvider evaluationContextProvider) {
            Optional<QueryLookupStrategy> optStrategy = super.getQueryLookupStrategy(key,
                    evaluationContextProvider);
            return optStrategy.map(this::createSoftDeleteQueryLookupStrategy);
        }
    
        private SoftDeleteMongoQueryLookupStrategy createSoftDeleteQueryLookupStrategy(QueryLookupStrategy strategy) {
            return new SoftDeleteMongoQueryLookupStrategy(strategy, mongoOperations);
        }
    }

    public class SoftDeleteMongoRepositoryFactoryBean<T extends Repository<S, ID>, S, ID extends Serializable>
            extends MongoRepositoryFactoryBean<T, S, ID> {
    
        public SoftDeleteMongoRepositoryFactoryBean(Class<? extends T> repositoryInterface) {
            super(repositoryInterface);
        }
    
        @Override
        protected RepositoryFactorySupport getFactoryInstance(MongoOperations operations) {
            return new SoftDeleteMongoRepositoryFactory(operations);
        }
    }
```

Then we just need to reference the factory bean in a `@EnableMongoRepositories` annotation like this:

```java
    @EnableMongoRepositories(repositoryFactoryBeanClass = SoftDeleteMongoRepositoryFactoryBean.class)
```

If it is required to determine dynamically whether a particular repository needs to be 'soft-delete' or a regular
'hard-delete' repository, we can introspect the repository interface (or the domain class) inside our repository factory bean
and decide whether we need to change the `QueryLookupStrategy` or not.

## Predefined methods

### findAll() and so on

Of course, do not forget about retrievers/finders defined in `MongoRepository` (and its superinterfaces).
To do it, you need to implement a custom implementation of a repository as it is described in the
[documentation](https://docs.spring.io/spring-data/mongodb/docs/current/reference/html/#repositories.custom-implementations)
I'm not describing it here as it seems pretty straightforward.

### deleteXXX() methods

Also, deletion logic needs to be changed to make an update ('mark as deleted') instead of an actual deletion.
Same as above, this is done via custom repository implementation.

## How about reactive repositories?

Absolutely the same. You just need to use reactive counterparts (`ReactiveMongoOperations`, `ReactiveMongoRepository` and so on).