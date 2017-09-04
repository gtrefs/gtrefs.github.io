---
layout: post
title: "Refactoring towards a transaction monad"
modified: 2017-09-03
categories: [Code] 
description: "Composing actions in a transaction monad"
tags: [Java 8, Functional Programming]
comments: true
exclude-seo: true
canonical-url: https://techdev.io/en/developer-blog/refactoring-towards-a-transaction-monad 
share: true
---
In my [last article](https://techdev.io/en/developer-blog/higher-order-functions-are-useful), I discussed the idea of using higher-order functions to enable code reuse and as a means of describing effects. 
With higher-order functions, transactions can first be described and then executed.
Though, the composition of transactional functions remained as duplicated code in each CRUD method.
More subtle, a function does not convey the information whether a transaction is executed during application.
This article presents a monadic transaction as a solution to both problems.

{::options parse_block_html="true" /}
<div class="message">
This article was originally published on [techdev.de](https://techdev.io/en/developer-blog/refactoring-towards-a-transaction-monad).
</div>

## Before we start
If you are familiar with functions, higher-order functions and types, then you are well prepared for this article.
In any other case, I recommend you to check out this [presentation](https://github.com/lambdafy/presentation), [Artem](https://twitter.com/_akozlov) and I held at our local [Java User Group in Mannheim](http://www.majug.de/). 

## A brief background about functional languages
Different to imperative languages the model of computation of most functional languages is the lambda calculus by Alonzo Church. 
A model of computation describes what it means to compute and what the computational elements are.

In a nutshell, the lambda calculus is a language of anonymous functions and some conversion rules.
A line of symbols is an expression.
For example, expression `λx.x` is a function where the lambda sign denotes the beginning of the head and the dot the end of the head.
The latin letters within the head are variables.
On the right side of the dot is the body.

Functions can be applied to expressions.
That is, the argument expression substitutes the variable in the function body.
Evaluating application `(λx.x)y` yields `y` as result.
This means `λx.x` is the identity function.
With some other conversion rules, this is enough to be as powerful as other models of computation, like the Turing machine or the Java memory model.

One implication is, that each problem expressed in terms of the lambda calculus has an equivalent representation in any other model of computation.
The Haskell compiler proves this by being able to translate purely-functional Haskell code into imperative C code.

## A brief and mostly incorrect introduction to monads
The lambda calculus does not rely on effects like exceptions.
In functional languages this leads to a problem.
If we can only use conversion on expressions, how do we integrate effects like input and output and still adhere to the lambda calculus?
In other words, how does a functional language with effects stay *pure*?

[Wadler](http://homepages.inf.ed.ac.uk/wadler/papers/marktoberdorf/baastad.pdf) describes in his paper that some functional languages, like Clojure, augment the lambda calculus with a number of possible effects and become impure.
He further describes how monads can be used to integrate impure effects into pure functional languages.

Monad is a concept borrowed from category theory.
Though, we don't need to dig into this theory in order to use it properly.
To state [Mario Fusco](http://qconsp.com/sp2014/system/files/presentation-slides/MonadicJava-MarioFusco.pdf): 

> "In order to understand monads you first need to learn category theory" is like saying "In order to understand Pizza you need to first learn Italian."

Wadler continues to explain that monads simplify the code structure and increase the flexibility. 

A monad of type `M` represents some computation.
For example, a monad may represent computations with I/O, with potential absent values or with values available in the future. 
Computation `M` requires two operations.
First, we need a way to turn a value into the computation that just returns this value again. 

```java
M<T> of(T value);
```

Second, we need a way to apply a function of type `Function<T, M<U>>` to a computation of type `M<T>`.

```java
M<U> flatMap(M<T> computation, Function<T, M<U>> f);
```

The type signatures of the operations are important; not the naming.
For example, operation `of` is sometimes named `pure` or `return`.

A monad is a triple `(M, of, flatMap)` consisting of a generic type `M`, and two operations of the given generic types.
Expressions are written in the following form:

```java
M<String> hello = of("hello");
Function<String, M<String>> world = str -> of(str + "World");
M<String> helloWorld = flatMap(hello, world); 
```

The result of computation `hello` is bound to variable `str` and then used in computation `world`.
A monad must adhere to [three laws](http://learnyouahaskell.com/a-fistful-of-monads#monad-laws):

* **left identity** - if we turn a value into a computation using `of` and then feed it to a function using `flatMap`, it's the same as just taking the value and applying the function to it:  `flatMap(of(x), f) == f.apply(x)` 
* **right identity** - if we have a computation and we use `flatMap` to feed it to `of`, the result is our original computation: `flatMap(x, v -> of(v)) == x`
* **associativity** - if we have a chain of function applications with `flatMap`, it shouldn't matter how they are nested: `flatMap(flatMap(x, f), g) == flatMap(x, v -> flatMap(f.apply(v), g))`

A type which adheres to the laws is also called *monadic*.
The laws describe how the operations relate to each other and, thus, allow us to make reasonable assumptions about their behavior.
However, as Rúnar Bjarnason and Paul Chiusano pointed out in [Functional Programming in Scala](https://www.manning.com/books/functional-programming-in-scala) they don't necessarily provide us a mental model of what a monad is or what a monad means. 

Bjarnason and Chiusano continue to explain that we are used to perceive interfaces as a generalization of specific representations.
For example, `ArrayList` and `LinkedList` both implement `List` which provides terms of "which a lot useful and concrete application code can be written".
Different to this, "*Monad* doesn't generalize one type or another; rather, many vastly different data types can satisfy the *Monad* interface and laws. 
The Monad operations are often just a small fragment of the full API for a given data type that happens to be a monad. [...]
The Monad contract doesn’t specify what is happening between the lines, only that whatever is happening satisfies the laws of associativity and identity."

## Monads in Java 8
With the release of Java 8 we can back the observation about monads with evidence. 
For example, type `CompletableFuture` with operations `completedFuture` and `thenCompose` satisfies the laws of identity and associativity (see code below).

```java
completedFuture("hello").thenCompose(v -> completedFuture(v + "world"))
                        .thenCompose(v -> completedFuture(v + "2017"));

completedFuture("hello").thenCompose(v -> 
  completedFuture(v + "world").thenCompose(w -> completedFuture(w + "2017"))
);
```

Let's pretend we don't know what the operations mean and what is happening.
Still we can conclude that both expressions yield the same result because of the associativity law.
What happens if we exchange operation `thenCompose` with operation `thenComposeAsync`?
As it turns out, we just found another monad consisting of type `CompletableFuture` with operations `completedFuture` and `thenComposeAsync`.

Actually, type `CompletableFuture` allows us to implement the effect of time in our program.
We can reason about `CompletableFuture` even without knowing its behavior or implementation just because it is monadic. 

## Towards a transaction monad
A monadic transaction type represents computations which are run within a transaction.
It should provide operations to put an action into a transaction and to combine transactions into a single transaction.
Further, interpreting or running a transaction with a given entity manager should return a potential result.
The code below shows the usage of such a type.

```java
public void update(int id, Consumer<T>... updates) throws Exception {
    findById(id).apply(entityClass)
                .flatMap(entity -> updateEntity(entity, updates))
                .run(entityManager);
}

private Function<Class<T>, Transaction<T>> findById(int id) {
    return clazz -> Transaction.of(em -> em.find(clazz, id));
}

private Transaction<Void> updateEntity(T entity, Consumer<T>... updates) {
    return Transcation.withoutResult(em -> {
        for (Consumer<T> up : updates) {
            up.accept(entity);
        }
    });
}
```

Method `update` uses function `findById` to search for an entity within a transaction.
Function `findById` first accepts the ID of an entity and then its class.
It uses the `of` operation to return a transaction with an action searching for the entity in the database. 

The `flatMap` operation combines two transactions into a single transaction by composing their actions.
In case of the `update` method, the transaction which finds an entity is combined with the transaction which updates an entity.
Function `updateEntity` accepts an entity and some updates and creates a new transaction which applies the updates without returning anything.

The result of the combination is a transaction which both, finds an entity and updates it.
This transaction is executed by the side-effecting `run` interpreter which takes care of starting, committing or rolling back a transaction. 

The `run` method is called interpreter, because it interprets the transaction type as an actual database transaction.
For testing purposes a different interpreter could just execute a transaction without a database transaction.
This means the potential effect of the transaction type depends on the used interpreter.

Given the capability of describing a transaction before execution, a simple set of CRUD transactions can be factored in an own interface.
All data access objects (DAO) or repositories which would like to support CRUD functionalities just inherit this interface.

```java
public interface CrudTransactions<T> {

    default Transaction<Void> saveEntity(T entity) {
        return transactional(em -> em.persist(entity));
    }

    default Transaction<Void> removeEntity(T entity) {
        return transactional(em -> em.remove(entity));
    }

    default Transaction<Void> updateEntity(T entity, Consumer<T>... updates) {
        return transactional(em -> {
            for (Consumer<T> up : updates) {
                up.accept(entity);
            }
        });
    }
    
    default Transaction<Void> transactional(Consumer<EntityManager> action) {
        return Transaction.withoutResult(action);
    }

    default Function<Class<T>, Transaction<T>> findById(int id) {
        return clazz -> Transaction.of(em -> em.find(clazz, id));
    }
}
```

For example, the `BaseEntityDao` exposes CRUD methods which rely on the inherited transactions.

```java
public class BaseEntityDao<T> implements EntityDao<T>, CrudTransactions<T> {

    private EntityManager entityManager;
    private Class<T> entityClass;

    @Inject
    public BaseEntityDao(
            @WeldExampleModule.MySQLDatabase EntityManager entityManager,
            @WeldExampleModule.UserClass Class<T> entityClass) {
        this.entityManager = entityManager;
        this.entityClass = entityClass;
    }

    public List<T> findAll(String table) {
        return Transaction.of(em -> {
            final String str = "select e from table e";
            final String jpql = str.replace("table", table);
            return (List<T>) em.createQuery(jpql).getResultList();
        }).run(entityManager);
    }

    public T find(int id) {
        return findById(id).apply(entityClass).run(entityManager);
    }

    public void update(int id, Consumer<T>... updates) throws Exception {
        findById(id).apply(entityClass)
                    .flatMap(entity -> updateEntity(entity, updates))
                    .run(entityManager);
    }

    public void save(T entity) {
        saveEntity(entity).run(entityManager);
    }

    public void remove(int id) {
        findById(id).apply(entityClass)
                    .flatMap(this::removeEntity)
                    .run(entityManager);
    }
}
``` 

This looks even more concise than just using higher-order functions.
The responsibility of combining transactions by composing their actions is moved to `flatMap`.

From a developers perspective this is also quite convenient.
Beneath `CrudTransactions` we are able to build up special purpose transactions and group them in different interfaces.
By just inheriting these interfaces we get a lot of functionality for free.

This means, we built up a reusable vocabulary to talk in domain terms with our database.
For example, `UserDetailsDao implements ReadTransactions<User>, CountTransactions<User>` may represent a DAO which backs a user details page. 

The monadic transaction type itself looks as follows.

```java
package de.gtrefs.transaction;

import javax.persistence.EntityManager;
import java.util.function.Consumer;
import java.util.function.Function;

public abstract class Transaction<T> {
    private final Function<EntityManager, T> action;

    private Transaction(Function<EntityManager, T> action) {
        this.action = action;
    }

    static Transaction<Void> withoutResult(Consumer<EntityManager> action) {
        return of(em -> {
            action.accept(em);
            return null;
        });
    }

    static <U> Transaction<U> of(U value) {
        return of(em -> value);
    }

    static <U> Transaction<U> of(Function<EntityManager, U> action) {

        return new Transaction<U>(action) {
            @Override U run(EntityManager entityManager) {
                try {
                    entityManager.getTransaction().begin();
                    final U result = action.apply(entityManager);
                    entityManager.getTransaction().commit();
                    return result;
                } catch (RuntimeException e) {
                    entityManager.getTransaction().rollback();
                    throw e;
                }
            }
        };

    }

    <U> Transaction<U> map(Function<? super T, ? extends U> mapper) {
        return of(action.andThen(mapper));
    }

    <U> Transaction<U> flatMap(Function<? super T, ? extends Transaction<U>> mapper) {
        return of(em -> {
            final Transaction<U> transaction = action.andThen(mapper).apply(em);
            return transaction.action.apply(em);
        });
    }

    abstract T run(EntityManager entityManager);
}
```

Operations `withoutResult(Consumer<EntityManager> action)`, `of(U value)` and `of(Function<EntityManager, U> action)` put an action into a transaction.
The first two are convenient for working with consumers and values.
The last one creates a new transaction with the given action.
It implements the `run` method which interprets a `Transaction` by starting an actual database transaction.

The `map` operation composes the action of the current transaction with the given mapper and returns a new transaction containing the composition.
This means, both are run in the same transaction. 

The `flatMap` operation combines transactions.
Similar to `map`, `action` and `mapper` are composed and applied to the entity manager.
Next, the result is flattened by also applying the action of the resulting transaction.
All actions are executed in the same transaction. 

Type `Transaction` with operations `of` and `flatMap` fulfills the monad laws.
The proof is left to the interested reader.

## Benefit of monads
By now, we saw two monads in the standard library and created one by ourselves.
But what is the benefit of a monad?
Why do we care about the laws and why do we try to adhere to them?

Knowing a monadic type and its operations, allows us to reason about the behavior.
We can safely assume that the outcome does not change when we exploit the associativity law and rearrange the function chaining. 

The problem is, that Java itself is a poor tool for monads.
In Scala and Haskell there is a special syntax for traversal.
For example, using the for comprehension in Scala we can rewrite the `update` method like the following.  

```scala
def findAndUpdate(id:Int, clazz:Class[User], update:Consumer[User]){
  val transaction = for {
    entity <- findById(id)(clazz)
    update <- updateEntity(entity, update)
  } yield update
  transaction.run(entityManager)
}
``` 

The Scala compiler translates each step in a for comprehension to a `flatMap` call and the final `yield` to `map`. 
This looks much more like the imperative code we are used to.
This works with any type which exposes a `flatMap` and a `map` operation. 

Another point is, that monads in Java feel artificial because they are part of a solution to a problem which never existed in Java.
Java is designed as a side-effecting imperative language describing computations as alterations of state.
In contrast, pure functional languages adhere to an abstract mathematical thinking in functions, types and laws.

Though, monadic types help us dealing with side-effects in a predictable way.
For example, as stated in the [documentation of vavr]("http://www.vavr.io/vavr-docs/#_try"), `Try` represents a computation that may either result in an exception, or return a successfully computed value.
It puts exception handling back into the normal application flow and forces us to think about exceptional cases.

## Summary
In this article we looked at the background of functional languages and how monads fit in.
By showcasing `CompletableFuture` we saw, that a monad is merely a triple consisting of a type and two operations which adhere to the laws of identity and associativity.
Different to the interfaces we are used to, the monad concept describes a *self-containing* interface which is satisfied merely when the laws are fulfilled. 

From another perspective a monad is a computation which has an operation to put values into a new computation and another operation which composes two operations.
Such a computation may be I/O or asynchronous.

We created a monadic transaction type for combining transactions and were able to define reusable transactions.
The more general concept is known as Reader monad.
It is intended to read values from an environment and feed them to functions.
With Reader we are able to do [dependency injection](https://www.youtube.com/watch?v=ZasXwtTRkio) without any framework.

In the end we discussed the benefits of monads and saw that Scala has a better support.
For more information about functional programming in general and with Scala in particular, I recommend you the book [Functional Programming in Scala](https://www.manning.com/books/functional-programming-in-scala).
