---
layout: post
title: Spring Data Elasticsearch Soft-delete Repositories
date: '2019-10-30T21:26:00.000+04:00'
author: rpuchkovskiy
tags:
- spring-data-elasticsearch
- soft-delete
- repository
- java
excerpt_separator: <!--more-->
---

Continuing on the [soft-delete repositories topic]({% post_url 2019-10-30-spring-data-elasticsearch-soft-delete-repositories %}),
let's look at Elasticsearch.

It has [Spring Data support](https://docs.spring.io/spring-data/elasticsearch/docs/current/reference/html/#reference).
And having sorted this out for Mongo, it is quite easily to do the same for Elasticsearch.

<!--more-->

Let's jump to the ...

## Solution

Again, we can hack into the standard `QueryLookupStrategy` implementation to add the desired
'soft-deleted documents are not visible by finders' behavior:

```java
public class SoftDeleteElasticsearchQueryLookupStrategy implements QueryLookupStrategy {
    private final QueryLookupStrategy strategy;
    private final ElasticsearchOperations elasticsearchOperations;
    private final SoftDeletion softDeletion = new SoftDeletion();

    public SoftDeleteElasticsearchQueryLookupStrategy(QueryLookupStrategy strategy,
            ElasticsearchOperations elasticsearchOperations) {
        this.strategy = strategy;
        this.elasticsearchOperations = elasticsearchOperations;
    }

    @Override
    public RepositoryQuery resolveQuery(Method method, RepositoryMetadata metadata, ProjectionFactory factory,
            NamedQueries namedQueries) {
        RepositoryQuery repositoryQuery = strategy.resolveQuery(method, metadata, factory, namedQueries);

        if (method.getAnnotation(SeesSoftlyDeletedRecords.class) != null) {
            return repositoryQuery;
        }

        if (!(repositoryQuery instanceof ElasticsearchPartQuery)) {
            return repositoryQuery;
        }
        ElasticsearchPartQuery partTreeQuery = (ElasticsearchPartQuery) repositoryQuery;

        return new SoftDeletePartTreeElasticsearchQuery(partTreeQuery);
    }

    private class SoftDeletePartTreeElasticsearchQuery extends ElasticsearchPartQuery {
        SoftDeletePartTreeElasticsearchQuery(ElasticsearchPartQuery partTreeQuery) {
            super((ElasticsearchQueryMethod) partTreeQuery.getQueryMethod(),
                    SoftDeleteElasticsearchQueryLookupStrategy.this.elasticsearchOperations);
        }

        @Override
        public CriteriaQuery createQuery(ParametersParameterAccessor accessor) {
            CriteriaQuery query = super.createQuery(accessor);
            return withNotDeleted(query);
        }

        private CriteriaQuery withNotDeleted(CriteriaQuery query) {
            return query.addCriteria(softDeletion.notDeleted());
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
public class SoftDeleteElasticsearchRepositoryFactory extends ElasticsearchRepositoryFactory {
    private final ElasticsearchOperations elasticsearchOperations;

    public SoftDeleteElasticsearchRepositoryFactory(ElasticsearchOperations elasticsearchOperations) {
        super(elasticsearchOperations);
        this.elasticsearchOperations = elasticsearchOperations;
    }

    @Override
    protected Optional<QueryLookupStrategy> getQueryLookupStrategy(QueryLookupStrategy.Key key,
            QueryMethodEvaluationContextProvider evaluationContextProvider) {
        Optional<QueryLookupStrategy> optStrategy = super.getQueryLookupStrategy(key,
                evaluationContextProvider);
        return optStrategy.map(this::createSoftDeleteQueryLookupStrategy);
    }

    private SoftDeleteElasticsearchQueryLookupStrategy createSoftDeleteQueryLookupStrategy(QueryLookupStrategy strategy) {
        return new SoftDeleteElasticsearchQueryLookupStrategy(strategy, elasticsearchOperations);
    }
}

public class SoftDeleteElasticsearchRepositoryFactoryBean<T extends Repository<S, ID>, S, ID extends Serializable>
        extends ElasticsearchRepositoryFactoryBean<T, S, ID> {

    public SoftDeleteElasticsearchRepositoryFactoryBean(Class<? extends T> repositoryInterface) {
        super(repositoryInterface);
    }

    @Override
    protected RepositoryFactorySupport getFactoryInstance(ElasticsearchOperations operations) {
        return new SoftDeleteElasticsearchRepositoryFactory(operations);
    }
}
```

Then we just need to reference the factory bean in a `@EnableElasticsearchRepositories` annotation like this:

```java
@EnableElasticsearchRepositories(repositoryFactoryBeanClass = SoftDeleteMongoRepositoryFactoryBean.class)
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

Absolutely the same. You just need to use reactive counterparts (`ReactiveElasticsearchOperations`, `ReactiveElasticsearchRepository` and so on).