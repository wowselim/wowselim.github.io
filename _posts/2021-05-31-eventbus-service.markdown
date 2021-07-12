---
layout: post
title:  "Idiomatic Kotlin Abstractions for the Vert.x EventBus"
date:   2021-05-31
tags: vert.x kotlin codegen
---

The EventBus plays an important role in Vert.x applications. It is a convenient way for modules to communicate with each other using various messaging patterns.

Making use of the EventBus is fairly easy. Talking to a service and handling the response could look like this:

```kotlin
vertx.eventBus()
  .localConsumer<Void>(address)
  .handler { message ->
    message.reply(jsonObjectOf("message_of_the_day" to "Hello World"))
  }

val response = vertx.eventBus()
  .request<JsonObject>(address, null)
  .await()
  .body()

println(response.getString("message_of_the_day"))
```

The data types that we can transfer are fairly limited by default, but extensible via custom codecs.

If you make extensive use of the EventBus however, a few things might be reasons for concern:
1. What happens when we have a typo in the address?
2. What happens if the receiving side expects a `String`, but we send an `Int`?
3. How do we handle exceptions?

Sadly, the answer to the first two is runtime errors. So is there a way to avoid them, or even catch them at compile-time? Can we additionally improve exception handling and reduce boilerplate?

In order to answer these questions, let's take a step back and think about what more or less idiomatic kotlin code would look like. Ideally, we would expose a simple interface with the functionality that we would like to provide. The service in the code snippet above could be modelled as follows:

```kotlin
interface MessageOfTheDayService {
  suspend fun getMessageOfTheDay(): Result

  sealed class Result
  data class Success(val message: String): Result()
  data class Failure(val cause: Throwable): Result()
}
```

Simple, right? With a simple kotlin interface, there's no ambiguity in what type this service needs as input and what types of output it can produce. Additionally, when calling it, we don't need to specify an address, or think about exceptions too much. Wouldn't it be nice if we could define our services like this and still make use of the EventBus while not having to worry about any of the drawbacks?

What if we could get all of the benefits above with a single extra line? Turns out with a little bit of annotation processing and code generation we can!

[EventBus-Service](https://github.com/wowselim/eventbus-service) is a small library that takes care of this for us. Let's annotate the interface with `@EventBusService` and then take a look at the generated code:

```kotlin
private const val TOPIC: String = "co.selim.sandbox.messageofthedayservice"

public class MessageOfTheDayServiceImpl(
  private val vertx: Vertx
) : MessageOfTheDayService {
  public override suspend fun getMessageOfTheDay(): MessageOfTheDayService.Result = vertx.eventBus()
    .request<MessageOfTheDayService.Result>(TOPIC + ".getMessageOfTheDay", Unit, deliveryOptions)
    .await()
    .body()
}
```

This looks quite similar to the code we wrote ourselves but there are two important things to note here:
1. The address is inferred from the function defined in our service. This leaves no room for typos.
2. We replaced `Void` and `JsonObject` with the types that we're actually interested in.

Now, how would we reply to these requests? The same file also contains an extension property for handling them:

```kotlin
internal val Vertx.getMessageOfTheDayRequests: Flow<EventBusServiceRequest<Unit,
    MessageOfTheDayService.Result>>
  get() = eventBus()
    .localConsumer<Unit>(TOPIC + ".getMessageOfTheDay")
    .toChannel(this)
    .receiveAsFlow()
    .map { EventBusServiceRequestImpl<Unit, MessageOfTheDayService.Result>(it) }
```

Using these two bits of code, we can avoid all of the problems that were mentioned earlier. Let's rewrite our initial example using the generated code:

```kotlin
vertx.getMessageOfTheDayRequests
  .onEach { (_, reply) -> reply(MessageOfTheDayService.Success("Hello World")) }
  .launchIn(scope)

val motdService = MessageOfTheDayServiceImpl(vertx)
val motd = motdService.getMessageOfTheDay()

when (motd) {
  is MessageOfTheDayService.Failure -> motd.cause.printStackTrace()
  is MessageOfTheDayService.Success -> println(motd.message)
}
```

That was easy, right? We wrote simpler, more idiomatic code while gaining type safety for free. If you're interested in finding out more or trying this yourself, feel free to check out the [GitHub repository](https://github.com/wowselim/eventbus-service) for the project.