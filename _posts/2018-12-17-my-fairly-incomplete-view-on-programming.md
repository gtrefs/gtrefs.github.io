---
layout: post
title: "My fairly incomplete view on programming"
modified: 2018-12-17
categories: [funtional programming, programming] 
description: "What is meant by computation and model of computation? How does it refer to functional programming?"
tags: [Functional Programming, Referential transparency]
comments: true
share: true
---
Some time ago I had an argument about the notion of behavior and the definitions of referential transparency and purity in functional programming. Due to my incomplete knowledge I could not back my arguments and, therefore, decided to do some research. This post is a write up of my research and primarily serves me as a source. Nonetheless, I welcome you to comment and correct my result.

# Models of computation
Theoretical computer science emerged primarily from the work of Alonzo Church and Alan Turing. One of their contributions was the logical foundation of computation. Such a model of computation describes the process of computation and, thus, provides the meaning of different computational steps.

Different to imperative programming languages the model of computation of functional programming is the lambda calculus by Alonzo Church. [Raúl Rojas](http://www.inf.fu-berlin.de/lehre/WS03/alpi/lambda.pdf) and [Joscha Bach](http://palmstroem.blogspot.de/2012/05/lambda-calculus-for-absolute-dummies.html) provide some very nice explanations. 

The lambda calculus is a language of anonymous functions and conversion rules. A line of symbols is an expression. For example, expression `λx.x` is a function where the lambda sign denotes the beginning of the head and the dot the end of the head. The head defines the variables used in the body; the right hand side of the dot. 

Functions can be applied to expressions. That is, the argument expression substitutes the variable in the function body. Evaluating application `(λx.x)y` yields `y` as result. This means `λx.x` is the identity function. The lambda calculus is equivalent to another, more well known model of computation: The Turing machine by Alan Turing.

The Turing machine has a state and symbols on an infinite tape. A table contains the instructions commanding the machine what compute, given a symbol read from the tape and the current state. The machine might erase or write a symbol, then move the head, assume a new state or remain in the current.

The main purpose of the Turing machine was not to be a suitable model for implementing programming languages but to answer general computation questions. For example, Turing was able to show that the [Entscheidungsproblem](https://en.wikipedia.org/wiki/Entscheidungsproblem) has no general solution. 

Turing's research in this area resulted in a new field of study: The computability theory. In fact, the term *turing complete* refers to the property of a logical system to be able to compute all functions a Turing machine can compute. This means, if two models of computations are turing complete, then one can emulate the other.

Church was the PhD supervisor of Turing. This means the lambda calculus was known to Turing while he invented the Turing machine. Have a look [here](https://www.quora.com/Did-Alan-Turing-ever-meet-Alonzo-Church) for more information about the relationship between Church and Turing.

For further information about models of computation, please see also [Models of Computation](https://cs.brown.edu/people/jes/book/pdfs/ModelsOfComputation.pdf).   

# Formal semantics
One of the crucial points is that a model of computation can be seen as a tool for describing the formal semantics of a programming language. A formal semantic is a set of semantic functions which assigns meaning to a syntactical correct program. Formal hereby means, that the rules are unmistakably expressed and often with the help of mathematics. For example, if a programming language adheres to the lambda calculus we know how to compute the result by applying the conversion rules. The conversion rules are the semantic functions because they express what the symbols in an expression mean.

In Java, the [Java memory model](https://en.wikipedia.org/wiki/Java_memory_model) provides the semantics of the Java programming language. It is the answer to the question of how to correctly assign meaning to an imperative program if it is computed concurrently.  

# Meaning of state
The notion of state in programming languages often sparks a lot of debate in the programming community. From a theoretical point of view, the lambda calculus does not have a global mutable state. There is just applying conversion rules to expressions until no more rules can be applied. In real functional languages this leads to a problem. If we can only use conversion on expressions, how do we model input and output and still adhere to the lambda calculus? In other words, how does a functional language with I/O stay *pure*?

This question lead to a debate what purity actually means and how it can be defined to capture the intention of only using expressions. *Referntial transparancy*, a concept borrowed from the analytical philosophy, is often used to define *purity*. A typical definition looks like follows:

> An expression e is referentially transparent if, for all programs p, all occurrences of e in p can be replaced by the result of evaluating e without affecting the meaning of p.  A function f is pure if the expression f(x) is referentially transparent for all referentially transparent x. -- Rúnar Bjarnason and Paul Chiusano, Functional Programming in Scala, page 10

This sounds reasonable. Though, there is a subtlety which Rúnar and Paul point out in their [companion booklet](http://blog.higher-order.com/assets/fpiscompanion.pdf). What is the *meaning* of a program? Rúnar and Paul explain that the meaning of a program depends on "how we interpret or evaluate it, and whether some effect of the evaluation is to be considered a side effect depends on the observer. [...] So ultimately what we consider to break referential transparency depends very much on what we can or care to observe." 

In the "Functional Programming in Scala" book, they continue to argue that "a mutation that happens inside a function is not a side effect if nothing outside the function refers to the mutated state". For example, an imperative quicksort function with a mutable array is pure if the code outside does not hold a reference to the array. 

However, I/O is a side effect which is observable by the outer code when performed by a function. It breaks referential transparency.

# On referential transparency
As usual when concepts are borrowed from other fields the definition changes during the transfer. In his throughout answer to a [Stack overflow question](http://stackoverflow.com/questions/210835/what-is-referential-transparency) Uday Reddy points out the linguistic and semantic background of referential transparency. Informally defined, two expressions a and b are referential transparent if they refer two the same entity e and a can be replaced by b in a context c without altering the meaning. Uday exemplifies this definition. 

`The Scottish Parliament meets in the capital of Scotland.`

`The Scottish Parliament meets in Edinburgh.`

The terms *capital of Scotland* and *Edinburgh* refer to the same entity. In the context of *The Scottish Parliament...* they are replaceable by each other and, thus, referential transparent. However, in the context of *Edinburgh has been the capital of Scotland since 1999.* this replacement is not possible without altering the meaning. The terms are not referential transparent in this context.

Uday explains that the differences between the definitions of referential transparency in functional languages and analytical philosophy are the concepts of evaluation and value versus the concept of reference. While values are the result of evaluation and only exist inside a language, references refer to an actual referent which exists outside the language. For example, Edinburgh is a reference to the actual city in Scotland which exists, even without a reference.
