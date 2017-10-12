---
layout:     post
title:      FnProject Flow 101
date:       2017-10-10 09:00:00
summary:    First intro to FnProject Flow
---

The makeup of the newly-released [FnProject](http://fnproject.io) is explained in [a good amount of detail](https://twitter.com/chadarimura/status/917706536759234560) by Chad Arimura, with one of the major components being **Fn Flow**. Flow allows developers to build high-level workflows of functions with some notable features:

  - Very flexible model of function composition. Sequencing, fan out/in, retries, error-handling and more
  - Code-driven. Flow does not rely on massive yaml descriptors or visual graph-designers. Workflows are defined *in code*, expressed as a function, naturally.
  - Inspectable. Flow shows you the current state of your workflow, allows you to drill into each stage to see logs, stack traces and more.
  - Language agnostic. The initial Flow implementation which this post will use is in Java but support for other langauges has already started. TODO: LINKS!!

--

* TOC
{:toc}

  
## What is a Flow?

A Flow is a way of linking together functions, and incidentally provides a way to define those functions inline if you need to. It's a FaaS-friendly way of saying stuff like

> Start with **this**, then do **that**, then take the result and do **these things** in parallel then when they've all finished do **this one last thing**, and if there's any errors then do **this** to recover

Where all the *this* and *that* are FaaS functions.


Lets see what a super-simple Flow function looks like:

```java

    public String handleRequest(int x) {

        Flow flow = Flows.currentFlow();

        return fl.completedValue(x)
                 .thenApply(i -> i+1)
                 .thenCompose( s -> Flows.invokeFunction("./isPrime", s) )
                 .get();
    }

```

If you've used a promises-style API before then this will be very familiar. The closest analogue in core Java is the [CompletionStage API](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html) which was [apparently](http://cs.oswego.edu/pipermail/concurrency-interest/2012-December/010423.html) called `Promise` in a pre-release draft.

Anyway it's easy to tell the stages of what's going to happen:

  - Start with a value provided by the user
  - Apply some transformation `i -> i+1`
  - Pass that to an external function called `./isPrime`
  - Then return get the result and return it

Internally the `Flow` class submits each stage in this workflow to an Fn service we call the "Completer". You'll meet it soon. The Completer will then orchestrate each stage as an individual call to Fn. The completer is responsible for working out which stages are ready to be called, calling them, handling the results and triggering any following stages until you reach the point where there's no more work to do.

This example could easily be written without Flow but it's good to start simple.

## Running your first Flow

Currently FnProject is available to download, to experiment with, and to run on your private cloud. A managed service by Oracle is in the works. To play with Flow at the moment you will need to run everything locally, but it's not hard. We need **`fn`**, the **Fn server**, the **Completer** and not necessary but nice-to-have is the Completer **UI**

### Setting up

Install the **`fn`** CLI tool:

```shell
⇒ curl -LSs https://raw.githubusercontent.com/fnproject/cli/master/install | sh
```

Then start the **Fn server**:

```shell
⇒ fn start
...
time="2017-10-11T13:12:44Z" level=info msg="Serving Functions API on address `:8080`"
        ______
       / ____/___
      / /_  / __ \
     / __/ / / / /
    /_/   /_/ /_/
        v0.3.119
```

The **Completer** needs to know how to call the Fn server, so ask Docker which IP address to use.

```shell {% raw %}
⇒ DOCKER_LOCALHOST=$(docker inspect --type container -f '{{.NetworkSettings.Gateway}}' functions)
{% endraw %}```

Start the **Completer**:

```shell
⇒ docker run --rm -d \
       -p 8081:8081 \
       -e API_URL="http://$DOCKER_LOCALHOST:8080/r" \
       -e no_proxy=$DOCKER_LOCALHOST \
       --name completer \
       fnproject/completer:latest
```

Then start the Completer **UI**:

```shell
⇒ docker run --rm -d \
       -p 3002:3000 \
       --name flowui \
       -e API_URL=http://$DOCKER_LOCALHOST:8080 \
       -e COMPLETER_BASE_URL=http://$DOCKER_LOCALHOST:8081 \
       fnproject/completer:ui
```

Now, everything's set so lets crack on!

### A simple Flow function

Create a new function:

```shell
⇒ fn init --runtime=java simple-flow
⇒ cd simple-flow
```

Flow has a comprehensive test framework, but lets concentrate on playing with the code for the time being:

```shell
⇒ rm -rf src/test   ## yolo
```

Make peace with yourself after that, then let's get the code in shape. Change `HelloFunction.java` to look like this:

```java
package com.example.fn;

import com.fnproject.fn.api.flow.Flow;
import com.fnproject.fn.api.flow.Flows;

public class HelloFunction {

    public String handleRequest(int x) {

	Flow fl = Flows.currentFlow();

	return fl.completedValue(x)
                 .thenApply( i -> i*2 )
	         .thenApply( i -> "Your number is " + i )
	         .get();	
    }
}
```

Then deploy this to an app which we call `simple` on the local Fn server, and configure the function to talk to the Completer

```shell
⇒ fn deploy --app simple --local
⇒ fn apps config set simple COMPLETER_BASE_URL "http://$DOCKER_LOCALHOST:8081"
```

You can now invoke the function using `fn call`:

```shell
⇒ echo 2 | fn call simple /simple-flow
Your number is 4
```

or equivalently with `curl`:

```shell
⇒ curl -d "2" http://localhost:8080/r/simple/simple-flow
Your number is 4
```

### Exploring the UI

Browsing to [http://localhost:3002](http://localhost:3002) you should see something like this:

![flow-ui]({{ "assets/simple-flow-ui.png" | absolute_url }})

Which is showing us 3 function invocations:

  * The main flow function, in blue
  * `.thenApply` for the code `i -> i*2`
  * `.thenApply` for the code `i -> "Your number is " + i`
  
Click on any of these and see the detail for each one expanded at the bottom of the screen.

The blue function is shown as running for the whole time that the `thenApply` stages are. Why? Because we are calling `.get()` at the end, so this is synchronously waiting for the final result of the chain. Exercise: Try removing the `.get()` from the code (you'll need to return another String, and don't forget to re-deploy). Now it will look like:

![flow-ui]({{ "assets/simple-flow-ui-async.png" | absolute_url }})

This shows that Flow is well-suited for asynchronous functions which result in a side-effect (posting to slack, for example)

## Summary

Congratulations - we've covered a lot! You've got a Flow function running, seen how to use the API to compose simple transformations and run things in parallel. Head to the [Flow 102](TBD) post to take your Flows to the next level.
