---
layout: post
title: "Combinator Pattern with Java 8"
modified: 2016-07-28
categories: [Code] 
description: "Compose simple primitives into complex structures"
tags: [Java 8, Functional Programming]
comments: true
share: true
---
The [Combinator Pattern](https://wiki.haskell.org/Combinator_pattern) is well-known in functional programming. The idea is to combine primitives into more complex structures. At my last talk at the [majug](http://www.majug.de/2016/07/21/functions-for-a-greater-good/) I presented a way of how to employ this pattern with Java 8. In this post we will have a look at this design.

## Before we start 
If you know functions and higher-order functions in Java 8, then you are more than prepared to read this post. If not, I recommend you to look at this [presentation](https://github.com/lambdafy/presentation). There you will find a nice introducution to functions in Java 8 by [Artem](https://www.twitter.com/_akozlov). The **code** can be found [here](https://github.com/gtrefs/blog-combinator).

# On primitives and combinators
Like many constructs in functional programming, primitives and combinators are names for abstractions. Wikipedia provides a description for language primitives:

> In computing, language primitives are the simplest elements available in a programming language. [...] an atomic element of an expression in a language. 
> -- [Wikipedia](https://en.wikipedia.org/wiki/Language_primitive) 

Likewise, primitives used in the combinator pattern are the simplest elements within a domain. For example, addition and multiplication in the domain of integers. Combinators provide means to compose primitives and/or already composed structures into more complex structures. For example, combining multiplication and addition. Due to the declarative nature of this pattern, the code itself employs a lot of domain terms.

# Validation with combinators
Let's pretend we are tasked to validate users. A user is valid if the name is not empty and the email contains an `@` sign. A straight-forward way to model this, is to add query methods to the corresponding entity:

```java
public class User{
   public final String name;
   public final int age;
   public final String email;

   public User(String name, int age, String email){
     this.name = name;
     this.age = age;
     this.email = email; 
   }

   public boolean isValid(){
     return nameIsNotEmpty() && eMailContainsAtSign();
   }

   private boolean nameIsNotEmpty(){
     return !name.trim().isEmpty();
   }

   private boolean eMailContainsAtSign(){
     return email.contains("@");
   }
}

new User("Gregor", 30, "nicemail@gmail.com").isValid(); // true
```

While this reads well, there are some disadvantages. What happens when the validation rules change? For example, for some use cases users must be older than 14. In this case, we need to adapt the `isValid` method. So this looks like a violation of the [SRP](https://en.wikipedia.org/wiki/Single_responsibility_principle). Before we refactor and extract the validation part into an own class, let us sit back and think about what we want to express. A user is valid if and only if a given set of rules are satisified. A rule describes the properties a user must have. Different rules can be combined to more complex ones. Applying a rule yields a validation result which describes the outcome. For now, let us assume that a `true` boolean value indicates that a user is valid. In more functional terms, we can express this in the following way:  

```java
public interface UserValidation extends Function<User, Boolean>{}
UserValidation nameIsNotEmpty = user -> !user.name.trim().isEmpty();
UserValidation eMailContainsAtSign = user -> user.email.contains("@");
User gregor = new User("Gregor", 30, "nicemail@gmail.com");

nameIsNotEmpty.apply(gregor) && eMailContainsAtSing.apply(gregor); // true
```

There is some boilerplate code here. Why do we have to create the `nameIsNotEmpty` and `eMailContainsAtSign` rules by ourselves? Why do we have to combine the results manually? Why are there 4 lines of initialization code for just one application line? Although the code is a bit wordy, it contains all building blocks for the combinator pattern: `nameIsNotEmpty` and `eMailContainsAtSign` are primitives and `&&` is a combinator. With Java 8 we are able to move primitives and combinators into the `UserValidation` type. 

```java
interface UserValidation extends Function<User, Boolean> {
    static UserValidation nameIsNotEmpty() {
        return user -> !user.name.trim().isEmpty();
    }

    static UserValidation eMailContainsAtSign() {
        return user -> user.email.contains("@");
    }

    default UserValidation and(UserValidation other) {
        return user -> this.apply(user) && other.apply(user);
    }
}

User gregor = new User("Gregor", 30, "nicemail@gmail.com");
nameIsNotEmpty().and(eMailContainsAtSign()).apply(gregor); // true
```

Primitives are translated to static methods and combinators to default methods. Both primitives, `nameIsNotEmpty` and `eMailContainsAtSign`, return a lambda expression with `UserValidation` as target type. Different to primitives, the `and` combinator composes `UserValidation`s by using the `&&` operator. There are two important observations in using this pattern. First, there is no application during combining time. This means, one first constructs a `UserValidation` and then executes it. This distinction makes one `UserValidation` applicable for many users. Second, `UserValidation` has no own state. This means, one `UserValidation` can be applied in a parallel environment without race conditions. For example, the following code conducts user validations in parallel.   

```java
List<User> users = findAllUsers()
    .stream().parallel()
    .filter(nameIsNotEmpty().and(eMailContainsAtSign())::apply) // to Predicate
    .collect(Collectors.toList());
```

# Validation result reasoning
Do you think that `Boolean` is a reasonable choice as type for validation results? In general, built-in language types are often a bad choice for representing domain information. Two reasons why: First, in Java it is hardly possible to adapt code we don't own. We cannot add query methods to the `Boolean` type. This makes it hard to determine which rules invalidated the result. Second, the semantics of a built-in type is implicit and context-specific. For example, in the validation context `true` means that a user is valid; i.e. all desired properties are present. This assumption might not hold, when we pass validation results to other contexts. There is a need for a new type `ValidationResult` containing the reason why a user is invalid under a specific rule.

```java
interface ValidationResult{
    static ValidationResult valid(){
        return ValidationSupport.valid();
    }
    
    static ValidationResult invalid(String reason){
        return new Invalid(reason);
    }
    
    boolean isValid();
    
    Optional<String> getReason();
}

private final static class Invalid implements ValidationResult {

    private final String reason;

    Invalid(String reason){
        this.reason = reason;
    }

    public boolean isValid(){
        return false;
    }

    public Optional<String> getReason(){
        return Optional.of(reason);
    }

    // equals and hashCode on field reason
}

private static final class ValidationSupport {
    private static final ValidationResult valid = new ValidationResult(){
        public boolean isValid(){ return true; }
        public Optional<String> getReason(){ return Optional.empty(); }
    };

    static ValidationResult valid(){
        return valid;
    }
}
```  
A `ValidationResult` is either `valid` or `Invalid`. The equality of two `ValidationResult`s is defined by the equality of their properties and types; i.e. two `Invalid` instances are equal if and only if they have the same invalidation reason. An anonymous singleton instance yields the same meaning in the `valid` case. This means, `valid` and `Invalid` are [value object](https://en.wikipedia.org/wiki/Value_object)s. In order to use the new `ValidationResult`, we have to adapt `UserValidation`.

```java
interface UserValidation extends Function<User, ValidationResult> {
    static UserValidation nameIsNotEmpty() {
        return holds(user -> !user.name.trim().isEmpty(), "Name is empty.");
    }
  
    static UserValidation eMailContainsAtSign() {
        return holds(user -> user.email.contains("@"), "Missing @-sign.");
    }

    static UserValidation holds(Predicate<User> p, String message){
        return user -> p.test(user) ? valid() : invalid(message);
    }

    default UserValidation and(UserValidation other) {
        return user -> {
            final ValidationResult result = this.apply(user);
            return result.isValid() ? other.apply(user) : result;
        };
    }
}

UserValidation validation = nameIsNotEmpty().and(eMailContainsAtSign());
User gregor = new User("", 30, "mail@mailinator.com");

ValidationResult result = validation.apply(gregor);
result.getReason().ifPresent(System.out::println); // Name is empty.
```
Primitive `holds` tests a given `Predicate` and, either, returns `valid` or a new `Invalid`. Primitives `nameIsNotEmpty` and `eMailContainsAtSign` have been refactored accordingly. The `and` combinator only computes the result of `other` if `this` yields `valid`. For example, `eMailContainsAtSign` is not evaluated for a given user, if `nameIsNotEmpty` is already `Invalid`. From a developers perspective this little API change conveys much more information compared to a plain `Boolean` value. However, there are some flaws in the design. For example, in a web context one would like to evaluate all validation rules in order to have an extensive report, but the `and` operator is short-circuiting. Thus, there is a need for an `all` combinator evaluating all validation rules. 

{::options parse_block_html="true" /}
<div class="exercise">
Implement an `all` combinator which executes all given `UserValidation`s and gathers the results. Use the following method declaration.

```java
static UserValidtion all(UserValidation... validations){ 
  // Your code here 
}
```
</div>

# Summary
This post presented a possible design for the combinator pattern in Java 8. We discussed primitives and combinators as building blocks. A validation use case illustrated how the pattern is applied to a specific domain. Last, we reasoned about the result of a validation. There are some benefits of a well-applied combinator pattern. From a developers perspective the API is made of terms from the domain. For example, primitives `nameIsNotEmpty` and `eMailContainsAtSign` clearly state their purpose. It strengthens the Single Responsibility Principle by enabling an easier seperation of concers. Further, there is a clear distinction between combining and application phase. One first constructs an instance and then executes it. This makes the pattern applicable in a parallel environment.
