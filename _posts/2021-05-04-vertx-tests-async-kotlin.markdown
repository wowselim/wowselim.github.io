---
layout: post
title:  "Writing Async Tests for Vert.x using Kotlin"
date:   2021-05-04T01:00:00Z
tags: vert.x kotlin testing async java
---

Testing asynchronous code can be a challenging task. Usually some tooling is required to make it work with testing frameworks like junit.

For Vert.x projects, there's an [official junit module](https://vertx.io/docs/vertx-junit5/java/) to help writing these tests.
The `VertxTestContext` in that module is comparable to a standard `CountdownLatch`. The testing framework waits for the latch to unlock and a callback does the unlocking.

Unfortunately this leads to a quite a bit of boilerplate code for simple unit tests. Luckily though, Kotlin's coroutines provide a clean solution to write tests in a more traditional-looking way.

In this article, we're going to explore how this can be achieved by migrating a simple example to use Kotlin's coroutines.

Here's a slightly modified version of the first test:

```java
@Test
@Timeout(value = 5, timeUnit = TimeUnit.SECONDS)
void service_is_healthy(Vertx vertx, VertxTestContext testContext) {
  HttpClient httpClient = vertx.createHttpClient();

  httpClient.request(HttpMethod.GET, 8080, "localhost", "/health")
    .compose(HttpClientRequest::send)
    .compose(HttpClientResponse::body)
    .onComplete(testContext.succeeding(buffer -> {
      JsonObject responseJson = buffer.toJsonObject();

      Assertions.assertEquals("up", responseJson.getString("status"));
      testContext.completeNow();
    }));
}
```

Now, what are the problems with this code? Notice how the the test context clutters the test code. Wouldn't it be nicer if we didn't have to worry about it?

Let's rewrite this using Kotlin coroutines:

```kotlin
@Test
@Timeout(5, unit = TimeUnit.SECONDS)
fun `service is healthy`(vertx: Vertx): Unit = runBlocking(vertx.dispatcher()) {
  val httpClient = vertx.createHttpClient()

  val request = httpClient.request(HttpMethod.GET, 8080, "localhost", "/health").await()
  val response = request.send().await()
  val responseJson = response.body().await().toJsonObject()

  Assertions.assertEquals("up", responseJson.getString("status"))
}
```

As you can see, we got rid of almost everything that's unrelated to the actual test. This is the main benefit of using coroutines for tests with Vert.x.

Let's go over the changes that we introduced here. First, we wrapped the test body in `runBlocking` in order to be able to start coroutines inside the test.
> Note: It's important that the coroutines run on the Vert.x threads. Luckily, Vert.x provides a `#dispatcher` function that we can use for `runBlocking`.

The `#await` calls make sure that exceptions make the tests fail. They also force junit to wait for our test, esentially making the `VertxTestContext` obsolete for our tests.

On top of that, we avoid callbacks and nesting. This makes our tests a little more readable.

## Summary
In this short article, we saw that Kotlin's coroutines can vastly reduce the noise in asynchronous tests. Instead of fighting with the asynchronous nature of our code, we can focus on the tests themselves. In [the next part]({{site.baseurl}}{% link _posts/2021-05-04-vertx-tests-boilerplate.markdown %}), we will see how we can add some helpers to make writing tests even easier.

## Source
A full project can be found in this [GitHub repo](https://github.com/wowselim/async-testing#readme).
