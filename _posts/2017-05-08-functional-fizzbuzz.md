---
layout: post
title: "Six Ways to functional FizzBuzz with Vavr"
modified: 2017-05-08
categories: [Code] 
description: "Functional programming solutions for the FizzBuzz Kata using vavr and common FP features like streams, pattern matching, and combinator."
tags: [Java 8, Functional Programming, Vavr]
comments: true
exclude-seo: true
canonical-url: https://www.sitepoint.com/functional-fizzbuzz-with-vavr 
share: true
---
Sometime last year I stumbled over the excellent post of [twenty ways to do FizzBuzz](http://ditam.github.io/posts/fizzbuzz/) in JavaScript.
I asked myself, how some functional solutions in Java 8 and [vavr](http://www.vavr.io/) could look.
In this article I present 6 different solutions using functional data structures, higher order functions and property checking.
All these solutions adhere to the test harness I introduced in my [last article](https://www.sitepoint.com/property-based-testing-with-vavr/).

{::options parse_block_html="true" /}
<div class="message">
This article was originally published on [sitepoint.com](https://www.sitepoint.com/functional-fizzbuzz-with-vavr).
</div>

## Before We Get Started

My motivation behind this article was to experiment with different functional approaches to get used to a more functional thinking.
The presented solutions not necessarily line up in a linear procession from bad to good.
They are just different ways to skin the same cat and have different trade-offs, which is exactly what I want to explore.
If you know another solution or want to discuss a presented one, please leave a comment.

Before we start with the actual functional solutions we will look at an imperative one and discuss its responsibilities.
Further, we do some functional reasoning and introduce the stream data structure which we will use extensively.
Last, the properties a FizzBuzz function should fulfill are discussed.   

### Imperative Solution and Functional Reasoning

A simple imperative solution exactly states how to count from 1 to 100, how to compute the modulo, and how to decide what is printed when.
It is like following a recipe.

```java
public void fizzBuzzFrom1to100(){
    for(int i = 1; i <= 100; i++){
        if (i % 3 == 0 && i % 5 == 0) {
            System.out.println("FizzBuzz");
        } else if (i % 3 == 0) {
            System.out.println("Fizz");
        } else if (i % 5 == 0) {
            System.out.println("Buzz");
        } else {
            System.out.println(i);
        }
    }
}
```

From an outside perspective, this method does not exhibit any behavior other than printing to the console.
It is quite tedious to test because in order to verify the computed results we need to mock `System.out` and count the calls.
But in order to do the mocking we need to know the internals.

When we already risk a look inside the method we see that it is responsible for computing and displaying the result.
We can factor the logic of computing into an own function and use a secondary method on an outer layer to print the result.

```java
public List<String> fizzBuzz(int amount) {
    List<String> ret = new ArrayList<>();
    for (int i = 1; i <= amount; i++) {
        if (i % 3 == 0 && i % 5 == 0) {
            ret.add("FizzBuzz");
        } else if (i % 3 == 0) {
            ret.add("Fizz");
        } else if (i % 5 == 0) {
            ret.add("Buzz");
        } else {
            ret.add(String.valueOf(i));
        }
    }
    return ret;
}

public void printList(List<String> list){
    for(String element: list){
        System.out.println(element);
    }
}

printList(fizzBuzz(100));
```

Testing the `fizzBuzz` function becomes easy because everything the function does is represented by the returned list.
We can check any element in the list for its value.

Given a particular input, the `fizzBuzz` function will always produce the same output.
We could replace function application `fizzBuzz(100)` with the computed list without changing the meaning of our program.
That is, the `fizzBuzz` function is *pure*.

The reasoning is, that a method with side effects can always be factored into a pure function and two side-effecting methods supplying input and coping with output.
That is, we push the side effects to the boundaries of our application.
All presented solutions follow this principle by shifting the responsibility of iteration and displaying to vavr's stream data structure.
The shell code of these functions looks as follows.

```java
public static void main(String... args){
    fizzBuzzStream(100).forEach(System.out::println);
}

private static Stream<String> fizzBuzzStream(int amount){
    return Stream.from(1)
        .map(fizzBuzz())
        .take(amount);
}
```

In the next section we have a deeper look on streams, but let me point out the important parts of this code.
First, in the `fizzBuzzStream` function we create a stream of integers which counts `from` 1 on.
Then, we `map` these integers with the help of a `fizzBuzz` function which computes a `FizzBuzz` element for a given number.
This is the part most solutions will implement.
Next, we convert the infinite stream to a finite one by taking `amount` elements.
In the `main` method these elements are printed onto the console.

Notice how the stream hides details like iteration and creation.
We operate on a very declarative level and become very concise in expressing our program.
Also notice that the side effect of printing the elements is the last action that is performed.
Effectively it is the most outer layer in our program.

### vavr Stream

The [functional data structure](http://www.vavr.io/vavr-docs/#_data_structures_in_a_nutshell) `Stream` may be `Empty` or a `Cons`.
A `Cons` consists of a head element and a lazy computed tail `Stream`.
An infinite `Stream` does not contain an `Empty` tail.

We write `Stream.cons(1, () -> Stream.empty())` to define a stream which contains 1 as head and an empty tail.
Following this notation, defining an infinite stream of 1s may look like this: `Stream.cons(1, () -> Stream.cons(1,...))`.
However, this is not valid syntax in Java and would be rather tedious.
Therefore, we employ a recursive function which computes the next 1 on demand.

```java
Stream<Integer> ones() {
    return Stream.cons(1, () -> ones());
}
```

Java [evaluates method arguments](https://www.sitepoint.com/java-in-praise-of-laziness/) before a method is called.
In case of an infinite stream this is tricked with a `Supplier` in order to prevent a stack overflow.
This also allows a memory efficient representation of a stream because just the head is kept in memory.
This differs from a `List` which contains all elements and from a Java 8 stream which is a composed aggregate operation on a sequence of data elements from some source.

### Test Harness

There are three invariants, a stream of FizzBuzz elements should satisfy:

* Every third element should start with Fizz
* Every Multiple of 5 must end with Buzz
* Every non-third and non-fifth Element should remain a number

In property based testing (PBT), each invariant is combined with a sample generator forming a property.
For example, in case of the Fizz invariant, a generator creates multiples of 3.
For each generated sample, the invariant is checked.
If one check yields false, the property is said to be *falsified*.
Otherwise, it is called *satisfied*.

For a more thorough introduction and a build-up of the FizzBuzz test harness have a look at my article on [property based testing with vavr](https://www.sitepoint.com/property-based-testing-with-vavr/).

## 6 Ways to FizzBuzz

The first three solutions are about writing pure functions computing FizzBuzz elements.
At their core, each of these functions maps a number to a word.
This information is exposed by their type: `Function<Integer, String>`.
In fact, any of the three implementations could be used to replace any other one.
That is, they are [referentially transparent](https://www.sitepoint.com/what-is-referential-transparency).

In the fourth and fifth solution we have a look at how functions on streams can be employed to create a FizzBuzz solution.
We always stay in the stream context and the return type is `Stream<String>`.

The sixth and last solution explores another approach contributed by [Pierre-Yves](https://www.sitepoint.com/author/pysaumont/).

### 1. A Simple Function

This is the computational core of the imperative solution.
Depending on the modulo operations, the corresponding FizzBuzz element is returned.

```java
static Function<Integer, String> fizzBuzz(){
    return i -> {
        if (i % 3 == 0 && i % 5==0) {
            return "FizzBuzz";
        } else if (i % 3 == 0) {
            return "Fizz";
        } else if (i % 5 == 0) {
            return "Buzz";
        } else {
           return i.toString();
        }
   }
}
```

What I like about this solution is, that it is pretty straight forward and that it shows that a pure function can be factored out of a side-effecting method.
However, this solution is still quite inflexible and looks like an imperative solution.
For example, if we want to add another case for replacing every multiple of 4 by *Bizz* we need to add another clause.

### 2. Pattern Matching

This solution differs from the first one by employing pattern matching instead of if-statements.
Think of [pattern matching](http://blog.vavr.io/pattern-matching-essentials/) as a [switch-statement](https://www.sitepoint.com/javas-switch-statement/) on steroids that examines the structure of an expression and has no *fall-through* semantics.

```java
static Function<Integer, String> withPatternMatching(){
    return e -> Match(Tuple.of(e % 3, e % 5)).of(
        Case(Tuple2($(0),$(0)), "FizzBuzz"),
        Case(Tuple2($(0), $()), "Fizz"),
        Case(Tuple2($(), $(0)), "Buzz"),
        Case($(), e.toString())
    );
}
```

We match the tuple of the computational result against patterns described in cases.
If a case matches, the string on the right side is returned.
For example, if a number is both divisible by 3 and by 5, `"FizzBuzz"` is returned.

This solution is more concise than the first one and we got rid of the explicit if-statements.
However, we became even less flexible.
For example, to test for another number we now need to extend the match clause and add another case.

### 3. Combinator Pattern

In this solution I used the [combinator pattern](https://gtrefs.github.io/code/combinator-pattern/).

```java
static Function<Integer, String> withCombinator(){
    return fizzBuzz()
        .orElse(fizz())
        .orElse(buzz())
//      .orElse(word("Bizz").ifDivisibleBy(4))
        .orElse(number());
}

interface NumberToWord extends Function<Integer, String> {

    static NumberToWord fizzBuzz(){
        return fizz().and(buzz());
    }

    static NumberToWord fizz(){
        return word("Fizz").ifDivisibleBy(3);
    }

    static NumberToWord buzz(){
        return word("Buzz").ifDivisibleBy(5);
    }

    static NumberToWord word(String word){
        return i -> word;
    }

    static NumberToWord number(){
        return Object::toString;
    }

    default NumberToWord ifDivisibleBy(int modulo){
        return i -> i % modulo == 0 ? apply(i) : "";
    }

    default NumberToWord orElse(NumberToWord other){
        return i -> {
            String result = apply(i);
            return result.isEmpty()
                ? other.apply(i)
                : result;
        };
    }

    default NumberToWord and(NumberToWord other){
        return i -> {
            String first = this.apply(i);
            String second = other.apply(i);
            return (first.isEmpty() || second.isEmpty())
                ? "";
                : (first + second);
        };
    }
}
```

The combinator pattern helps us to write a little embedded [domain specific language](https://en.wikipedia.org/wiki/Domain-specific_language) with terms from the FizzBuzz domain.
Even more, the language is shaped in a way that allows us to add new words for numbers without changing the core `WordToNumber` type.

I like how declarative this solution is, but there is a loss in conciseness.
The increase in readability (assuming you internalized the combinator pattern) compensates this more than enough, though.

### 4. Zipped Streams

In this solution we have a different perspective on the FizzBuzz problem that allows us to compute no element at all.
The idea is to see each FizzBuzz subsequence, for example the one going `nothing, nothing, "Fizz"`, as its own stream.

```java
static Stream<String> withZippedStreams(){
    return Stream.of("","","Fizz").cycle()
            // creates a Stream<Tuple2<String, String>>
            .zip(Stream.of("","","","","Buzz").cycle())
            // creates a Stream<Tuple2<Tuple2<String, String>, Integer>>
            .zip(Stream.from(1))
            // flattens Tuple2<Tuple2<String, String>, Integer> to String
            .map(flatten());
}

private static Function<Tuple2<Tuple2<String, String>, Integer>, String>
        flatten() {
    return tuple -> {
        String fizzBuzz = tuple._1._1 + tuple._1._2;
        String number = tuple._2.toString();
        return fizzBuzz.isEmpty()
            ? number
            : fizzBuzz;
    };
}
```

Every third element should be Fizz is expressed as a stream which cycles over a sequence of two empty elements and the third element being "Fizz".
This results into a stream which repeats this sequence infinitely.
The same idea applies to the to the Buzz element.

Both streams are merged using the `zip` function.
The name of the function states what it does:
It binds two streams into a stream of tuples (see also [zipper](https://en.wikipedia.org/wiki/Zipper)).

This zipped stream is merged again with another stream containing the integers from `1` on.
The `flatten` function maps the result by deciding what the actual element should be.
For example, if both elements from the Buzz and Fizz stream are empty, the element becomes the number.

I like this solution because it follows a thought outside the box.
What if everything I have is a stream?

### 5. Zipped Transformation Stream

Different from the previous solution we are now zipping streams of functions.

```java
static Stream<String> withFunctions(){
    return fizzBuzzTransformations()
            .zip(Stream.from(1))
            .map(functionAndNumber ->
                    functionAndNumber._1.apply(functionNumber._2));
}

private static Stream<Function<Integer, String>> fizzBuzzTransformations() {
    Function<Integer, String> empty =  i -> "";
    return Stream.of(empty, empty, i -> "Fizz").cycle()
            // creates Stream<Tuple2<Function<Int, Str>, Function<Int, Str>>>
            .zip(Stream.of(empty, empty, empty, empty, i -> "Buzz").cycle())
            // flattens Tuple2<Function<Int, Str>, Function<Int, Str>>
            // to Function<Int, Str>
            .map(flatten());
}

private static Function<
            Tuple2<Function<Integer, String>, Function<Integer, String>>,
            Function<Integer, String>>
        flatten() {
    return functions -> i -> {
        String fizzBuzz = functions._1.apply(i) + functions._2.apply(i);
        return fizzBuzz.isEmpty()
            ? i.toString()
            : fizzBuzz;
    };
}
```

The Fizz element, for example, is translated to a stream which cycles over a sequence of three functions.
The first two elements, are the empty function which returns nothing for a given integer.
The last element, is the Fizz function which returns Fizz for a given number.
The Buzz element is translated equivalently.

The interesting part is how we map the stream of tuples of functions.
For a given tuple we return a function which takes a number and applies the functions in the tuple.
That is, for each tuple in the stream we compose a new function which uses the previous functions for deciding what the actual element is.
For example, if both elements are the empty function, the actual element becomes the number.
This is similar to the `flatten` function from the previous solution.

The resulting stream of composed functions is zipped with a stream which counts from `1` on.
The only thing left is to apply the function in each tuple to the number.

What I very much like about this solution is how it displays that functions are values which can be passed around.

### 6. A Wheel

During our [internal review session](https://www.sitepoint.com/behind-the-scenes-sitepoints-peer-review-program/), [Pierre-Yves](https://www.sitepoint.com/author/pysaumont/) noticed and pointed out that a *wheel* is the underlying pattern of the two stream based solutions.
Wheel is a metaphor for a periodic sequence of elements.
The repeated subsequence starts from *1* and ends with the first *FizzBuzz*.
When you distribute the elements of the subsequence equally over a wheel, then determining the next element is advancing the wheel to the next element.
Pierre-Yves visualized this idea with the following code.

```java
public static void main(String... args) {
    String[] wheelArray = new String[] {
            "", "", "Fizz", "", "Buzz",
            "Fizz", "", "", "Fizz", "Buzz",
            "", "Fizz", "", "", "FizzBuzz"};
    Stream<String> fizzBuzz = fizzBuzzStream(wheelArray);
    fizzBuzz.take(30).forEach(System.out::println);
}

private static Stream<String> fizzBuzzStream(String[] wheelArray) {
    return Stream
            .iterate(new Wheel(wheelArray), Wheel::next)
            .map(Wheel::getValue);
}

static class Wheel {

    private final String[] wheel;
    private final int numValue;
    private final String value;

    private Wheel(String[] wheelArray) {
       this(wheelArray, 0);
    }

    private Wheel(String[] wheelArray, int n) {
        this.wheel = wheelArray;
        this.numValue = n + 1;
        int index = n % wheel.length;
        this.value = wheel[index].length() == 0
            ? Integer.toString(numValue)
            : wheel[index];
    }

    private String getValue() {
        return this.value;
    }

    private Wheel next() {
        return new Wheel(this.wheel, this.numValue);
    }
}
```

The interesting part is the use of an immutable class `Wheel` to keep track of the state and [method references to an instance method of an arbitrary type](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html).
The wheel takes an initial sequence of elements.
It then computes the numeral value, the index, and the value determined by the position of the wheel.
If the element in the wheel array at the index position is not empty, it becomes the value.
Otherwise, the numeral value becomes the value.

The constructed stream iterates wheels, returned by the `next` function, each being one step further compared to the previous one.
This infinite stream of wheels is then mapped to the computed FizzBuzz element using `Wheel::getValue`.

## Summary

The FizzBuzz problem is easy to understand.
Nonetheless, there are manifold ways to solve it.
In this article we concentrated on functional solutions using the [vavr](http://www.vavr.io/) library.

We first discussed an imperative solution and how we can factor a functional core.
This led us to a more general discussion about how to reason with functional programs.
The key insight here is, that every non-functional method can be transformed into a pure function and two methods which supply input and cope with output.
Side effects are pushed to the boundaries of our program.

In the second part of the article we had a look on 6 different solutions for the FizzBuzz problem.
The first three solutions focused on computing the FizzBuzz element for a given number while the last two put their focus on using functions on streams.
All solutions had some advantages and disadvantages.
The key insight here is, that thinking outside the box may lead to unexpected solutions.

Do you know another functional FizzBuzz solution or a simple problem which might be solved in different ways?
Leave a comment!
