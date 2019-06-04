+++
title = "Welcome, Epics!"
date = 2019-03-06T00:00:00
draft = false

[header]
image = ""
caption = "Welcome, Epics!"
+++

We've been working for some time doing Continuous Delivery in our teams, and we found that managing feature toggles in our code
was a challenge. Not only because toggles are complex to implement properly and each of us implemented their toggles in different ways,
but also because toggles evolved rapidly when we were releasing new features.

A good example of the evolution of a toggle would be to, first, enable our new shiny feature for a subset of users. When the feature has been probed with
a subset of the customers, we can enable it for a bigger set of customers (probably multiple countries). However, we want to make sure that the new feature
doesn't break the system that is already in production, so we would want to disable it in case the new service that we depend on is down. After testing the
new feature under a big load, we might want to progressively enable it to all customers, but with the ability to disable it quickly to all of our customers
if we find something problematic (like a security bug). At the end, we want to just enable the feature for all customers and clean up the feature toggle.

![Live of a feature](/img/blog/welcome-epics/feature-flow.png)

However, managing this flow manually is really complex, is error prone and messes up with the code. For example, we would have with something like:

```java
var myRelease = feaggle.release("MY_RELEASE");
var myExperiment = feaggle.experiment("MY_RELEASE");

if (myRelease.isEnabled() && myExperiment.isEnabledFor(myCurrentUser)) {
  newShinyFeature();
}
```

That later would evolve to:


```java
var myRelease = feaggle.release("MY_RELEASE");
var myExperiment = feaggle.experiment("MY_EXPERIMENT");
var reportsService = feaggle.operational("REPORTS_SERVICE_IS_UP");

if (myRelease.isEnabled() && myExperiment.isEnabledFor(myCurrentUser) && reportsService.isEnabled()) {
  newShinyFeature();
}
```

We found that every change to our rollout strategy would mean that we changed the code that was in production, meaning that we could break it (*spoiler*, we broke it). 

We though that having some kind of abstraction over a feature, that could evolve during the lifetime of the feature, was necessary. That's why we came up with the
concept of *drum-roll* **Epic**.

So, what is an epic itself? An epic aggregates a set of toggles (of different types) and is enabled only if **all** toggles inside the epic are enabled for a cohort.
Building an epic is quite easy:

```java
Epic<MyCohort> epic = feaggle.epic()
  .release(feaggle.release("MY_RELEASE")) // you can add more than one release toggle, just calling .release again
  .experiment(feaggle.experiment("MY_EXPERIMENT")) // also, you can add more than one experiment with .experiment
  .operational(feaggle.operational("REPORTS_SERVICE_IS_UP")) // also here :D
  .build();

// usage:

if (epic.isEnabledFor(myCurrentUser)) {
  newShinyFeature();
}
```

There are several awesome features from *epics* that will let you fall in love with them:

* Epics are lazy. They will evaluate toggles by complexity until finds one that is disabled. For example, if a release is turned off, 
the epic won't call any experiment or operational toggle.
* Epics are configured once and can be injected, if you want to change how they work, just reconfigure them.
* Epics are unaware of how other toggles work. You can use epics with toggles that are stored in your database with 
[feaggle-jdbc](https://github.com/feaggle/feaggle-jdbc) or toggles in memory. Your epic won't change.

We believe that epics will be a game changer on how we do Continuous Deliver, as we have more power on how we release our features
without sacrificing ease. You can start using epics in feaggle 2.0.0: despite the major version, it's backwards compatible to feaggle 1.x
and feaggle-jdbc!

Read more information on how to use feaggle in the [the feaggle repository](https://github.com/feaggle/feaggle). You can also check the 
[Getting Started](https://www.feaggle.org/tutorial/getting-started/) guide for more information about how to use feaggle in your project.

And remember that if you have any feedback or suggestion,please fill us an issue so we can keep track of them 😍.

