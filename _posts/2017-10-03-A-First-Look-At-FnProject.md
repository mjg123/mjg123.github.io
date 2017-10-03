---
layout:     post
title:      A First Look at FnProject
date:       2017-10-03 08:11:05
summary:    FnProject plus fdk-java
---

Yesterday at the [Java One Keynote](https://youtu.be/UNg9lmk60sg?t=4653) Mark Cavage and Chad Arimura announced the release of [FnProject](http://fnproject.io/) which is an [Open-Source](https://github.com/fnproject), container-based, cloud-agnostic FaaS platform.

## FnProject

FnProject is **language agnostic** - a function is a container image and what goes in it is *up to you*. The [contract](https://github.com/fnproject/fn/blob/master/docs/function-format.md) is over STDIN/OUT, so if your favourite language can use those then you can use `fn`. Even if your favourite language is [Befunge](https://esolangs.org/wiki/Befunge).

### Getting started, installing Fn

Everything starts with the `fn` command, so we'd better install it:

```shell
$ curl -LSs https://raw.githubusercontent.com/fnproject/cli/master/install | sh
fn version 0.4.6
```

And create our first function

```shell
$ fn init --runtime=go

        ______
       / ____/___
      / /_  / __ \
     / __/ / / / /
    /_/   /_/ /_/

Runtime: go
Function boilerplate generated.
func.yaml created. 
```

We've used `runtime=go` because `runtime=befunge` isn't supported yet (PRs accepted [look here](https://github.com/fnproject/cli/tree/master/langs)), so we have 3 files created by the `go` template.

  * `func.go` is a super-simple go program which:
    * (optionally) reads JSON from STDIN
	* Prints some JSON to STDOUT
  * `func.yaml` is the metadata for this function. For now we'll accept the defaults
  * `test.json` specifies some expected input/output values.

From this point on, you will need a newish version of Docker, which is FnProject's only dependency.

So what can we do?

### Run the function

```shell
$ fn run
Building image my-first-function:0.0.1
Sending build context to Docker daemon   5.12kB
Step 1/8 : FROM funcy/go:dev as build-stage
 ---> 4cccab7fc828
Step 2/8 : WORKDIR /function
 ---> Using cache
 ---> 3cbac4137ce0
Step 3/8 : ADD . /go/src/func/
 ---> cd9dfe6d13f9
Removing intermediate container 8c80673c236f
Step 4/8 : RUN cd /go/src/func/ && go build -o func
 ---> Running in 78ca7f971b61
 ---> bea65786d05d
Removing intermediate container 78ca7f971b61
Step 5/8 : FROM funcy/go
 ---> 573e8a7edc05
Step 6/8 : WORKDIR /function
 ---> Using cache
 ---> 5e1dd9693dd5
Step 7/8 : COPY --from=build-stage /go/src/func/func /function/
 ---> Using cache
 ---> 532f355bbd08
Step 8/8 : ENTRYPOINT ./func
 ---> Using cache
 ---> 14d0cde76c0f
Successfully built 14d0cde76c0f
Successfully tagged my-first-function:0.0.1
{"message":"Hello World"}
```

There's a bit of docker output there, but the last 2 lines are important: We've built a totally standard container image (try it with `docker run` if you don't believe me), and the last line is our function's output.

Check that STDIN parsing is working:

```shell
$ echo '{"name": "Matthew"}' | fn run
...
{"message":"Hello Matthew"}
```

### Test the function

Check that the spec provided in `test.json` is adhered to:

```shell
$ fn test
...
Running 2 tests...running tests on my-first-function:0.0.1 :

Test 1
PASSED -    ( 925.574963ms )

Test 2
PASSED -    ( 1.123083022s )

2 tests passed, 0 tests failed.
```

### Running as a service

Now we've seen an easy way to *create* functions, how about *deploying* them? FnProject can run anywhere a container can. The easiest place to start with is right where you already are, so start the fn service already:

```shell
$ fn start
...
time="2017-10-03T08:18:54Z" level=info msg="available memory" ram=266539061248
time="2017-10-03T08:18:54Z" level=info msg="Serving Functions API on address `:8080`"

        ______
       / ____/___
      / /_  / __ \
     / __/ / / / /
    /_/   /_/ /_/
        v0.3.135

```

That'll sit there in the foreground, so in another terminal we can deploy our app to the localhost server:

```shell
$ fn deploy --app my-app --local
...
Successfully tagged my-first-function:0.0.4
Updating route /my-first-function using image my-first-function:0.0.4...
```

In the command above, `--app my-app` is necessary as functions must be namespaced into apps. I'm using `--local` to skip the step where `fn` will try to push my image to a repository (default: Dockerhub).

Now we can call our function as a web service

```shell
$ curl -d '{"name": "Matthew over HTTP"}' http://localhost:8080/r/my-app/my-first-function
{"message": "Hello Matthew over HTTP"}
```

### Fin

That was a super-quick look at how to get started with FnProject - I hope it was helpful. There's a lot [more documentation](https://github.com/fnproject/fn/tree/master/docs) if you need it, and there's a lot more to fn, too.

Next post I'll look at the Java support. If you have any questions - there's an [official Fn Slack channel](https://join.slack.com/t/fnproject/shared_invite/enQtMjIwNzc5MTE4ODg3LTdlYjE2YzU1MjAxODNhNGUzOGNhMmU2OTNhZmEwOTcxZDQxNGJiZmFiMzNiMTk0NjU2NTIxZGEyNjI0YmY4NTA), problems - there's [GitHub issues](https://github.com/fnproject/fn/issues) or just ask on Slack.
