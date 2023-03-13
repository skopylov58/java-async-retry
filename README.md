![example workflow](https://github.com/skopylov58/java-async-retry/actions/workflows/gradle.yml/badge.svg)
[![Javadoc](https://img.shields.io/badge/JavaDoc-Online-green)](https://skopylov58.github.io/java-async-retry/)

## `Retry<T>` - non-blocking asynchronous functional retry procedure

Retry is ancient strategy of failure recovering. We using Retry a lot when connecting to databases, doing HTTP requests, sending e-mails, etc., etc.
Naive retry implementations typically do retries in the loop and sleeping some time when exceptions occur. The main disadvantage ot this approach is that `Thread.sleep(...)` is blocking synchronous operation that freezes working thread. Even you move such retry code to the some thread pool executor, it will just move problem to another place.

`Retry<T>` is compact single Java class utility without external dependencies to perform non-blocking asynchronous retry procedure on given `Supplier<T>` using CompletableFutures.

Minimalistic sample usage with default retry settings is as following:

```java
CompletableFuture<Connection> futureConnection = 
Retry.of(() -> DriverManager.getConnection("jdbc:mysql:a:b"))
.retry();
```
This code will retry `getConnection(...)` forever with fixed delay 1 second using `ForkJoinPool.commonPool()`. You can specify any other executor for your retry process by using `Retry#withExecutor(...)` method.

```java
CompletableFuture<Connection> futureConnection = 
Retry.of(() -> DriverManager.getConnection("jdbc:mysql:a:b"))
.withFixedDelay(Duration.ofMillis(200))
.retry(5);
```
This code will retry `getConnection(...)` 5 times with fixed delay of 200 millisecond. 

You can retry only on specific exceptions, let`s say IOException
```java
CompletableFuture<Connection> futureConnection = 
Retry.of(() -> DriverManager.getConnection("jdbc:mysql:a:b"))
.retryOnException(IOException.class::isInstance)
.retry();
```

You can retry only on specific returned values, say "Service not available" or null values.
```java
CompletableFuture<Connection> futureConnection = 
Retry.of(() -> DriverManager.getConnection("jdbc:mysql:a:b"))
.retryOnValue(Objects::isNull)
.retry();
```
Retry behavior may be controlled by user supplied backoff function.
In terms of functional programming, backoff function is a `Function<Long,Duration>` which maps current try number (starting with 0)  to the duration to wait until next try will happen.

You can easily implement any desired backoff strategy. Lets say you want Fibonacci backoff, it is simple

```java
    int [] fibonacci = {1,1,2,3,5,8,13,21,34};
    Duration fibonacciBackoff(int i, Duration min, Duration max) {
        if (i >= fibonacci.length) {
            return max;
        } else {
            var d = min.multipliedBy(fibonacci[i]);
            return d.compareTo(max) > 0 ? max : d;
        }
    }

    ...
    CompletableFuture<Connection> futureConnection = 
    Retry.of(() -> DriverManager.getConnection("jdbc:mysql:a:b"))
    .withBackoff(i -> fibonacciBackoff(i, Duration.ofMillis(10), Duration.ofSeconds(1))
    .retry();
```
 

