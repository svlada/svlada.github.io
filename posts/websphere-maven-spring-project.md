---
title: Multi-module Websphere web application
description: This article describes how to create Spring web application that can be deployed on WebSphere application server. The goal is to to create maven project that can be used to generate EAR file.
date: 2015-09-10
tags:
  - java
  - websphere
  - spring
layout: layouts/post.njk
permalink: "websphere-maven-spring-project/index.html"
---

## Table of contents:
1. <a href="#introduction">Introduction</a>
2. <a href="#maven-project-structure">Maven project structure</a>
4. <a href="#source-code">Source code</a>
 
### <a name="introduction" id="introduction">Introduction</a>

This article describes how to create a Spring web application that can be deployed on the WebSphere application server. The goal is to create a maven project that can be used to generate an EAR file.

### <a name="maven-project-structure" id="maven-project-structure">Maven project structure</a>

The following list shows the multi-module maven project directory structure:

```text
|- websphere-maven-spring-project 
|-- app-ear/
|---- pom.xml
|-- app-webapp/
|---- src/
|---- .classpath
|---- pom.xml
|- pom.xml
```

The project consists of one parent/aggregator module and two sub/child modules. For more information on multi-module please go to [maven documentation website](http://maven.apache.org/pom.html#Aggregation).

**Aggregator module**

The aggregator is a top-level module used to join multiple modules.

The following is an excerpt from aggregator `pom.xml`:

```java
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.svlada</groupId>
  <artifactId>app</artifactId>
  <name>app</name>
  <version>1.0.0</version>
  <packaging>pom</packaging>
  <url>http://maven.apache.org</url>
  <modules>
    <module>app-ear</module>
    <module>app-webapp</module>
  </modules>
```

**Sub-modules**

Both sub-modules (`app-ear` and `app-webapp`) must include reference to the parent module as follows:

```java
<parent>
	<artifactId>app</artifactId>
	<groupId>com.svlada</groupId>
	<version>1.0.0</version>
</parent>
```

**Sub-module: app-ear** 

WAR module (`app-webapp`) needs to be included in the list of EAR dependencies.

```java
<dependencies>
	<dependency>
		<groupId>com.svlada</groupId>
		<artifactId>app-webapp</artifactId>
		<version>1.0.0</version>
		<type>war</type>
	</dependency>
</dependencies>
```

Include ```maven-ear-plugin``` in the build plugins section of the app-ear pom.xml:

```java
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-ear-plugin</artifactId>
	<version>2.10</version>
	<configuration>
		<finalName>APP_EAR</finalName>
		<modules>
			<webModule>
				<groupId>com.svlada</groupId>
				<artifactId>app-webapp</artifactId>
				<bundleFileName>app-webapp.war</bundleFileName>
				<contextRoot>/app</contextRoot>
			</webModule>
		</modules>
		<generateApplicationXml>true</generateApplicationXml>
	</configuration>

</plugin>
```

**Sub-module: app-webapp**

Web app module is an simple web application generated with the spring initializr.

### <a name="source-code" id="source-code">Source code</a> 

You can clone entire project from the following [github repository](https://github.com/svlada/websphere-maven-spring-project).

