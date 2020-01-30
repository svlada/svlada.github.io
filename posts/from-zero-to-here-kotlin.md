---
title: draft
description: From zero to hero - How to build lightweight API infrastructure
date: 2020-02-02
tags:
  - java
  - spring-boot
  - jwt
  - spring-security
layout: layouts/post.njk
permalink: "how-to-build-lightweight-api-with-kotlin-and-springboot/index.html"
---

## Introduction

**Target audience** _Junior developers that are fresh out of college. I hope that these series of posts will help them along the way_. 

This article is an opinionated guide for building lightweight APIs using Spring Boot and Kotlin. This is the first part of the series. Upcoming posts will include implementation details and working samples on Github. 

The API development on JVM is marketed as a heavyweight, dreadful process. Often, development using Java is compared to operating an armored fighting vehicle. In some cases, this is a correct statement. Big enterprises are using deprecated libraries, heavy-weight servers, and frameworks (e.g. IBM Websphere or IBM Portal). The decision to go with an enterprisey stack is job security. Hey, nobody got fired for choosing IBM.

The reality is different. Many lightweight servers and frameworks are developed on top of JVM stack, enabling a team to develop busines features at a high pace. Unmatched security and performance benefits given by state-of-the-art JVM platform are hard to find in other ecosystems (e.g. Rails or Django).

The following components are basic building blocks for any API:

1. Web framework
2. Centralized Logging
3. Performance metrics and monitoring
4. Error handling
5. Resource design
6. Documentation
7. CI/CD
8. Code versioning
9. Semantic versioning
10. Persistence layer

The developer should try to keep code clean and simple by following by introducing:

1. SOLID principles
2. Clean architecture and modular application design
3. Exception handling

The following concepts are equaly important if you are into building microservices:

1. API Gateway
2. Service registration and discovery
3. Dynamic routing and load balancing
4. Circuit Breaker: Fault tolerance and resilience
5. Distributed tracing
6. Dynamic Configuration
7. Inter-process communication
8. Security
9. Distributed Transactions
10. Centralized logging
11. Centralized metrics & monitoring

## Basic building blocks

This section presents high-level overview of critical API components. 

### Web frameworks

This sections shows only frameworks that are stable and ready for production use.

The following is the list of battle tested JVM web frameworks. These ones can be used for critical production apps:

1. **Spring boot** - The most robust JVM framework available to date.
2. **Dropwizard** - Excellent lightweight stack for skilled teams. You'll need to implement a decend amount of "glue" code.
3. **Micronaut** - Good for serverless applications.
4. **Vert.x** - Event-driven framework with similar concepts found in NodeJS.
  
For pet projects, you can test out a couple of pure kotlin that are yet to be proven in wild:

1. http4k
2. ktor

The focus here is on Spring Boot as it's inevitable framework that you will encounter in the wild.

### Code versioning



### Resource design

### Error handling

### Centralized Logging

### Performance metrics and monitoring

### Persistence layer

### Documentation

### CI/CD

### Semantic versioning

## Code guidelines