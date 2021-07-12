---
layout: post
title:  "Reducing Boilerplate in Vert.x Tests written in Kotlin"
date:   2021-05-04T02:00:00Z
tags: vert.x kotlin testing
---

In the [previous part]({{site.baseurl}}{% link _posts/2021-05-04-vertx-tests-async-kotlin.markdown %}), we saw how Kotlin can reduce noise in asynchronous tests. In this part we will introduce some syntactic sugar that will help us avoid repetition in our Verticle tests.

When testing a Verticle, we need to tell junit to deploy the Verticle before the test, and undeploy it afterwards. This is fairly straightforward but it can be tedious to add to every single test file:

```kotlin
  private lateinit var deploymentId: String

  @BeforeEach
  fun deployVerticle(): Unit = runBlocking(vertx.dispatcher()) {
    deploymentId = vertx.deployVerticle(verticleClass.java, DeploymentOptions()).await()
  }

  @AfterEach
  fun undeployVerticle(): Unit = runBlocking(vertx.dispatcher()) {
    vertx.undeploy(deploymentId).await()
  }
```

There's a very simple way we can avoid this boilerplate. If we move this into an abstract class that does this once, we can simply extend it and focus on our tests instead.

We should make sure that this is flexible enough so in this case we want some configuration options. We should be able to:
* Specify which Verticle we would like to test
* Pass configuration to this Verticle such as a database URI or a temporary directory
* Customize the deployment if necessary
* Run test code on Vert.x threads

So let's define this helper class:

```kotlin
abstract class AsyncTest(
  private val vertx: Vertx,
  private val verticleClass: KClass<out Verticle>,
  private val deploymentOptions: DeploymentOptions = DeploymentOptions()
) {
  private lateinit var deploymentId: String

  open fun deployVerticle(): String = runBlocking(vertx.dispatcher()) {
    vertx.deployVerticle(verticleClass.java, deploymentOptions).await()
  }

  @BeforeEach
  fun assignDeploymentId() {
    deploymentId = deployVerticle()
  }

  @AfterEach
  fun undeployVerticle(): Unit = runBlocking(vertx.dispatcher()) {
    vertx.undeploy(deploymentId).await()
  }

  protected fun runTest(block: suspend () -> Unit): Unit = runBlocking(vertx.dispatcher()) {
    block()
  }
}
```

If we have things like Http clients, we can also put them in here and make them accessible to all tests. Now, when we want to test a Verticle, we simply extend this class add tests:

```kotlin
@ExtendWith(VertxExtension::class)
class TestMainVerticle(vertx: Vertx) : AsyncTest(vertx, MainVerticle::class) {

  @Test
  fun `service is healthy when deployed`() = runTest {
    val request = httpClient.request(HttpMethod.GET, 8080, "localhost", "/health").await()
    val response = request.send().await()
    val responseJson = response.body().await().toJsonObject()

    assertEquals("up", responseJson.getString("status"))
  }
}
```

Our helper class will make sure that:
* The verticle is deployed & undeployed for each test
* Tests wrapped in `runTest` run on a Vert.x thread automatically

## Summary
In this part, we looked at how to further reduce boilerplate in our asynchronous tests. In [the next part]({{site.baseurl}}{% link _posts/2021-05-04-vertx-testcontainers.markdown %}), we will see how we can combine everything we've learned so far in order to write integration tests using Testcontainers.

## Source
A full project can be found in this [GitHub repo](https://github.com/wowselim/async-testing#readme).
