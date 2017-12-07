---
layout:     post
title:      Announcing Spring Cloud Function support for Fn Project
date:       2017-12-02 14:05:00
summary:    Deploying Spring Cloud Function code on Fn Project
tags:
- fnproject
- fdk-java
---

NB Similar to the JAX-RS support we [announced](/2017/11/08/Jax-RS-on-Fn.html) last month, the Spring Cloud Function support on Fn Project was largely done by one of our awesome summer interns from the [University of Bristol](http://www.bristol.ac.uk/) - [Will Price](https://about.me/will_price).

## Spring Cloud Function

Spring has long been a byword for powerful Java libraries which emphasize ease of use. So it was no surprise when [Spring Cloud Function](https://github.com/spring-cloud/spring-cloud-function) (singular) [was announced](https://spring.io/blog/2017/07/05/introducing-spring-cloud-function) earlier this year. From their post, the goals are:

  - Promote the implementation of business logic via functions.
  - Decouple the development lifecycle of business logic from any specific runtime target so that the same code can run as a web endpoint, a stream processor, or a task.
  - Support a uniform programming model across serverless providers, as well as the ability to run standalone (locally or in a PaaS).
  - Enable Spring Boot features (auto-configuration, dependency injection, metrics) on serverless providers.

Especially in light of #3, it seems natural to provide support for using Spring Cloud Function on Fn Project.


## A look at the code

Please have a look around our [Spring Cloud Function example project](https://github.com/fnproject/fn-spring-cloud-function-example).

In typical Spring style, there is a lot of configuration-by-convention which means that code can be very straightforward. The extra work you need to do to have your Fn Project function use Spring Cloud Function is to override the [FunctionInvoker](https://github.com/fnproject/fdk-java/blob/master/api/src/main/java/com/fnproject/fn/api/FunctionInvoker.java) using a [configuration method](https://github.com/fnproject/fdk-java/blob/master/docs/FunctionConfiguration.md). You need to pass a reference to a class from which Spring can create an ApplicationContext, so it's sensible to make your configuration method static and pass in a reference to its own class:

```java
@Configuration
@Import(ContextFunctionCatalogAutoConfiguration.class)
public class SCFExample {

    @FnConfiguration
    public static void configure(RuntimeContext ctx) {
        ctx.setInvoker(new SpringCloudFunctionInvoker(SCFExample.class));  // <-- like that
    }

    // Unused. See https://github.com/fnproject/fdk-java/issues/113
    public void handleRequest() { }

    @Bean
    public Function<String, String> function() {
        return String::toUpperCase;
    }
}
```

Then, define the function in your `func.yaml` like this:

```yaml
cmd: com.fnproject.fn.springcloudfunction.CSFExample::handleRequest
```


### Which function will be called?

Your functions are defined as Spring Beans. The method defined in your `func.yaml` will not be called (in fact, we will be removing the need to define it at all in a future release).

The `@Bean` annotated methods in Spring Cloud Function code are factory methods - they are called once at initialization time and return lambdas which are cached and used when your function is invoked. The beans are named after the method which created them. By default our integration will look for:

  - A `@Bean` called `function` which returns an instance of `java.util.function.Function`
  - A `@Bean` called `consumer` which returns an instance of `java.util.function.Consumer`
  - A `@Bean` called `supplier` which returns an instance of `java.util.function.Supplier`

These can be overridden by providing environment variables `FN_SPRING_{FUNCTION,CONSUMER,SUPPLIER}` which name `@Bean` instances of the appropriate type.


## Integration with other Spring components

As Spring is managing the lifecycle of your components it is fine to use the regular Spring application features, for example `@AutoWired` to inject dependencies or `@Repository` to set up really simple database access.

## Fluxin'

As well as the normal `Function<IN, OUT>` -type functions, Spring Cloud Function also supports reactive streams, by allowing [Project Reactor](https://projectreactor.io) `Flux`-es as to represent a stream of input events: `Function<Flux<IN>, Flux<OUT>>`. In this case you deal with your input events as a stream rather than individually and you have loads of control over how they are treated. A simple example looks like:

```java
    @Bean
    public Function<Flux<String>, Flux<String>> lowercase() {
        return flux -> flux.map(value -> value.toLowerCase());
    }
```


You can split/map/filter/combine/batch/etc your input events with a [super flexible](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html) API. This seems like a good fit for high-velocity events such as IoT sensor data.


## Summary

Spring has been one of the leading Java application frameworks for over a decade, so many Java developers will be familiar with Spring apps. Spring Cloud Function extends that familiarity into FaaS in a provider-agnostic way and we are really happy to be able to offer support on Fn Project.

For more detail, please see the [example app](https://github.com/fnproject/fn-spring-cloud-function-example#example-spring-cloud-function) which details build and deployment of an app onto an Fn server.
