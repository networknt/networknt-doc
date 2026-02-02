# Switch

The `Switch` module (source package `com.networknt.switcher`) provides a mechanism to manage runtime feature flags or service availability switches. It allows parts of the system to turn logic on or off dynamically without restarting the server.

A common use case is during the server shutdown process: a "service available" switch can be turned off to stop accepting new requests while allowing existing requests to complete.

## Components

-   **Switcher**: A simple entity class representing a named switch with a boolean state (`on` or `off`).
-   **SwitcherService**: An interface defining operations to manage switchers (init, get, set, listener).
-   **LocalSwitcherService**: The default in-memory implementation of `SwitcherService`.
-   **SwitcherUtil**: A static utility class that provides global access to the `SwitcherService`.

## Usage

The module is primarily used via the `SwitcherUtil` class.

### Initializing a Switcher

You can initialize a switcher with a name and a default state.

```java
import com.networknt.switcher.SwitcherUtil;

// Initialize a switcher named "maintenanceMode" to false (off)
SwitcherUtil.initSwitcher("maintenanceMode", false);
```

### Checking Switch State

Check if a switch is currently enabling.

```java
if (SwitcherUtil.isOpen("maintenanceMode")) {
    // Logic when maintenance mode is ON
} else {
    // Normal operation logic
}
```

You can also check with a default value if the switcher might not have been initialized yet.

```java
// Returns true if "newFeature" is open, otherwise returns the default (false)
boolean isFeatureEnabled = SwitcherUtil.switcherIsOpenWithDefault("newFeature", false);
```

### Setting Switch State

Change the state of a switch programmatically.

```java
// Turn on maintenance mode
SwitcherUtil.setSwitcherValue("maintenanceMode", true);
```

### Listeners

You can register a `SwitcherListener` to be notified when a switcher's value changes.

```java
SwitcherUtil.registerSwitcherListener("maintenanceMode", new SwitcherListener() {
    @Override
    public void onSwitcherChanged(String key, boolean value) {
        System.out.println("Switcher " + key + " changed to " + value);
    }
});
```

## Example: Graceful Shutdown

In the `light-4j` server shutdown hook, the framework might use a switcher to signal that the server is shutting down.

1.  Server startup: `SwitcherUtil.initSwitcher(Constants.SERVER_STATUS_SWITCHER, true);`
2.  Request Handler: Checks `SwitcherUtil.isOpen(Constants.SERVER_STATUS_SWITCHER)`. If false, returns 503 Service Unavailable.
3.  Shutdown Hook: `SwitcherUtil.setSwitcherValue(Constants.SERVER_STATUS_SWITCHER, false);`

This allows the registry to be notified and the load balancer to stop sending traffic, while the server wraps up in-flight requests.
