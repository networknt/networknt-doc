# Memory Assault

The `memory-assault` module in `light-chaos-monkey` allows you to inject memory pressure into your application. When triggered, it allocates memory in increments until a target threshold is reached, holds that memory for a specified duration, and then releases it. This is useful for testing how your application behaves under low-memory conditions and identifying potential memory-related issues.

## Core Components

### MemoryAssaultHandler
The middleware handler responsible for consuming memory.
*   **Behavior**: When enabled and triggered (based on the `level` configuration), it starts an asynchronous process to "eat" free memory. It incrementally allocates byte arrays until the total memory usage reaches the `memoryFillTargetFraction` of the maximum heap size.
*   **Safety**: The handler includes logic to prevent immediate OutOfMemoryErrors by controlling the increment size and respecting the maximum target fraction. It also performs a `System.gc()` after releasing the allocated memory to encourage the JVM to reclaim the space.

### MemoryAssaultConfig
Configuration class that loads settings from `memory-assault.yml`.

## Configuration

The module is configured via `memory-assault.yml`.

**Key Settings:**
*   `enabled`: Enable or disable the handler (default: `false`).
*   `bypass`: If `true`, the assault is skipped even if enabled (default: `true`).
*   `level`: The probability of an attack, defined as "1 out of N requests".
    *   `level: 10` means approximately 1 in 10 requests (10%) will trigger the memory assault.
    *   `level: 1` means every request can trigger it (though typically it runs until finished once started).
*   `memoryMillisecondsHoldFilledMemory`: Duration (in ms) to hold the allocated memory once the target fraction is reached (default: `90000`).
*   `memoryMillisecondsWaitNextIncrease`: Time (in ms) between allocation increments (default: `1000`).
*   `memoryFillIncrementFraction`: Fraction of available free memory to allocate in each increment (default: `0.15`).
*   `memoryFillTargetFraction`: The target fraction of maximum heap memory to occupy (default: `0.25`).

**Example Configuration:**

```yaml
enabled: true
bypass: false
level: 10
memoryFillTargetFraction: 0.5
memoryMillisecondsHoldFilledMemory: 60000
```
