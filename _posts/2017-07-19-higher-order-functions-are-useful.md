---
layout: post
title: "Higher Order Functions are useful"
modified: 2017-07-19
categories: [Code] 
description: "Composition with higher order functions"
tags: [Java 8, Functional Programming]
comments: true
exclude-seo: true
canonical-url: https://techdev.de/higher-order-functions-are-useful
share: true
---
In March I gave my first talk at a conference.
It was about the [combinator pattern](https://gtrefs.github.io/code/combinator-pattern/) and I wondered why there was only little feedback in regard of the pattern itself.
The only question I got was surprisingly about monads.
This kept me wondering and a little idea came to my mind: higher-order functions are useful.
This article is the result of this thought, where I am going to show how to use higher-order functions for code reuse and as a means of describing effects.

{::options parse_block_html="true" /}
<div class="message">
This article was originally published on [techdev.de](https://techdev.de/higher-order-functions-are-useful/).
</div>

## Before we start
If you are familiar with functions, higher-order functions and types, then you are well prepared for this article.
In any other case, I recommend you to check out this [presentation](https://github.com/lambdafy/presentation), [Artem](https://twitter.com/_akozlov) and I held at our local [Java User Group in Mannheim](http://www.majug.de/). 

## On higher-order functions
A higher-order function (HOF) is a function, which takes one or more other functions as parameters and/or returns another function.
Functions are values: They are constructed and passed around.
A function which is not a HOF is a first-order function.
Look at the example below.

```java
int calculate(int i, Function<Integer, Integer> f){
  return f.apply(i);
}
```

The `calculate` function takes an integer and a function and applies the function to the integer.
The interesting part is, that before and after the function application other code can be run (see listing below).

```java
int increaseAndCalculate(int i, Function<Integer, Integer> f){
  int increased = i + 1;
  return f.apply(increased);
}
```

This simple example demonstrates that HOFs can be used to separate concerns.
Increasing the integer value `i` is a different concern than applying function `f`.

## Code Reuse
The idiom of separating concerns is also found in the *execute around method pattern*.
Marius Herring has a nice [blog post](http://www.deadcoderising.com/transactions-using-execute-around-method-in-java-8/) where he uses this pattern to unify transaction handling in one place. 
Let's illustrate and discuss this idea by refactoring the code of [Alejandro Gervasio](https://www.sitepoint.com/author/agervasio/) in his article about using [Weld to inject Entity Managers](https://www.sitepoint.com/cdi-weld-inject-jpa-hibernate-entity-managers/).

[Weld](http://weld.cdi-spec.org/) is the reference implementation of the [Context and Dependency Injection (CDI)](http://cdi-spec.org/) specification and an important part of the Java EE platform.
CDI specifies how beans declare their dependencies and how the context can be used to manage a bean's life cycle.
Further, it also manages resources like data sources or entity managers.
 
Alejandro demonstrates how dependency injection with Weld can be used to inject managed resources and beans in Java SE. 
As part of the presented idea, Alejandro builds up a generic Data Access Object (DAO; see code below). 

```java
public class BaseEntityDao<T> implements EntityDao<T> {

    private EntityManager entityManager;
    private Class<T> entityClass;

    @Inject
    public BaseEntityDao(
            @MySQLDatabase EntityManager entityManager,
            @UserClass Class<T> entityClass) {
        this.entityManager = entityManager;
        this.entityClass = entityClass;
    }

    public T find(int id) {
        return entityManager.find(entityClass, id);
    }

    public List<T> findAll(String table) {
        String str = "select e from table e";
        String jpql = str.replace("table", table);
        return entityManager.createQuery(jpql).getResultList();
    }

    public void update(int id, Consumer<T>... updates) throws Exception {
        T entity = find(id);
        Arrays.stream(updates).forEach(up -> up.accept(entity));
        beginTransaction();
        commitTransaction();
    }

    public void save(T entity) {
        beginTransaction();
        entityManager.persist(entity);
        commitTransaction();
    }

    public void remove(int id) {
        T entity = find(id);
        beginTransaction();
        entityManager.remove(entity);
        commitTransaction();
    }

    private void beginTransaction() {
        try {
            entityManager.getTransaction().begin();
        } catch (IllegalStateException e) {
            rollBackTransaction();
        }
    }

    private void commitTransaction() {
        try {
            entityManager.getTransaction().commit();
        } catch (IllegalStateException | RollbackException e) {
            rollBackTransaction();
        }
    }

    private void rollBackTransaction() {
        try {
            entityManager.getTransaction().rollback();
        } catch (IllegalStateException | PersistenceException e) {
            e.printStackTrace();
        }
    }
}
```
The `BasicEntityDAO` contains the Create, Read, Update and Delete (CRUD) functionalities one would expect.
Alejandro focused on using constructor injection by exposing the dependencies through the constructor.
He uses CDIs feature for creating custom annotations like `MySQLDatabase`, called [custom qualifiers](https://www.sitepoint.com/introduction-contexts-dependency-injection-cdi/), to specify which kind of beans and resources are demanded.

In this implementation all CRUD methods deal explicitly with transaction handling: transactions are started, committed or aborted.
For example, in case of the `save` method transaction handling makes up 2/3 of the complete code.

The idea of the *execute around method pattern* is to move a scattered and/or repeated responsibility into one single method.
This method accepts an action and executes the responsibility around it. 
In case of the DAO, transaction handling is refactored in this way (see code below). 

```java
public class BaseEntityDao<T> implements EntityDao<T> {

    private EntityManager entityManager;
    private Class<T> entityClass;

    @Inject
    public BaseEntityDao(
            @WeldExampleModule.MySQLDatabase EntityManager entityManager,
            @WeldExampleModule.UserClass Class<T> entityClass) {
        this.entityManager = entityManager;
        this.entityClass = entityClass;
    }

    public T find(int id) {
        return entityManager.find(entityClass, id);
    }

    public List<T> findAll(String table) {
        String str = "select e from table e";
        String jpql = str.replace("table", table);
        return entityManager.createQuery(jpql).getResultList();
    }

    public void update(int id, Consumer<T>... updates) throws Exception {
        T entity = find(id);
        transactional(em -> Arrays.stream(updates).forEach(up -> up.accept(entity)));
    }

    public void save(T entity) {
        transactional(em -> em.persist(entity));
    }

    public void remove(int id) {
        transactional(em -> em.remove(find(id)));
    }

    private void transactional(Consumer<EntityManager> action){
        try{
            entityManager.getTransaction().begin();
            action.accept(entityManager);
            entityManager.getTransaction().commit();
        }catch (RuntimeException e){
            entityManager.getTransaction().rollback();
            throw e;
        }
    }
}
```

The `transactional` method accepts an action and executes a new transaction around it.
All other methods use `transactional` to ensure that a given logic is executed in a transaction.
This leads to more concise code and reduces the potential of incorrect transaction management.

There is only one way a pure function may return nothing: when it does nothing.
Any other observed effect is a side effect.
Because of this, `transactional` is not a HOF, but rather a higher-order procedure. 
From an outside perspective this makes it hard to reason about how it affects the program state.

Further, we might want to have more control about how actions are executed in a transaction.
For example, when the DAO evolves, more sophisticated actions, like updating and removing entities at the same time, might be necessary.
How do we compose those actions in the same transaction without relying on code duplication again?

## Means of describing effects
Let us sit back and look at the `transactional` method.
Transaction execution is the apparent side effect.
What we need is a means to describe the transaction and delay its execution.
This would allow us to compose descriptions without the burden of side effects.

A function is an interface exposing its input and output types.
Adhering to this contract, `transactional` can be refactored to a pure HOF.
Let's have a look at the code below.

```java
public void save(T entity) {
    Function<EntityManager, Void> transaction = noResult(em -> em.persist(entity));
    transaction.apply(entityManager);
}

public T convert(int id, UnaryOperator<T> converter){
   Function<EntityManager, T> find = em -> em.find(entityClass, id);
   Function<EntityManager, T> transaction = transactional(find.andThen(converter));
   return transaction.apply(entityManager);
}

private Function<EntityManager, Void> noResult(Consumer<EntityManager> action) {
    return transactional(em -> {
         action.accept(em);
         return null; 
    });
}

private Function<EntityManager, T> transactional(Function<EntityManager, T> action){
    return em -> {
         try {
              em.getTransaction().begin();
              final T result = action.accept(em);
              em.getTransaction().commit();
              return result;
        } catch (RuntimeException e) {
              em.getTransaction().rollback();
              throw e;
        }
    };
}
```

The `save` method show-cases the distinction between description and execution.
First, it uses the `noResult` function to describe that saving an entity should be done in a transaction.
Then it executes the transaction by applying it to the `entityManager`.
Thus, transaction description is pure and execution is side effecting.

Method `convert` illustrates how descriptions are composed.
It is responsible for finding an entity, applying the conversion and returning the converted entity. 
The responsibility is split into the functions `find` and `convert`.
The action which is executed within a transaction is the composition of these functions.

This approach has some draw backs.
Interface `Function` hides transaction handling details and, thus, does not convey the information whether a transaction is executed during application or not.
This may lead to unwanted side effects.
To demonstrate this, the code below contains the `transactionalConvert` method which has the same responsibility as the `convert` method.

```java
public T transactionalConvert(int id, UnaryOperator<T> converter){
   Function<EntityManager, T> find = transactional(em -> em.find(entityClass, id));
   Function<EntityManager, T> convert = transactional(em -> converter.apply(entity));
   Function<EntityManager, T> transaction = find.andThen(convert);
   return transaction.apply(entityManager);
}
```

The responsibility is split into the `find` and `convert` functions.
This time, however, both functions are transactions by themselves.
Composing and executing these functions results in two separate transactions being executed.
This might not be what the developer intended.

Another more subtle drawback is that the responsibility of composition is duplicated in each CRUD method.
Would it not be better to shift this to some [combinators](https://gtrefs.github.io/code/combinator-pattern/)?

# Summary and Conclusion
In this article we have seen that the *execute around method pattern* can be used to move the scattered responsibility of transaction management into one single method `transactional` which accepts an action and executes a transaction around it.
Due to this immediate side effect `transactional` is not a higher-order function but rather a higher-order procedure.

We then used the concept of a function as interface which allowed us to compose descriptions without the burden of side effects. 
This made composing actions easier and we were able to defer different parts of a transaction into different functions.

However, a function does not convey the information whether a transaction is executed during application or not.
This makes it hard to reason about side effects.
Another drawback was the repetition of transaction composition in every CRUD method. 

There is a solution to both problems: monads.
In my next article, I will describe what a monad is and how it helps us to combine transactions.
