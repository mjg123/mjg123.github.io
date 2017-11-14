---
layout:     post
title:      Announcing JAX-RS support for Fn
date:       2017-11-08 14:48:00
summary:    Deploying RESTful Java Apps on Fn
tags:
- fnproject
- fdk-java
- draft
---

NB This is a guest post by our awesome intern [Rae Jeffries-Harris](https://github.com/raej), describing part of the work she did with our Bristol team over the summer. Thanks Rae.

## JAX-RS on FaaS?

A lot of Java programmers are familiar with building RESTful web applications using frameworks such as [Jersey](https://jersey.github.io/) which adhere to the [JAX-RS](https://github.com/jax-rs) specification. JAX-RS abstracts much of the low-level detail away from the programmer, so that tasks such as client-server communication become less fiddly.

But you still have to have a server running night and day, scale it yourself etc. All things which the new [Serverless](http://blog.rowanudell.com/the-serverless-compute-manifesto/) architectures can take off your hands.  Ideally you'd be able to deploy a JAX-RS application as a FaaS application, allowing you to have 'the best of both worlds'. A simple to build and cost efficient application. It's important to note that here the term 'FaaS application' is used to refer to an application composed of multiple FaaS functions, this will have more meaning soon.

A framework has already been developed specifically for this purpose. An open source project developed by Björn Bilger, [JRestless](https://github.com/bbilger/jrestless) is an impressive tool that allows you to wrap up a Jersey application and deploy it as a serverless function using a variety of different platforms

JRestless initially targetted running on AWS Lambda but has since added OpenWhisk support, and through this PR [TODO -LINK!] now can run your JAX-RS apps on the [Fn Project](http://fnproject.io/) which was recently announced at JavaOne  in San Fransisco. 

## To The Code!

Please look at (clone, edit, make PRs for) our [Fn/JRestless example project](https://github.com/fnproject/fn-jrestless).

If you have seen JAX-RS code before then none of this will be a surprise:

```java
    public BloggingResource(@Context RuntimeContext context){
        database = setUpDatabase(context);
    }

    @GET
    @Path("/html")
    @Produces({MediaType.TEXT_HTML})
    public InputStream getWebPage() {
        return this.getClass().getResourceAsStream("/index.html");
    }

    @GET
    @Path("/blogs")
    @Produces({MediaType.APPLICATION_JSON})
    public List<BlogPost> getAllPosts() {
        return database.getAllPosts();
    }
	
    @POST
    @Path("/add")
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.TEXT_PLAIN)
    public String createPost(BlogPost post) {
        database.postData(post);
        return (post.getTitle()) + " added";
    }
```

Standard stuff from a JAX-RS perspective, but isn't it neat to have three separate functions defined in one place with shared DTO and database code? This is what we alluded to earlier: a FaaS Application - not just a standalone Function which does one stateless thing in isolation but a coherent application built on a more flexible foundation without sacrificing its integritry.

The only extra code you need is a class acting as an entrypoint to your function, which defines which classes Jersey should consider as part of the app:

```java
import com.jrestless.fnproject.FnFeature;
import com.jrestless.fnproject.FnRequestHandler;
import org.glassfish.jersey.server.ResourceConfig;

public class BloggingApp extends FnRequestHandler {

    public BloggingApp() {

        ResourceConfig config = new ResourceConfig();
        config.packages(getClass().getPackage().getName());  // scan all classes in this package
        config.register(FnFeature.class);

        init(config);

        start();
    }
}
```

Extending `FnRequestHandler` means we can specify the entrypoint in our `func.yaml` like this:

```yaml
cmd: com.example.fnjrestless.blog.BloggingApp::handleRequest
```


## Under the hood

In response to the *first* request, JRestless initialises Jersey with the `ResourceConfig` provided (`ResourceConfig` extends `Application` which [defines a whole JAX-RS app](https://docs.oracle.com/javaee/7/api/javax/ws/rs/core/Application.html)). We recommend you use Hot Functions to amortise the cost of initialising Jersey.

Then JRestless provides a mapping from Fn to Jersey so that from the point of view of your app it is just serving regular HTTP requests and returning responses as usual. Feel free to use [MessageBodyWriters](https://docs.oracle.com/javaee/7/api/javax/ws/rs/ext/MessageBodyWriter.html), [ExceptionMappers](https://docs.oracle.com/javaee/7/api/javax/ws/rs/ext/ExceptionMapper.html), etc.

![JRestless flow]({{ "assets/jrestless-flow.png" | absolute_url }})

## Summary

FaaS Applications can be difficult. A FaaS function is normally a single entity that doesn’t rely on any external state so composing these functions in ways to make coherent applications is challenging.

The stateless designs encouraged by JAX-RS mean that many applications fit very nicely into a FaaS. JRestless makes this easy to do, with minimal boilerplate.

For more detail, please see the [example app](https://github.com/fnproject/fn-jrestless#fn-project-jrestless-blogging-example) which details build and deployment of an app onto an Fn server.
