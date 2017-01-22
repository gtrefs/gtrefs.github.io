---
layout: post
title: "Java annotations as GitHub names"
modified: 2015-09-06
categories: [Java, GitHub] 
description: "Java annotations and GitHub name references"
tags: [Java, GitHub, Fun]
omments: true
share: true
date: 2015-09-06T11:23:20+02:00
---
Twitter, Facebook and GitHub have the name reference functionality in common. To mention another user, person and/or organization in your post, tweet, timeline and/or commit just type `@` and add the corresponding name. Escpecially, GitHub and Java have an interesting relationship because whenever a comment or commit containing a Java annotation is published an equally-named GitHub user is notified. In this post I exploit this by registering `FunctionalInterface` and `RestController` as GitHub users and by providing a little script to check for available Java-GitHub-Names.

# Names 
The first name I registered was `FunctionalInterface` which was introduced with Java 8 and marks an interface to have a single abstract method. In this way it may be used by the compiler for [type inference](https://docs.oracle.com/javase/specs/jls/se8/html/jls-18.html) for lambda expressions. Next, I registered `RestController` which is defined by the Spring MVC framework indicating classes as [Front Controllers](https://en.wikipedia.org/wiki/Front_Controller_pattern) for RESTful services. Some Java annotations were already registered as GitHub users. To check available names I created a little Python script.

{% gist gtrefs/a30a6e4c5c1e9514b10c %}

Each line in the annotations file is considered to be a user name. The list contains all annotations in Java SE 8 and some Spring annotations. Following 27 names were taken: After, Around, Aspect, Autowired, Before, Component, Order, Pointcut, Qualifier, Repeat, Repository, Required, Resource, Rollback, Scope, Service, Timed, Transactional, Deprecated, Documented, FunctionalInterface, Inherited, Override, Retention, SuppressWarnings, Target and RestController. 

# Results

|Annotation|Project|User|# mentions|Example link|
|----------|-------|----|------------------|----|
|FunctionalInterface|Eclipse JDT core|asbharadwaj|1|[commit](https://github.com/eclipse/eclipse.jdt.core/commit/7484e25e299b8203b28cf436ac2de883b2f5dd8e)|
|FunctionalInterface|sai|jmorwick|6|[commit](https://github.com/jmorwick/sai/commit/55f995ebc7bac1667049fb21e2048e5dad303634)|
|FunctionalInterface|ironrhino|zhouyanming|1|[commit](https://github.com/ironrhino/ironrhino/commit/aadd031b5e36fc3cac1bb7c2f7e8bf33cfc7ff03)|
|FunctionalInterface|spark|bentolor|5|[commit](https://github.com/bentolor/spark/commit/36b03a2601590dde015b57579125cce0bd1f9152)|
|FunctionalInterface|OTBProject|NthPortal|2|[commit](https://github.com/Thelonedevil/OTBProject/commit/1c5af3e041c03515630ff30cf91780bccda8357b)|
|FunctionalInterface|SpongeAPI|gabizo|1|[comment](https://github.com/SpongePowered/SpongeAPI/commit/8c3ed14457608ab755f1152766c414eb359f53d1#commitcomment-12986294)|
|FunctionalInterface|Eclipse Platform ui|vogella|1|[commit](https://github.com/eclipse/eclipse.platform.ui/commit/cc4af637dd218fe87cd60569aa699bd0a4d6b931)|
|RestController|kuitos.github.io|Kuitos|1|[issue](https://github.com/kuitos/kuitos.github.io/issues/9)|
|RestController|firstRest|prochiy|-|-|
|RestController|Spring|UnAmi|-|-|
|RestController|springfox|rcruzper|1|[issue](https://github.com/springfox/springfox/issues/934)|
|RestController|spring-integration-kafka|ashishsoni|1|[issue](https://github.com/spring-projects/spring-integration-kafka/issues/59)|
|RestController|exercise-spring-boot-integration-tests|ebragaparah|1|[commit](https://github.com/dudadornelles/exercise-spring-boot-integration-tests/commit/6a938b16c3b9b8783037a6b91a267fc5f27435dd)|
|RestController|find|matthew-gordon-h|1|[commit](https://github.com/hpautonomy/find/commit/c920c13dcdfefdc40cce324bef8ac2b9fefc7cab)|
|RestController|swagger4spring-web-jdk6|wkennedy|2|[commit](https://github.com/hardenCN/swagger4spring-web-jdk6/commit/dd3b8810f1063556a9346a9fe51708bf05a6b685)|
|RestController|spring-security-oauth|berrytchaks|1|[issue](https://github.com/spring-projects/spring-security-oauth/issues/568)|
|RestController|springside|shurun19851206|2|[commit](https://github.com/shurun19851206/springside/commit/2478aa8f3295964fc3eb2f71dba1d4ddd614a698)|

I would have assumed to get more notifactions from annotation `FunctionalInterface`. One reason for this limited recall might be that an user is only notified when the mention is in a comment or commit message. Looking at the projects themselves there is eclipse as the biggest one and sai as the smallest one. Also the commit sizes differ: SpongeAPI has rather large commits while OTBProject has smaller ones. Different to `FunctionalInterface` `RestController` is referenced by a lot more toy projects (for example Spring and firstRest) which yields more notifications. Personally, I perceived the notfications as pleasent because I get informed about new code related to topics I am interested in. I would not mind, if GitHub would add a topic based subscription feature.

