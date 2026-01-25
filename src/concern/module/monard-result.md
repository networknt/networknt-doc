# Monad Result

The `monad-result` module provides a functional way to handle success and failure occurrences in your application logic. It is an implementation of a Result monad, similar to `Optional` in Java, but designed to carry failure information (error status) instead of just being empty.

This approach promotes cleaner code by avoiding the use of exceptions for expected business logic failures, making the control flow more explicit and easier to follow.

## Core Components

The module consists of three main parts:

### 1. Result<T>
An interface that represents the result of an operation. It can be either a `Success` or a `Failure`.

### 2. Success<T>
An implementation of `Result` that holds the successful value.

### 3. Failure<T>
An implementation of `Result` that holds a `com.networknt.status.Status` object describing the error.

## Usage

### Creating Results

You can create success or failure results using the static factory methods:

```java
import com.networknt.monad.Result;
import com.networknt.monad.Success;
import com.networknt.monad.Failure;
import com.networknt.status.Status;

// Create a success result
Result<String> success = Success.of("Hello World");

// Create a failure result with a Status object
Status status = new Status("ERR10001");
Result<String> failure = Failure.of(status);
```

### Transforming Results

The `Result` interface provides several monadic methods to work with the values without explicitly checking for success or failure:

#### Map
Applies a function to the value if the result is a success. If it's a failure, it returns the failure as-is.

```java
Result<Integer> length = success.map(String::length);
```

#### FlatMap
Similar to `map`, but the mapping function returns another `Result`. This is useful for chaining multiple operations that can fail.

```java
Result<User> user = idResult.flatMap(id -> userService.getUser(id));
```

#### Fold
Reduces the `Result` to a single value by providing functions for both success and failure cases. This is often used at the end of a chain to convert the result into a response body or an external format.

```java
String message = result.fold(
    val -> "Success: " + val,
    fail -> "Error: " + fail.getError().getMessage()
);
```

#### Lift
Allows combining two `Result` instances by applying a function to their internal values. If either result is a failure, it returns a failure.

```java
Result<String> result1 = Success.of("Hello");
Result<String> result2 = Success.of("World");
Result<String> combined = result1.lift(result2, (r1, r2) -> r1 + " " + r2);
```

### Conditional Execution

You can perform actions based on the state of the result using `ifSuccess` and `ifFailure`:

```java
result.ifSuccess(val -> System.out.println("Got value: " + val))
      .ifFailure(fail -> System.err.println("Error Status: " + fail.getError()));
```

## Why use Monad Result?

1. **Explicit Error Handling**: Failures are part of the method signature and cannot be accidentally ignored like unchecked exceptions.
2. **Improved Readability**: Functional chaining (map, flatMap) leads to cleaner logic compared to deeply nested if-else blocks.
3. **Composability**: It is easy to combine multiple operations into a single result using `flatMap` and `lift`.

## Dependency

Add the following to your `pom.xml`:

```xml
<dependency>
    <groupId>com.networknt</groupId>
    <artifactId>monad-result</artifactId>
    <version>${version.light-4j}</version>
</dependency>
```
