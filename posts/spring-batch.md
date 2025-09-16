---
title: Spring batch job repository configuration for WebSphere and Oracle
description: This article describes problem with using default Spring batch Job Repository configuration in an application deployed on the WebSphere.
date: 2015-12-02
tags:
  - java
  - spring-batch
layout: layouts/post.njk
permalink: "spring-batch-job-repository-configuration-for-websphere-and-oracle/index.html"
---

### Introduction

This article describes an issue with the default Spring Batch Job Repository configuration when deployed on a WebSphere application server backed by an Oracle database.

The following exception may occur:

```java
org.springframework.dao.DataAccessResourceFailureException: Could not create Oracle LOB
Caused by: org.springframework.dao.InvalidDataAccessApiUsageException: Couldn't initialize OracleLobHandler because Oracle driver classes are not available. Note that OracleLobHandler requires Oracle JDBC driver 9i or higher!; nested exception is java.lang.ClassNotFoundException: oracle.sql.BLOB
Caused by: java.lang.ClassNotFoundException: oracle.sql.BLOB
```

This issue is documented on the [IBM support website](http://www-01.ibm.com/support/docview.wss?uid=swg21633083).

### Configuration

The proper solution is to configure the LOB handler dependency of the [Job Repository](http://docs.spring.io/spring-batch/apidocs/org/springframework/batch/core/repository/JobRepository.html) and inject an [OracleLobHandler](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/jdbc/support/lob/OracleLobHandler.html) bean.

For WebSphere support, the `lobHandler` bean must be instantiated with a compatible `nativeJdbcExtractor` provided by WebSphere.

The following snippet shows the Job Repository configuration details:

```java
<batch:job-repository id="jobRepository" 
    data-source="dataSource" 
    transaction-manager="transactionManager" 
    lob-handler="lobHandler" />

<bean id="lobHandler" 
    class="org.springframework.jdbc.support.lob.OracleLobHandler">
    <property name="nativeJdbcExtractor" ref="nativeJdbcExtractor" />
</bean>

<bean id="nativeJdbcExtractor"
    class="org.springframework.jdbc.support.nativejdbc.WebSphereNativeJdbcExtractor" />
```

**Data source configuration**
```java
<bean id="dataSource" class="org.springframework.jndi.JndiObjectFactoryBean">
    <property name="jndiName" value="java:comp/env/jdbc/DS_NAME" />
    <property name="lookupOnStartup" value="true" />
    <property name="cache" value="true" />
    <property name="proxyInterface" value="javax.sql.DataSource" />
</bean>
```

**Transaction manager configuration**
```java
<bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager" primary="true">
    <property name="entityManagerFactory" ref="entityManagerFactory"/>
</bean>
```

