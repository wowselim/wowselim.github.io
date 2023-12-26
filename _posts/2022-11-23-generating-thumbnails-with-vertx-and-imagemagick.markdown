---
layout: post
title:  "Writing Vert.x Integration Tests with Kotlin & Testcontainers"
date:   2021-05-04T03:00:00Z
tags: vert.x kotlin testing testcontainers
---

Testcontainers are widespread in integration testing where mocking external services such as databases may not be desirable. In this article, we will explore how to use Testcontainers in our Verticle tests.

In the [previous article]({{site.baseurl}}{% link _posts/2021-05-04-vertx-tests-boilerplate.markdown %}) we built a helper class that served as a starting point for all Verticle tests. For our integration tests, we will define a new class. In this example we will be using a Postgres database.

```kotlin
private const val DB_NAME = "integration"
private const val DB_USERNAME = "integration"
private const val DB_PASSWORD = "integration"

@Testcontainers
@ExtendWith(VertxExtension::class)
abstract class IntegrationTest(
  private val vertx: Vertx,
  private val verticleClass: KClass<out Verticle>,
  private val deploymentOptions: DeploymentOptions = DeploymentOptions()
) : AsyncTest(vertx, verticleClass, deploymentOptions) {
  companion object {
    @Container
    private var postgresqlContainer: PostgreSQLContainer<Nothing> = PostgreSQLContainer<Nothing>("postgres:12")
      .apply {
        withDatabaseName(DB_NAME)
        withUsername(DB_USERNAME)
        withPassword(DB_PASSWORD)
      }
  }

  override fun deployVerticle(): String = runBlocking(vertx.dispatcher()) {
    val dbUri = postgresqlContainer.jdbcUrl.substringAfter("jdbc:")
    deploymentOptions.config = jsonObjectOf(
      "db.uri" to dbUri,
      "db.username" to DB_USERNAME,
      "db.password" to DB_PASSWORD,
    )
    vertx.deployVerticle(verticleClass.java, deploymentOptions).await()
  }
}
```

The `@Testcontainers` and `@Container` annotations will let junit take care of the container lifecycle for us. For more details check the [official docs](https://www.testcontainers.org/test_framework_integration/junit_5/).

By overriding the `deployVerticle` function, we can update the Verticle config and insert our database credentials for the test. Since the Verticle in our example uses the reactive Postgres client, we need to remove the `jdbc:` prefix of the URI. The Verticle can access this config as follows:

```kotlin
private val pool: Pool by lazy {
  val connectOptions = PgConnectOptions.fromUri(config["db.uri"])
    .setUser(config["db.username"])
    .setPassword(config["db.password"])
  PgPool.pool(vertx, connectOptions, PoolOptions())
}
```

This means that our Verticle will now be using our testcontainer for all its queries. We can now purely focus on writing tests. The following test will automatically start a database, deploy our Verticle and call an Http endpoint:

```kotlin
@ExtendWith(VertxExtension::class)
class HealthTest(vertx: Vertx) : IntegrationTest(vertx, MainVerticle::class) {
  @Test
  fun dbIsUp() = runTest {
    val request = httpClient.request(HttpMethod.GET, 8080, "localhost", "/db-health").await()
    val response = request.send().await()
    val responseJson = response.body().await().toJsonObject()

    Assertions.assertEquals("up", responseJson.getString("postgres"))
  }
}
```

## Summary
In the last article of the series, we looked at how to test a Vert.x application against a real database using Testcontainers. We were able to hide the complexity of the database lifecycle and Verticle deployment behind a simple helper class.

## Source

A full project can be found in this [GitHub repo](https://github.com/wowselim/async-testing#readme).
