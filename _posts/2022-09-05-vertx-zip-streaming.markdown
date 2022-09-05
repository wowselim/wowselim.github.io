---
layout: post
title:  "Streaming Zip Archives to the Browser using Vert.x & Coroutines"
date:   2022-09-05
tags: vert.x kotlin async coroutines
---

When we want to serve users multiple files as a download, a common way is to combine those files into an archive. The JDK
provides the `ZipOutputStream` to build ZIP archives. Unfortunately this is based on blocking IO so using it with Vert.x
can be a bit cumbersome. It becomes even trickier if you want to skip storing the archive on the file system temporarily.
Let's see how we can make things easier using Kotlin's coroutines.

Assume we want to send a number of photos to the user:

```kotlin
ctx.response().isChunked = true
ctx.response().putHeader(HttpHeaders.CONTENT_TYPE, "application/zip")
ctx.response().putHeader(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"album.zip\"")

val byteArrayOutputStream = ByteArrayOutputStream()
ZipOutputStream(byteArrayOutputStream).use { zip ->
  images.forEach { image ->
    awaitBlocking { zip.putNextEntry(ZipEntry("${image}.jpeg")) }
    val bytes = vertx.fileSystem().readFile(image).await()
    awaitBlocking {
      zip.write(bytes.bytes)
      zip.closeEntry()
      zip.flush()
    }
    ctx.response().write(byteArrayOutputStream.toBuffer()).await()
    byteArrayOutputStream.reset()
  }
}

ctx.end(byteArrayOutputStream.toBuffer()).await()
```

Using chunked transfer encoding available in HTTP 1.1, we can send smaller chunks of data to the browser. In the snippet above
we are reading each image into memory, compressing it and sending it to the browser.

There are two drawbacks with this approach:
* By using chunked transfer encoding, the browser doesn't know how many bytes will be sent in total. This means that users will
typically see an indeterminate progress indicator and they won't know how long they are going to be waiting for the download to
finish.
* While reading images into memory might be OK, this approach falls apart when we're dealing with a large number of concurrent download
requests or when we want to serve larger files. Our application will run out of memory pretty quickly.

While we can't do anything about the first drawback, let's see how we can improve the memory consumption of our approach.

```kotlin
ctx.response().isChunked = true
ctx.response().putHeader(HttpHeaders.CONTENT_TYPE, "application/zip")
ctx.response().putHeader(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"archive.zip\"")

val byteArrayOutputStream = ByteArrayOutputStream()
ZipOutputStream(byteArrayOutputStream).use { zip ->
  images.forEach { image ->
    vertx.fileSystem().open(image, OpenOptions())
      .await()
      .use { channel ->
        awaitBlocking { zip.putNextEntry(ZipEntry("${image}.jpeg")) }
        for (chunk in channel) {
          awaitBlocking {
            zip.write(chunk.bytes)
            zip.flush()
          }
          ctx.response().write(byteArrayOutputStream.toBuffer()).await()
          byteArrayOutputStream.reset()
        }
      }
    awaitBlocking {
      zip.closeEntry()
      zip.flush()
    }
    ctx.response().write(byteArrayOutputStream.toBuffer()).await()
    byteArrayOutputStream.reset()
  }
}

ctx.end(byteArrayOutputStream.toBuffer()).await()
```

While this is a bit trickier to read it allows us to read smaller amounts of data at any given time, reducing the memory
footprint of our application.

Coroutines make the code a lot easier to read and we can define some helper methods to make the
above examples compile.

The `AsyncFile` class that comes with Vert.x doesn't implement `Closeable` so we can define our own version to prevent
resource leaks:
```kotlin
private suspend fun AsyncFile.use(block: suspend (ReceiveChannel<Buffer>) -> Unit) {
  try {
    block(toReceiveChannel(vertx))
  } finally {
    close().await()
  }
}
```

This extension function reduces some boilerplate when wrapping byte arrays in a Vert.x `Buffer`:
```kotlin
private fun ByteArrayOutputStream.toBuffer(): Buffer {
  return Buffer.buffer(toByteArray())
}
```
