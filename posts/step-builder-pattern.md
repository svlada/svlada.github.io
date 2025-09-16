---
title: Step builder pattern
description: The Step Builder pattern is an object creation software design pattern.
date: 2016-06-09
tags:
  - java
  - design-patterns
layout: layouts/post.njk
permalink: "step-builder-pattern/index.html"
---

## Introduction
I recently decided to use the Amazon SES API to send emails to my [Microservices Weekly](http://microservicesweekly.com) subscribers. The main reason? Price. It's cheap. However, the Java client API Amazon provides isn't exactly pleasant to work with, so I built a [small wrapper](https://github.com/svlada/ziggy) around it.

To make a long story short, the real purpose here is to share my experience with a less well-known variation of the Builder pattern: the Step Builder pattern.

The Step Builder pattern is an object-creation design pattern that doesn't get much attention in popular design pattern discussions. Compared to the traditional builder, it offers some neat advantages. Chief among them is that it guides the client in using your API correctly. In practice, it feels like a mix of the builder pattern and a state machine—and it's often described as a wizard for building objects. 

## Pros and cons

**Pros**

1. User guidance for your API through the object creation process step by step.
2. API User can call the builder's build() method once the object is in a consistent state.
3. Reduced opportunity for the creation of inconsistent object instances.
4. Sequencing initialization of mandatory fields.
5. Fluent API.
6. No need for providing validate() method for field validation.

**Cons**

1. Low readability of code needed to implement the pattern itself.
2. No eclipse plugin to help with code generation. (On the other hand, there are plenty of code generators for Builder pattern generator).

## Code walkthrough
Since the Step Builder pattern is a creational design pattern, its main purpose is the creation of objects.

Here's a simple example of how the API can be used:

```java
Email email = Email.builder().from(EmailAddress.of("Microservices Weekly <mw@microservicesweekly.com>"))
	.to(EmailAddress.of("svlada@gmail.com"))
	.subject(Subject.of("Subject"))
	.content(Content.of("Test email"))
	.build();
```

Now, let's look at how the API enforces object initialization in a predefined order.

The diagram below shows a state machine for constructing an Email object with the Step Builder pattern. Mandatory values are marked in purple, while optional values are shown in yellow:
![Builder pattern](/img/step-builder/step-builder.png)

Rules of thumb for implementation:

1. Add dependencies to your class. It's recommended to add a private modifier to class attributes.
2. Define creational steps as inner interfaces in your base class. 
3. Each creational step should return the next step (interface) in the chain.
4. The final step should be an interface called "Build" which will provide build() method.
5. Define one inner static Builder class that implements all of the defined steps.
6. Implement step interface methods.

## Source code
Complete Example of Step by Step builder pattern:

```java
public class Email {
	private EmailAddress from;
	private List<EmailAddress> to;
	private List<EmailAddress> cc;
	private List<EmailAddress> bcc;
	private Subject subject;
	private Content content;
	
	public static FromStep builder() {
		return new Builder();
	}

	public interface FromStep {
		ToStep from(EmailAddress from);
	}
	
	public interface ToStep {
		SubjectStep to(EmailAddress... from);
	}
	
	public interface SubjectStep {
		ContentStep subject(Subject subject);
	}
	
	public interface ContentStep {
		Build content(Content content);
	}
	
	public interface Build {
		Email build();
		Build cc(EmailAddress... cc);
		Build bcc(EmailAddress... bcc);
	}
	
	public static class Builder implements FromStep, ToStep, SubjectStep, ContentStep, Build {
		private EmailAddress from;
		private List<EmailAddress> to;
		private List<EmailAddress> cc;
		private List<EmailAddress> bcc;
		private Subject subject;
		private Content content;
		
		@Override
		public Email build() {
			return new Email(this);
		}
		@Override
		public Build cc(EmailAddress... cc) {
			Objects.requireNonNull(cc);
			this.cc = new ArrayList<EmailAddress>(Arrays.asList(cc));
			return this;
		}
		@Override
		public Build bcc(EmailAddress... bcc) {
			Objects.requireNonNull(bcc);
			this.bcc = new ArrayList<EmailAddress>(Arrays.asList(bcc));
			return this;
		}
		@Override
		public Build content(Content content) {
			Objects.requireNonNull(content);
			this.content = content;
			return this;
		}
		@Override
		public ContentStep subject(Subject subject) {
			Objects.requireNonNull(subject);
			this.subject = subject;
			return this;
		}
		@Override
		public SubjectStep to(EmailAddress... to) {
			Objects.requireNonNull(to);
			this.to = new ArrayList<EmailAddress>(Arrays.asList(to));
			return this;
		}
		@Override
		public ToStep from(EmailAddress from) {
			Objects.requireNonNull(from);
			this.from = from;
			return this;
		}
	}

	private Email(Builder builder) {
		this.from = builder.from;
		this.to = builder.to;
		this.cc = builder.cc;
		this.bcc = builder.bcc;
		this.subject = builder.subject;
		this.content = builder.content;
	}

	public EmailAddress getFrom() {
		return from;
	}

	public List<EmailAddress> getTo() {
		return to;
	}

	public List<EmailAddress> getCc() {
		return cc;
	}

	public List<EmailAddress> getBcc() {
		return bcc;
	}

	public Subject getSubject() {
		return subject;
	}

	public Content getContent() {
		return content;
	}
}
```

## Eclipse plug-in

So far, I haven't found an Eclipse plug-in that supports Step Builder code generation.

To fill that gap, I've started a GitHub repository for an Eclipse plug-in that provides Step Builder pattern generation:
👉 [svlada/alyx](https://github.com/svlada/alyx)
