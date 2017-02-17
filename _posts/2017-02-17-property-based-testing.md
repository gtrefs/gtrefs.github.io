---
layout: post
title: "Property Based Testing with Javaslang"
modified: 2017-02-17
categories: [Code] 
description: "Learning property based testing, or property checking, in Java using Javaslang and the FizzBuzz Kata."
tags: [Java 8, Functional Programming, Javaslang]
comments: true
exclude-seo: true
canonical-url: https://www.sitepoint.com/property-based-testing-with-javaslang/
share: true
---
Usual unit testing is based on examples: We define the input for our unit and check for an expected output.
Thus, we can ensure that for a specific sample our code works right.
However, this does not prove the correctness for all possible inputs.
Property based testing, also called property checking, takes another approach by adhering to the scientific theory of [Falsifiability](https://en.wikipedia.org/wiki/Falsifiability):
As long as there is no evidence that a proposed property of our code is false, it is assumed to be true.
In this article we are using [Javaslang](http://www.javaslang.io/)'s API for property based testing to verify a simple FizzBuzz implementation.

{::options parse_block_html="true" /}
<div class="message">
This article was originally published on [sitepoint.com](https://www.sitepoint.com/property-based-testing-with-javaslang/).
</div>

## Before We Start

I am using Javaslang `Stream`s to generate an infinite FizzBuzz sequence which should satisfy certain properties.
Compared with Java 8 `Stream`s, the main benefit in the scope of this article is that the Javaslang variant provides a good API to retrieve individual elements with [the `get` function](http://static.javadoc.io/io.javaslang/javaslang/2.0.3/javaslang/collection/Stream.html#get-int-).

## Property Based Testing

Property based testing (PBT) allows developers to specify the high-level behaviour of a program in terms of [invariants](https://en.wikipedia.org/wiki/Invariant_(computer_science)) it should fulfill.
A PBT framework creates test cases to ensure that the program runs as specified.

A _property_ is the combination of an invariant with an input values generator.
For each generated value, the invariant is treated as a predicate and checked whether it yields true or false for that value.

As soon as there is one value which yields false, the property is said to be _falsified_ and checking is aborted.
If a property cannot be falsified after a specific amount of sample data, the property is assumed to be _satisfied_.

### Reverse List Example

Take a function that reverses a list as an example.
It has several implicit invariants that must hold in order to provide proper functionality.
One of them is, that the first element of a list is the last element of the same list reversed.
A corresponding property combines this invariant with a generator which generates lists with random values and lengths.

Assume, that the generator generates the following input values: `[1,2,3]`, `["a","b"]` and `[4,5,6]`.
The list reverse function returns `[3,2,1]` for `[1,2,3]`.
The first element of the list is the last element of the same list reversed.
The check yields true.
The list reverse function returns `["a", "b"]` for `["a", "b"]`.
The first element of the list is **not** the last element of the same list reversed.
The check yields false.
The property is falsified.

In Javaslang the default amount of tries to falsify a property is 1000.

## Property Checking FizzBuzz

FizzBuzz is a child's play counting from 1 on.
Each child in turn has to say the current number out loud.
If the current number is divisible by 3 the child should say "Fizz", if it is divisible by 5 the answer should be "Buzz" and if it is divisible by both 3 and 5 the answer should be "FizzBuzz".
So the game starts like this:

> 1, 2, Fizz, 4, Buzz, Fizz, 7, 8, Fizz, Buzz, 11, Fizz, 13, 14, FizzBuzz, ...

FizzBuzz is often taken as a sample exercise in job interviews where applicants should fizzbuzz the numbers from 1 to 100.
FizzBuzz can also be used as a Kata simple enough to focus on the surroundings like language, libraries or tooling instead of the business logic.

A stream seems to be a good way to model FizzBuzz.
We will start with an empty one and gradually improve it to pass our tests.

```java
public Stream<String> fizzBuzz() {
    Stream.empty();
}
```

### Every Third Element Should Start with Fizz

From the description of FizzBuzz we can derive a first property about the FizzBuzz `Stream`: Every third element should start with _Fizz_.
Javaslang's property checking implementation provides fluent APIs to create generators and properties.

```java
@Test
public void every_third_element_starts_with_Fizz) {
    Arbitrary<Integer> multiplesOf3 = Arbitrary.integer()
        .filter(i -> i > 0)
        .filter(i -> i % 3 == 0);

    CheckedFunction1<Integer, Boolean> mustStartWithFizz = i ->
        fizzBuzz().get(i - 1).startsWith("Fizz");

    CheckResult result = Property
        .def("Every third element must start with Fizz")
        .forAll(multiplesOf3)
        .suchThat(mustStartWithFizz)
        .check();

    result.assertIsSatisfied();
}
```

Let's go through it one by one:

* `Arbitrary.integer()` generates arbitrary integers which are filtered for positive multiples of 3 using `filter`.
* The function `mustStartWithFizz` is the invariant:
  For a given index, the corresponding element in the stream (`get` is 0-indexed) must start with _Fizz_.
  The call to `suchThat` further below requires a `CheckedFunction1`, which is like a function but may throw a checked exception (something we do not need here).
* `Property::def` takes a short description and creates a `Property`.
  The call to `forAll` takes an `Arbitrary` and ensures that the invariant in `suchThat` must hold for all input values generated by the `Arbitrary`.
* The `check` function uses the generator to create multiples of 3 in the range from 0 to 100.
  It returns a `CheckResult` which can be queried about the result or be asserted.

The property is not satisfied, because for every generated multiple of 3 the corresponding element does not exist in the empty `Stream`.
Let's fix this by providing a suitable stream.

```java
public Stream<String> fizzBuzz() {
    return Stream.from(1).map(i -> i % 3 == 0 ? "Fizz": "");
}
```

`Stream::from` creates an infinite stream, which counts from the given number on.
Every multiple of 3 is mapped to _Fizz_.
The resulting stream contains a _Fizz_ for every third element.
Once we check the property again, it is satisfied and the result is printed on the console:

> Every third element must start with Fizz: OK, passed 1000 tests in 111 ms.

![Property Based Testing with Javaslang](https://dab1nmslvvntp.cloudfront.net/wp-content/uploads/2016/12/1482055944property-based-testing-Javaslang.jpg)

### Increase the Sample Size

Per default, `check` tries to falsify a property 1000 times.
Each time, the given `Arbitrary` generates a sample value which depends on a `size` parameter.
Per default, the size parameter is set to 100.
For `Arbitrary.integer()` the negative size parameter defines the lower bound and the positive size parameter defines the upper bound for the generated value.
This means, a generated integer has a value between -100 and 100.
We further limit the generated values to be only positive and to be multiples of 3.
This means, we test each multiple of 3 in the range from 1 to 100 approximately 30 times.

Fortunately, Javaslang allows us to adapt the size and the number of tries by providing corresponding parameters to the check function.

```java
CheckResult result = Property
	.def("Every third element must start with Fizz")
	.forAll(multiplesOf3)
	.suchThat(mustStartWithFizz)
	.check(10_000, 1_000);
```

Now, the generated multiples of 3 are drawn from a range between 1 and 10'000 and the property is checked 1000 times.

### Every Multiple of 5 Must End with Buzz

A second property which can be derived from the FizzBuzz description is that every multiple of 5 must end with _Buzz_.

```java
@Test
public void every_fifth_element_ends_with_Buzz() {
    Arbitrary<Integer> multiplesOf5 = Arbitrary.integer()
        .filter(i -> i > 0)
        .filter(i -> i % 5 == 0);

    CheckedFunction1<Integer, Boolean> mustEndWithBuzz = i ->
        fizzBuzz().get(i - 1).endsWith("Buzz");

    Property.def("Every fifth element must end with Buzz")
        .forAll(multiplesOf5)
        .suchThat(mustEndWithBuzz)
        .check(10_000, 1_000)
        .assertIsSatisfied();
}
```

This time, property checking and assertion are combined.

This property is falsified just like the one checking _Buzz_ was initially.
But this time not because there are no elements (since the last test the `fizzBuzz` function creates an infinite stream) but because every fifth element is an empty string or sometimes _Fizz_ but never _Buzz_.

Let's fix this by returning _Buzz_ for every fifth element.

```java
public Stream<String> fizzBuzz() {
    return Stream.from(1).map(i ->
        i % 3 == 0 ? "Fizz" : (i % 5 == 0 ? "Buzz" : "")
    );
}
```

When we check the property again, it is falsified after some passed tests:

> Every fifth element must end with Buzz: Falsified after 7 passed tests in 56 ms.

The message in the `AssertionError` describes the sample which falsified the property:

> Expected satisfied check result but was Falsified(propertyName = Every fifth element must end with Buzz, count = 8, sample = (30)).

The reason is, that 30 is divisible by 3 and by 5 but in the `fizzBuzz()` implementation above divisibility by 3 has a higher precedence than divisibility by 5.
However, a change in precedence does not solve the issue, because the first property that every third element should start with _Fizz_ would be falsified.
A solution is to return _FizzBuzz_ if it is divisible by 3 and 5.

```java
private Stream<String> fizzBuzz() {
    return Stream.from(1).map(i -> {
        boolean divBy3 = i % 3 == 0;
        boolean divBy5 = i % 5 == 0;

        return divBy3 && divBy5 ? "FizzBuzz" :
            divBy3 ? "Fizz" :
            divBy5 ? "Buzz" : "";
        });
}
```

Both properties are satisfied with this approach.

### Every Non-Third and Non-Fifth Element Should Remain a Number

A third property that can be derived from the FizzBuzz description is that every non-third and non-fifth element should remain a number.

```java
@Test
public void every_non_fifth_and_non_third_element_is_a_number() {
    Arbitrary<Integer> nonFifthNonThird = Arbitrary.integer()
        .filter(i -> i > 0)
        .filter(i -> i % 5 != 0)
        .filter(i -> i % 3 != 0);

    CheckedFunction1<Integer, Boolean> mustBeANumber = i ->
        fizzBuzz().get(i - 1).equals(i.toString());

    Property.def("Non-fifth and non-third element must be a number")
        .forAll(nonFifthNonThird)
        .suchThat(mustBeANumber)
        .check(10_000, 1_000)
        .assertIsSatisfied();
    }
```

The check of this property fails, because so far the `fizzBuzz` function maps non-_Fizz_ and non-_Buzz_ numbers to the empty String.
Returning the number solves this issue.

```java
private Stream<String> fizzBuzz() {
    return Stream.from(1).map(i -> {
	boolean divBy3 = i % 3 == 0;
	boolean divBy5 = i % 5 == 0;

	return divBy3 && divBy5 ? "FizzBuzz" :
		divBy3 ? "Fizz" :
		divBy5 ? "Buzz" : i.toString();
	});
}
```

### Other Properties

Checking these three properties is enough to shape the `fizzBuzz` function.
There is no need for a special _FizzBuzz_ property because this case is already covered by the _Fizz_ and _Buzz_ properties.
However, we could add another property which should *not* hold for all elements.
For example, there should be no white space between _Fizz_ and _Buzz_ in _FizzBuzz_.
I think this is a nice exercise and I leave it you, my dear reader, to define and check such a property.

There is another flaw in our tests, though:
Our properties are not specific enough.
Checking that an element starts with _Fizz_ or ends with _Buzz_ is also satisfied by a stream which just contains _FizzBuzz_ for any multiple of 3 and 5.
Can you fix our properties?

## Benefits and Disadvantages

As said before, PBT has the advantage that one can focus on specifying the expected behaviour of a program instead of defining each edge case.
This shifts testing to a more declarative level.

Compared with [other implementations](https://github.com/pholser/junit-quickcheck) Javaslang provides a nice fluent API which is unobtrusive regarding the used test framework.
It works with JUnit 4, [JUnit 5](https://www.sitepoint.com/junit-5-state-of-the-union/) and TestNG.
Spock has a BDD mindset and I am not sure if the term *specifcation* has the same meaning in Spock as in PBT but this could be explored in another article.
It seems to be a nice addition to the data-driven approach.

A disadvantage is that rare bugs may not be discovered on the first run and may brake later builds.
Further, coverage reports may differ from build to build.
However, this is negligible.

The crucial part of PBT is how well the underlying random number generator (RNG) performs and how well it is seeded.
A bad RNG yields a bad distribution of test samples and this may result in "blind spots" of the tested code.
Javaslang relies on `ThreadLocalRandom` and this is more than okay.

Javaslang currently has a limited amount of supported types.
It would be beneficial to also support generic subdomains like time and to provide a way to generate random entities and value objects of a core domain.

## Summary

We have seen that Property based testing is based on the theory of Falsifiability.
We learned that Javaslang has a fluent API which codifies this idea.
It enables us to define properties as the combination of an invariant and an input values generator.
Per default, an invariant is checked against 1000 generated samples.

This enabled us to think about FizzBuzz on more declarative level.
We defined and checked three properties and discussed the outcomes.

Last, we talked about benefits and disadvantages of PBT.
