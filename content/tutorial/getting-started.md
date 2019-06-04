+++
title = "Getting Started"
date = 2018-11-30T00:00:00

[header]
image = ""
caption = "Getting Started"
+++

feaggle is a library that simplifies a process: continuous delivery. The main idea of feaggle is to wrap new features that are under development, so they can be
enabled or disabled to a subset of your customers. This allows teams to get quality feedback faster and easens the burden of managing feature toggles.

We designed feaggle with simplicity and security in mind, that's why:

* Toggle configuration is done just once, and decoupled to the usage of the toggle.
* We promote a centralised configuration, outside production code. Changing the status of a toggle doesn't mean changing where we are using the toggle.
* We believe that releasing features can be done without deploying new code, just changing a toggle. That's why we support hot reloading in our 
[jdbc module](https://github.com/feaggle/feaggle-jdbc).
* Just plain Java, no annotations, no reflection. Can be used in any JVM language, compiles with Java 8, and runs in the GraalVM.

## Installing feaggle

feaggle is published regularly in jcenter, so it can be retrieved using gradle easily:

```groovy
repositories {
  // ...
  jcenter()
  // ...
}

depedencies {
  compile 'io.feaggle:feaggle:2.0.0'
}
```

Right now the latest version is: 
[![Download](https://api.bintray.com/packages/kmruiz/feaggle/feaggle/images/download.svg)](https://bintray.com/kmruiz/feaggle/feaggle/_latestVersion)

## How to use feature toggles

If you already have experience with feature toggles and you want to start right with the code, please go to the [configuring feaggle](#configuring-feaggle) section. If 
you want to refresh some knowledge or you are just starting from scratch on using feature toggles, welcome! We will go through the process of using toggles in our
production code and how to release incrementally a feature. We will be also exploring different types of toggles so we know which one to use in which circumstances.

Usually you want to release new features incrementally because allows you to:

* Verify that the feature works for a subset of your customers.
* Verify that the feature brings business value without implementing the whole epic.
* Avoid integration issues with other components, as the code is pushed as soon as possible to production.

Let's say that we want to implement a new version of a feature: a new reporting integration that will allow us to get more information about the billing on our platform.
Right now our code looks like:

```java
@Get("/report")
public Object getMyBillingReports(User user) {
  return reports.getReportForUser(user.getId());
}
```

Sadly, the new reporting service is still under development and lacks some important features. After some investigation we've found that probably we can release
the integration that we have for just a subset of our users: all users that are not premium. It looks like those features are only used for premium users, so we can
release earlier!

We might want to do something like this:

```java
@Get("/report")
public Object getMyBillingReports(User user) {
  if (user.isNotPremium()) {
    return newReports.getReport(user);
  } else {
    return reports.getReportForUser(user.getId());
  }
}
```

However we've found that the new service is not handling all the load and we overloaded it. It's reasonable to do two things:

* Find a way to easy disable the toggle for everyone if we have problems with the new reporting service.
* Rollout the feature to a subset of the non-premium users and be able to increase the number of users with access.

The easiest option right now would be:

```java
@Get("/report")
public Object getMyBillingReports(User user) {
  if (isNewReportFeatureEnabled && user.isNotPremium() && Math.random() < 0.2f) { // not evenly distributed, but aprox 20% of load.
    return newReports.getReport(user);
  } else {
    return reports.getReportForUser(user.getId());
  }
}
```

## Configuring feaggle

Configuring feaggle depends on which module are you using. In this getting started guide, we are going to assume that you want to use only the core features
of feaggle. If you want to store your toggles in a SQL database
