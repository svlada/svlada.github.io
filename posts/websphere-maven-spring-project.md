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

### Introduction

This article explains how to create a Spring web application for deployment on IBM WebSphere Application Server. The objective is to set up a Maven project that produces an EAR package (with a WAR module and application assembly) suitable for WebSphere.

### Maven project structure

The following list shows the directory structure of the multi-module Maven project:

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

The project consists of one parent (aggregator) module and two child (sub) modules.

For more details on working with multi-module builds, see the official [Maven documentation](http://maven.apache.org/pom.html#Aggregation).

**Aggregator module**

The aggregator is the top-level module that brings multiple modules together into a single build.

The following is an excerpt from the pom.xml of the aggregator module:

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

Both sub-modules (app-ear and app-webapp) must reference the parent module in their pom.xml files, as shown below:

```java
<parent>
	<artifactId>app</artifactId>
	<groupId>com.svlada</groupId>
	<version>1.0.0</version>
</parent>
```

**Sub-module: app-ear** 

The WAR module (app-webapp) must be included in the list of EAR dependencies.

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

In the app-ear/pom.xml, include the maven-ear-plugin in the `<build><plugins>` section:

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

The web app module is a simple web application generated with Spring Initializr.

### Source code 

You can clone the entire project from the following [GitHub repository](https://github.com/svlada/websphere-maven-spring-project).

