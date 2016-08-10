---
layout: post
title: "CraftConf 2016 - Crafting Code Workshop"
modified: 2016-04-26
categories: [Conference] 
description: "Outside-In TDD with Sandro Mancuso"
tags: [TDD, Mock, CraftConf]
image: 
  feature: 26042016_craftconf/01_workshop_day1_poster.jpg
comments: true
share: true
---
The day is here: First time [CraftConf](http://craft-conf.com/2016/) ever. So far, Budapest is a pleasurable stay. I got to know Thomas who studies medicine in Hungary and drove me into the city. Further, the room I booked revealed itself as a complete flat with 2 rooms, kitchen and bathroom. Hamburg, your accomondations has nothing on that! For me, the conference started with the two-days workshop *Crafting Code* by [Sandro Mancuso](https://twitter.com/sandromancuso).

# Crafting Code
> This course is designed to help developers write well-crafted code—code that is clean, testable, maintainable, and an expression of the business domain. The course is entirely hands-on, designed to teach developers practical techniques they can immediately apply to real-world projects.

The leading motive of this course is coding exercises in changing pairs. Each participant introduced himself and shortly described his favorite programming language and TDD experience. In my modesty, I explained that my favourite language is Java (even though I am learning Scala right now) and that I took part in another TDD workshop. In total, Java, by far, was most frequently stated. But, there were some C#, Python and one Scala developers. 

## First exercise
{% gist gtrefs/48fd0f4a780f8dbdd0c190b71142307c %}

I did my first exercise with two other Java developers ([Peter](https://twitter.com/Sini_P) was one of them) and we all agreed that a Stack should reasonable easy to implement. Intuitionally, we started to test our Stack to throw an exception if it is popped when empty. After that, we were trapped in a dead lock. How can we independently test the `push` and `pop` methods without introducing new behaviour like a `size` method? Answer is: We can't. However, we can describe a property of the stack which must hold: The last object which was pushed must be popped first. Thus, our next test case should first push an object and then assert that the `pop` method returns this object. Thereby, there should not be a single line of production code which is not forced by a test case. For example, consider the following snippet:

{% gist gtrefs/cfc5075926088e643ec10506259bd7d5 %}

The easiest way to make this test green is to store the given object in an instance variable:

{% gist gtrefs/ba1b87447d1c19d10b3efc2e5b22d584 %}

Searching the problem domain, we found another property resulting into a good test case: Objects which have been pushed must be popped in reverse order. Expressed as test case, this property forces us to store the objects properly. From this exercise, I learned the following:

- Only use terms from the domain (return vs. pop)
- Don't introduce additional business behaviour because of test code (cf. size method)
- Tests should have a setup (does not matter if AAA or Given-When-Then)
- Describe the behaviour instead of testing single methods
- Go from simple to complex 
- Tests get more specific while the business code gets more generic
- Assert first, then Act, last Arrange
- Command: void; Query: return something
  - difficult to have methods which do both (exception: Front controller)
  - Commands are hard to test because they change state

## Second exercise

{% gist gtrefs/89c543185fe14c3181cb270a5c935808 %}

For this exercise I teamed up with the only Scala developer. We wrote tests in [Specs2](https://etorreborre.github.io/specs2/) which was not familiar to me. The first thing, I realized is that I have to do more Katas, because I recognized the exercise from before but could not remeber the solution. Again, we were quick in finding a simple test case (`RomanNumberConverter(1) == "1"`) but got stuck deriving more complex test-cases from the domain. This resulted into a lot of `if-else` expressions for different roman numbers. Sandro helped us by introducing the [Transformation Priority Premise](https://blog.8thlight.com/uncle-bob/2013/05/27/TheTransformationPriorityPremise.html). After we applied some of these transformations to our code we were able to see patterns. For example, consider the following snippet:

{% gist gtrefs/a7c661593567ee692b37776626112f13 %}

A roman literal is added to the result as long as the corresponding arabic number still fits in the intermediate conversion result. We ignored the excpetional case of having a leading lower number (for example, `IV`) and came up with the following code:

{% gist gtrefs/263e3ebd2bda065e96fd149d4daea539 %}

Leading lower number cases are reflected as own entries in the `arabicToRoman` list:

{% gist gtrefs/55ec228a5ceba08e6c7682adab18deef %}

From this exercise I learned:

- Specs2 seems to be a nice Testing Framework
- Write enough tests to reveal a pattern 
- Treat excpetions in the end
  - Don't overengineer your code before

## Third exercise 

{% gist gtrefs/ba8afd2085976df8fa2407876430538e %}

For this exercise I teamed up with one of the python guys. Unfortunately, I did not get a hold of the source code. Nontheless, I learned the following:

- Introduce types where necessary (`Year(2016).isLeap()`)?
 - Primitive types don't know anything about the domain (cf. `int` vs. `Year`)
- Value objects attract behaviour (cf. `isLeap`)

## Fourth exercise 
Java: Outside-In TDD

- Write tests first (maybe red)
- Outside: Outside classes model a process (How to call the components) while inner classes do the actual work. Instead to verify the outcome it is necessary to verify the process (process vs. state)
  - Test with mocks
- Diff to FP: The building block ins FP are small functions which are taken to compose more complex scenarious (Classist approach)

*To be continued*
