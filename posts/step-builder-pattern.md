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

1. <a title="Introduction: Step builder pattern" href="#introduction">Introduction</a>
2. <a title="Pros and cons about step builder pattern" href="#step-builder-pattern-pros-and-cons">Pros and cons</a>
3. <a title="Code walkthrough" href="#code-walkthrough">Code-walkthrough</a>
4. <a title="Source code" href="#source-code">Source code</a>
5. <a title="Eclipse plug-in" href="#eclipse-plugin">Eclipse plug-in</a>

## <a name="introduction" id="introduction">Introduction</a>
I've recently decided to use the Amazon SES API to send emails to my [Microservices Weekly](http://microservicesweekly.com) subscribers. What prompted me to use Amazon SES API is its price. It's cheap. However, the client Java API provided by Amazon is not so simple to interact with, so I decided to create a [small wrapper](https://github.com/svlada/ziggy) around their API.

To make a long story short, the main purpose here is to share my experience with a less well-known derivate of the Builder pattern - Step Builder pattern.

The Step Builder pattern is an object creation software design pattern. It's not so commonly mentioned in popular readings about design patterns.

The Step Builder pattern offers some neat benefits when you compare it to a traditional builder pattern. One of the main Step Builder pattern benefits is providing the client with the guidelines on how your API should be used. It can be seen as a mixture of a builder pattern and a state machine and in fact, this pattern is often referred to as a wizard for building objects. 

## <a name="step-builder-pattern-pros-and-cons" id="step-builder-pattern-pros-and-cons">Pros and cons</a>

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

## <a name="code-walkthrough" id="code-walkthrough">Code-walkthrough</a>
Since the Step Builder pattern is a creational design pattern, let's focus on its purpose - the creation of objects.

Example of API usage is shown below:

```java
Email email = Email.builder().from(EmailAddress.of("Microservices Weekly <mw@microservicesweekly.com>"))
	.to(EmailAddress.of("svlada@gmail.com"))
	.subject(Subject.of("Subject"))
	.content(Content.of("Test email"))
	.build();
```

Now, let's see how API is enforcing the initialization of an object in the pre-defined order.

The following image is the graphical representation of the state machine for constructing Email object with the Step Builder pattern (mandatory values are marked purple and optional values are yellow):
![Builder pattern](/img/step-builder/step-builder.png)

Rules of thumb for implementation:

1. Add dependencies to your class. It's recommended to add a private modifier to class attributes.
2. Define creational steps as inner interfaces in your base class. 
3. Each creational step should return the next step (interface) in the chain.
4. The final step should be an interface called "Build" which will provide build() method.
5. Define one inner static Builder class that implements all of the defined steps.
6. Implement step interface methods.

## <a name="source-code" id="source-code">Source code</a>
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

## <a name="eclipse-plugin" id="eclipse-plugin">Eclipse plug-in</a>

So far, I haven't found a plug-in for Eclipse that provides Step Builder code generation feature.

I've created a GitHub repository to create an Eclipse plug-in that will provide support for Step Builder pattern generation: https://github.com/svlada/alyx
