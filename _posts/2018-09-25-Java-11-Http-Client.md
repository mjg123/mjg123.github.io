---
layout:     post
title:      The new Java HTTP client and the CompletionStage API
date:       2018-09-25 10:00:00
summary:    Java 11
tags:
- java
---

Happy Java 11 day!  This is a big release for Java - serveral interesting new features have been added, which you can see listed at [http://openjdk.java.net/projects/jdk/11/](http://openjdk.java.net/projects/jdk/11/). I'm sure there will be lots of _What's new in Java 11_ posts around, so I'd like to dive into detail on one specific feature from 11:

## JEP 321: HTTP Client

I suspect most people who have used an HTTP client from Java would have settled on the (excellent) [Apache Commons library](http://hc.apache.org/httpclient-3.x/tutorial.html), but in fact Java has had its own built-in HTTP client for more than 20 years.  [`HttpURLConnection`](https://docs.oracle.com/javase/8/docs/api/java/net/HttpURLConnection.html) has been there since Java 1.1. Using it looks like this:

```java
// Create a neat value object which represents a URL
URL url = new URL("http://example.com");

// Open a connection (?!) on the URL (?!!) and _cast_ the response (?!!!)
HttpURLConnection conn = (HttpURLConnection) url.openConnection(); 
// BTW do we capitalize acronyms or not?

// We can only set method, headers, etc _after_ the connection is opened (?!!!!)
conn.setRequestProperty("x-header", "value")

// So.... is this the line that actually causes the request to be made?
int responseCode = con.getResponseCode();

// Now we've got an InputStream to deal with - BufferedReader and iterate or IOUtils??
InputStream response = conn.getInputStream();     // Don't forget to close it LOL
```

As you can tell I'm not a huge fan of this 1990s API, so I was glad to see a more modern HTTP client come out of incubation in Java 11, and into the `java.net.http` package (and module of the same name). JavaDoc is [here](https://download.java.net/java/early_access/jdk11/docs/api/java.net.http/module-summary.html). The headline features are:

  - Support for HTTP/1.1, [HTTP/2](https://http2.github.io/faq/) and [WebSockets](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
  - Explicit support for HTTP proxies
  - Builder pattern for creating objects
  - Synchronous and Asynchronous APIs

Here's how a synchronous call looks:

```java
import java.net.http.*

// Create a client
var client = HttpClient.newHttpClient()

// Create a request object
var request = HttpRequest.newBuilder().uri(URI.create("http://example.com")).build();

// Make the request
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());

// Print the response
System.out.println(response.body())
```

This is a breath of fresh air compared to the old version isn't it?

I particularly like the [`BodyHandler`](https://download.java.net/java/early_access/jdk11/docs/api/java.net.http/java/net/http/HttpResponse.BodyHandler.html) abstraction and the [built-in implementations](https://download.java.net/java/early_access/jdk11/docs/api/java.net.http/java/net/http/HttpResponse.BodyHandlers.html) as it is often the case that you immediately need to parse the body of a response.

However, my favourite feature is how asynchronous calls are implemented using `CompletableFuture`. Now I've baited you this far with talk of HTTP clients I'll switch over to the CompletionStage API.

## The CompletionStage API and CompletableFutures

When Java 8 was released there were a lot of attention-grabbing features such as lambdas and the streams API, so the CompletionStage API is often overlooked. In my experience very few Java developers know about it, and even fewer use it, but it's an incredibly powerful framework for building asynchronous workflows.

The basics: [`CompletionStage`](https://download.java.net/java/early_access/jdk11/docs/api/java.base/java/util/concurrent/CompletionStage.html) is an interface. The only implementation is [`CompletableFuture`](https://download.java.net/java/early_access/jdk11/docs/api/java.base/java/util/concurrent/CompletionStage.html). A `CompletableFuture` also implements [`java.util.concurrent.Future`](https://download.java.net/java/early_access/jdk11/docs/api/java.base/java/util/concurrent/Future.html), as added in Java 5.

### What you get from j.u.c.Future

As with all other `Future` objects, a `CompletableFuture` is like a box which will at some point contain the result of an asynchronous action, like this:

```java
// This returns the future immediately.
Future f = ForkJoinPool.commonPool().submit( longRunningAction );

// We can do other stuff while the longRunningAction is happening on another thread
// This will block until the Future is completed:
Value v = f.get()
```

### What you get from j.u.c.CompletionStage

In addition to the `Future` behaviour, the `CompletionStage` API provides about 50 methods for combining individual stages by chaining, fan-out, fan-in, error handling and other means. Here's an example using the new HttpClient:

```java
var client = HttpClient.newHttpClient();
var req = java.net.http.HttpRequest.newBuilder()
                                   .uri(URI.create("http://example.com/face.png")).build();

CompletableFuture<HttpResponse<byte[]>> resp = 
                             client.sendAsync(req, HttpResponse.BodyHandlers.ofByteArray());
```

... now we have a `CompletableFuture` which will contain the response once its available. Note that we've made a request for a binary file (a png image of a face) this time, so we use the `ofByteArray` BodyHandler to get a `byte[]` out.  What next?

```java
var image = resp
        .thenApply( HttpResponse::body )           // now we have a CompletableFuture<byte[]>
        .thenApply( ByteArrayInputStream::new )    // NB constructor as method reference
        .thenApply( javax.imageio.ImageIO::read ); // now we have a CompletableFuture<BufferedImage>
```

Hopefully it's easy to see how [`thenApply`](https://download.java.net/java/early_access/jdk11/docs/api/java.base/java/util/concurrent/CompletionStage.html#thenApply(java.util.function.Function)) can make chaining methods together simple. The CompletableFuture implementation takes care of applying each stage once the previous one has finished, so this whole flow is asynchronous and we still need to call `.get()` if we want to block on the resulting `BufferedImage`.

Instead of just getting the BufferedImage, lets use an (imaginary) API to detect moods from images of faces. We have 3 different algorithms in our (imaginary) API, which can all be run concurrently:

```java
var mood1 = image.thenApply( moodDetectionSimple );
var mood2 = image.thenApply( moodDetectionComplex );
var mood3 = image.thenApply( moodDetectionExperimental );
```

By putting 3 `.thenApply` calls off the same CompletionStage we've made a super-simple fan-out. Now we need to wait for them all to finish. We can use the static method [`CompletableFuture.allOf`](https://download.java.net/java/early_access/jdk11/docs/api/java.base/java/util/concurrent/CompletableFuture.html#allOf(java.util.concurrent.CompletableFuture...)) to wait for multiple stages, and call `.join` on each stage to get its value.

```java
// .allOf returns a CompletableFuture<Void>
// We ignore the void parameter to the lambda in the next stage
var finished = CompletableFuture.allOf(mood1, mood2, mood3)
                                .thenApply( ignored ->
    combineResults(mood1.join(), mood2.join(), mood3.join()));
```

Why use `.join()` instead of `.get()`?  Because `join` throws only _unchecked_ exceptions, so we don't need to handle the possibility that one of the moodDetection algorithms threw an exception. Instead we can use the CompletionStage API's exception handling mechanisms. Because all the stages are happening asynchronously (ie on another thread, normally managed by the ForkJoinPool) we have no way of using try/catch mechanisms at the top level to handle errors, and it would add a lot of noise to the code to have to handle errors in every lambda. Instead there are a few methods like this:

```java
finished.whenComplete((result, exception) -> {...});
```

Exactly one of `result` or `exception` will be non-null. So we can handle errors in the same asynchronous way as we do all our other work. Neat, isn't it?

### The Method Zoo

Remember when I said there were around 50 methods on [CompletionStage](https://download.java.net/java/early_access/jdk11/docs/api/java.base/java/util/concurrent/CompletionStage.html) and [CompletableFuture](https://download.java.net/java/early_access/jdk11/docs/api/java.base/java/util/concurrent/CompletableFuture.html)? That sounds like a lot but they fall into a few categories. Generally there is a lot of repetition depending on whether you are handling values with a `Function` or a `Consumer` or a `Runnable` and depending on whether you want to use the default executor service or provide your own, or a new thread.

#### Methods which create a CompletableFuture from nothing

```java
CompletableFuture.completedFuture( T val )
CompletableFuture.completedStage( T val )
CompletableFuture.failedFuture( Throwable ex )
CompletableFuture.failedStage( Throwable ex )
CompletableFuture.supplyAsync( Supplier s()->val ) // also has another Async version
```

#### Methods which chain a CompletionStage off another

```java
CompletionStage.thenAccept( Consumer c ) // also has Async versions
CompletionStage.thenApply( Function f )  // also has Async versions
CompletionStage.thenRun( Runnable r )    // also has Async versions
```
#### Methods which join multiple CompletionStages together

```java
CompletableFuture.allOf(CompletionStage...)
CompletableFuture.anyOf(CompletionStage...)
CompletionStage<X>.thenCombine( otherStage<Y>, Function f(x,y)->z )   // also has Async versions
CompletionStage<X>.acceptEither( otherStage<Y>, Consumer c(x,y) )     // also has Async versions
CompletionStage<X>.applyToEither( otherStage<Y>, Function f(x,y)->z ) // also has Async versions
CompletionStage<X>.runAfterEither( otherStage<Y>, Runnable r )        // also has Async versions
```

#### Methods which handle errors from other stages

```java
CompletionStage.handle( Function f(value,exception)->newValue )  // also has Async versions
CompletionStage.whenComplete( Consumer c(value,exception) )      // also has Async versions
CompletionStage.exceptionally( Function f(exception)->newValue ) // skips non-exceptions
```

NB in these there will be _either_ a value _or_ an exception, not both.

#### Methods which apply timeouts

```java
CompletableFuture.completeOnTimeout( val, time ) // timeout with a value
CompletableFuture.orTimeout( time )              // timeout with an exception
```

#### Methods which add new CompletionStages at runtime

```java
CompletionStage.thenCompose( Function f(x)->CompletionStage )  // also has Async versions
```

This is a really powerful method, and IMO the hardest to understand. Instead of returning a value, the provided `Function` has to return a `CompletionStage` which is executed under the same rules as the original `CompletionStage`. Once the new stage has completed its value is used for `thenCompose`. For example you can use this to create a retry mechanism, where you don't know in advance how many stages you will need.


### Why CompletionStage?

For me, the biggest appeal of using CompletionStages is to define in a _single place_ a high-level asynchronous workflow. Callback-style code tends to scatter the actual logic around more than I want. Additionally, using the ForkJoinPool gives us an excellent work-stealing thread pool for good performance.

## Back to HTTP clients

Making HTTP requests is naturally an asychronous business. Any time we make a request there might be a delay before the response comes back, and using the CompletionStage API lets us avoid the trap of blocking a thread while waiting for the response. Instead we just provide a definition of what we would like to happen when the response is ready and let the CompletableFutures and ForkJoinPool handle getting the work done as quickly as possible. So this is certainly a _huge_ improvement over HttpURLConnection, and is a welcome addition to core Java.
