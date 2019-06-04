+++
# Custom widget.
# An example of using the custom widget to create your own homepage section.
# To create more sections, duplicate this file and edit the values below as desired.
widget = "custom"
active = true
date = 2016-04-20T00:00:00

# Note: a full width section format can be enabled by commenting out the `title` and `subtitle` with a `#`.
title = "Evolves with you"
subtitle = ""

# Order that this section will appear in.
weight = 42

+++

[feaggle-jdbc](https://github.com/feaggle/feaggle-jdbc) allows you to have toggles that can be hot-reloaded, just by changing the configuration.

```java
Feaggle feaggle = Feaggle.load(
    JdbcDriver.from(yourJdbcConnection)
        .defaults()
        .build()
);

feaggle.release("my-release!").isEnabled();
```

If you want to learn more about feaggle-jdbc, feel free to take a look at the [project documentation](https://github.com/feaggle/feaggle-jdbc).
