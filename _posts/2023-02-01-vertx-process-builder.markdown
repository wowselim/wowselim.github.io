---
layout: post
title:  "Starting External Processes from Vert.x"
date:   2023-02-01
tags:   vert.x kotlin coroutines
---

Sometimes it's necessary to run a script or command line application from your
JVM applications. In my case, I wanted to generate my thumbnails using the
ImageMagick CLI tools. Luckily the `convert` utility has support for reading
images from standard input and writing the thumbnails to standard output.

This is the code I ended up with:

```kotlin
private val QUALITY_PRESET = 85

private suspend fun InputStream.createThumbnail(px: Int): ByteArray {
  val command = listOf("convert", "-thumbnail", "${px}x${px}>", "-quality", "$QUALITY_PRESET", "-", "webp:-")

  return awaitBlocking {
    val convertProcess = ProcessBuilder(command)
      .redirectError(ProcessBuilder.Redirect.INHERIT)
      .start()

    convertProcess.outputStream.use(this::transferTo)

    convertProcess.inputStream.use(InputStream::readAllBytes)
  }
}
```

Here we are starting `convert` and giving it the desired resolution. The `>`
in the third list entry means that we don't want to scale up images that
are smaller than our target resolution. The last two entries specify the
input and output methods for the tool. The `-` signals standard IO and the
`webp:` specifies the output format for the thumbnails. Once the process is
started, we can feed it our image input stream and then read the thumbnail
from the process.
