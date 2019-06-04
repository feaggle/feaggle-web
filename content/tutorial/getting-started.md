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

* Toggle configuration is done just once, and decoupled from the usage of the toggle.
* We promote a centralised configuration, outside production code. Changing the status of a toggle doesn't mean changing where we are using the toggle.
* We believe that releasing features can be done without deploying new code, just changing a toggle. That's why we support hot reloading in our 
[jdbc module](https://github.com/feaggle/feaggle-jdbc).
* Just plain Java, no annotations, no reflection. Can be used in any JVM language, compiles with Java 8, and runs in the GraalVM.

## Installing feaggle

feaggle is published regularly in jcenter, so it can be retrieved using gradle:

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

It would be even harder if we want to make a healthcheck to the new reports service to make it work. However, to reduce complexity, we can keep it like that.

At the end, when the new reporting service is ready and has all the features from all types of users, we can start rolling out to all customers:

```java
@Get("/report")
public Object getMyBillingReports(User user) {
  if (isNewReportFeatureEnabled && Math.random() < 0.5f) { // not evenly distributed, but aprox 50% of load.
    return newReports.getReport(user);
  } else {
    return reports.getReportForUser(user.getId());
  }
}
```

And finally, we can release to everyone:

```java
@Get("/report")
public Object getMyBillingReports(User user) {
  return newReports.getReport(user); // we might want in some situations to keep the toggle to disable it in case of a failure
}
```

## Configuring feaggle

Configuring feaggle depends on which module are you using. In this getting started guide, we are going to assume that you want to use only the core features
of feaggle. If you want to store your toggles in a SQL database you might want to take a look at [feaggle-jdbc](https://github.com/feaggle/feaggle-jdbc). However,
we recommend you to do the complete `Getting Started` guide and then move to the jdbc module.

To configure feaggle, we need what is called a `DriverLoader`. A `DriverLoader` allows feaggle to build the necessary infrastructure to keep track of any kind of toggle.
In feaggle, there are three types of toggles:

* **ReleaseToggle** allows you to enable or disable a feature completely.
* **ExperimentToggle** allows you to enable or disable a feature for a subset of the users. They require a *ExperimentCohort* which represents 
the properties of a subset of useres.
* **OperationalToggle** allows you to enable or disable a feature depending on the status of the application. For example, you might want to disable a feature if your
application is getting out of disk (or gracefully degrade to another service), if an external endpoint is down, or the cpu usage is dangerous.

To create a DriverLoader, you will need to use the `BasicDriverLoader` class, which exposes a builder to simplify the creation of any kind of toggle.

```java
BasicDriverLoader.builder()
    .releases(...)
    .experiments(...)
    .operationals(...)
    .build()
```

A BasicDriverLoader up to three drivers, one for each type of toggle. If you want to use only releases, you can just skip calling the `.experiments` and 
`.operational` methods.

Let's start with releases.

### Releases

Releases need a ReleaseDriver, that will configure toggles to use the appropiate information. Conveniently, feaggle comes with
a `BasicReleaseDriver` that allows to set up, in memory, all our releases, so they can be used later.

```java
BasicReleaseDriver.builder()
              .release(ENABLED_RELEASE, true)
              .release(DISABLED_RELEASE, false)
              .build()
```

For example, the configuration would look like:

```java
var releases = BasicReleaseDriver.builder()
  .release(ENABLED_RELEASE, true)
  .release(DISABLED_RELEASE, false)
  .build();

BasicDriverLoader.builder()
    .releases(releases)
    .build()
```

### Experiments

Experiments are a bit more complicated, because they need a cohort. A Cohort is just a subset of the users, defined by common properties. First, you will need to define
your Cohort (a class with all common properties to all users, where you will create segments) implementing the ExperimentCohort interface. For example, for our
reporting application, we would like to segment by paid users and by the country.

```java
public class Cohort implements ExperimentCohort {
  public final String userId;
  public final String countryCode;
  public final boolean isPremium;

  ...

  @Override
  public String identifier() {
    return userId;
  }
}
```

With the Cohort, we can start defining experiments with the BasicExperimentDriver:

```java
BasicExperimentDriver.<Cohort>builder()
    .experiment(
        Experiment.<Cohort>builder()
            .toggle("my-experiment") 
            .segment(cohort -> cohort.countryCode == "P") // this experiment is only enabled for users in Portugal
            .enabled(true)
            .build()
    )
    .experiment(
        Experiment.<Cohort>builder()
            .toggle("my-experiment-2") 
            .segment(cohort -> cohort.isPremium) // this experiment is enabled for premium users
            .segment(Rollout.<Cohort>builder().percentage(25).build()) // but only a random 25% percent of them
            .enabled(true)
            .build()
    )
    .build()
```

At the end, your configuration would look like:

```java
var experiments = BasicExperimentDriver.<Cohort>builder()
    .experiment(
        Experiment.<Cohort>builder()
            .toggle("my-experiment") 
            .segment(cohort -> cohort.countryCode == "P") // this experiment is only enabled for users in Portugal
            .enabled(true)
            .build()
    )
    .experiment(
        Experiment.<Cohort>builder()
            .toggle("my-experiment-2") 
            .segment(cohort -> cohort.isPremium) // this experiment is enabled for premium users
            .segment(Rollout.<Cohort>builder().percentage(25).build()) // but only a random 25% percent of them
            .enabled(true)
            .build()
    )
    .build()

BasicDriverLoader.builder()
    .experiments(experiments)
    .build()

```
### Operational Toggles

Operational toggles allows you to enable a feature based on the system state. Those kind of toggles are based on rules and sensors, that will gather information
of the system. For example, if you want to only enable a feature if the peer service is on, you would do:

```java
OperationalDriver.builder()
    .rule(
        Rule.builder()
            .toggle(TOGGLE_NAME)
            .enabled(true)
            .sensor(Healthcheck.builder()
                    .check(this::reportingServiceAvailability) // method that checks the remote server is available
                    .interval(1000) // interval in milliseconds between calls
                    .healthyCount(3) // nr of times the healthcheck should be working for being considered healthy
                    .unhealthyCount(2) // nr of times the healthcheck should fail for being considered unhealthy
                    .build())
            .build()
    ).build()
```

There are several sensors already bundled with feaggle. Some code examples:

#### Disk Usage

Only enable the feature if the current disk partition has less than 20TB.

```java
Rule.builder()
  .toggle(TOGGLE_NAME)
  .sensor(Disk.builder().fileStore(Disk.fileStoreOf(Paths.get("."))).predicate(Disk.spaceAvailableIsLessThan(20, Unit.TB)).build())
  .enabled(true)
  .build()
```

#### Memory

Only enable the feature if the memory usage is more than 1% of the available memory.

```java
Rule.builder()
  .toggle(TOGGLE_NAME)
  .sensor(Memory.builder().predicate(Memory.usageIsGreaterThan(1)).build())
  .enabled(true)
  .build()
```

#### CPU

Only enable the feature if the CPU usage is greater than 1% of the CPU.

```java
Rule.builder()
  .toggle(TOGGLE_NAME)
  .sensor(Cpu.builder().predicate(Cpu.usageIsGreaterThan(1)).build())
  .enabled(true)
  .build()
```

## Building a feaggle instance

Feaggle has a facade that will set up all the drivers and toggles. This way, for validating that a toggle is enabled, you only need to use a single object
that is consistent.

To create a Feaggle instance we need to call the `.load` static method with the created DriverLoader:

```java
var driverLoader = BasicDriverLoader.builder()
    .releases(...)
    .experiments(...)
    .operationals(...)
    .build();

Feaggle<Cohort> feaggle = Feaggle.load(driverLoader);
```

For checking the status of the toggles there are three methods, one per toggle type. The API is quite simple, so probably is clear with a code example:

```java
// release toggles
feaggle.release("my-new-cool-release").isEnabled();

// experiment toggles
var cohort = new Cohort("user-id", "P", true);
feaggle.experiment("my-experiment").isEnabledFor(cohort);

// operational toggles
feaggle.operational("my-operational-toggle").isEnabled();
```

For more advanced information and configuration, please check:

* [The test suite in feaggle](https://github.com/feaggle/feaggle/tree/master/src/test/java/io/feaggle/specs).
* [The test suite in feaggle-jdbc](https://github.com/feaggle/feaggle-jdbc/tree/master/src/test/java/io/feaggle/jdbc).
* [Other tutorials](/tutorial/)
