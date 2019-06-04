+++
# Custom widget.
# An example of using the custom widget to create your own homepage section.
# To create more sections, duplicate this file and edit the values below as desired.
widget = "custom"
active = true
date = 2016-04-20T00:00:00

# Note: a full width section format can be enabled by commenting out the `title` and `subtitle` with a `#`.
title = "Example"
subtitle = ""

# Order that this section will appear in.
weight = 40

+++

Feaggle is just a library, lightweight, and simple.

```java
var feaggle = Feaggle.load(
    BasicDriverLoader.builder()
            .releases(BasicReleaseDriver.builder()
                     .release("my-release!", true)
                     .build()
    )
);

feaggle.release("my-release!").isEnabled();
```

If you want to learn more about feaggle, feel free to take a look at the [Getting Started](./tutorial/getting-started/) guide.
