---
layout: post
title: "Simple HTTP API server with Jersey (+ Jetty)"
date: 2018-10-03 19:10:00 +0900
categories: sw
description: rest, server, jersey, jetty
---

Early in this year, I was working on developing a distributed system. (many things happened...) Apparently, the need for small service components which provide HTTP API endpoints arose. The most crucial part was choosing a lightweight and simple Web Services framework. At that time, I was sick of **Spring** and one of my co-workers was trying **Spring Boot**. Then, while browsing other solutions, I bumped into **[Jersey Web Services framework][Jersey Main]** which provided handy RESTful features.

Okay, the forehead was quite verbose. **[Jersey][Jersey Main]** is the main topic for this article and I'm going to share code snippets about it.

The most fascinating part for me is that it follows the **[JAX-RS][JAX-RS]** specification. If you has an experience in **Spring**, you may already know how convenient Annotation-based coding is to write APIs and business logics.

There are multiple available choices for a web container because **[Jersey][Jersey Main]** is the implementation of the official standard(!). My choice was **[Jetty][Jetty main]** which is well-known and lightweight.  
(Sometimes, several libraries in a single build have dependencies with the same artifact name but different versions. It also happened to me. I just carried on even though it would be perfectly okay to use another web container.)

*The [full requirement of RESTful][REST Architectural Constraints] is quite strict and, apparently, sometimes overkill. This is the reason I titled this post not 'RESTful API' but 'HTTP API'.*

*This article is based on Jersey 2.25.1 and Jetty 9.2.22.*

# Setting Servlet and Starting Server
The following code shows how to set a **Servlet** and a **Server**.  
It sounds a bit complicated, the **Servlet** is configured by **Resources**, **a Binder**, and **Exception Mappers** in the example. Each of them will be explained later.
In this case, my web container -**Jetty**- has only one servlet.
~~~ java
import org.eclipse.jetty.servlet.ServletContextHandler;
import org.eclipse.jetty.servlet.ServletHolder;
import org.glassfish.jersey.server.ResourceConfig;
import org.glassfish.jersey.servlet.ServletContainer;

public class MyServer {
    public static void main(String ...args) {
        // Configure resources, binder, exception mappers
        ResourceConfig config = new ResourceConfig();
        // Set the name of a package which contains classes annotated by 'Path'.
        config.packages("com.junsoh.server.resource"); 
        // Register a dependency binder to 'inject' user defined objects to resources.
        // Note that classes Dependency1 and Dependency2 are created by me.
        config.register(new DependencyBinder(new Dependency1(), new Dependency2()));
        // Register exception mappers.
        config.register(GenericExceptionMapper.class);
        config.register(BadRequestExceedsMapper.class);

        // Create a servlet with the configuration.
        ServletHolder servlet = new ServletHolder(new ServletContainer(config));

        // Prepare Jetty server.
        Server server = new Server(8888);
        ServletContextHandler context = new ServletContextHandler(server, "/*");
        context.addServlet(servlet, "/api/*");

        // Ignite
        server.start();
        server.join();
    }
}
~~~

# API Component
This is the simplest resource class which implements very simple API. If the API <code>/api/hello</code> is called, it returns the string with the content type <code>text/plain</code>.
~~~ java
@Path("/api")
public class ApiService {
    @GET
    @Path("hello")
    @Produces(MediaType.TEXT_PLAIN)
    public String getHelloWorld() {
        return "Hello, world!";
    }
}
~~~

The following code looks more realistic. Each API method is configured by its annotations. <code>Get</code> or <code>Post</code>, <code>Path</code> are quite intuitive. Also, as we define the prototype of a method, we could get different types of parameters and return the value of a type what we want to.  
In the real world, we need to interact with other instances which contains deeper business logic. <code>dependency1</code> and <code>dependency2</code> are the instances injected from outside of Jersey framework by **Binder**. Let's talk about it later.  
**Jersey** also provides very easy way to render a data object into a certain content type (such as JSON or XML) by using the annotation <code>Produce</code>.
~~~ java
import javax.inject.Inject;
import javax.servlet.http.HttpServletRequest;
import javax.ws.rs.*;
import javax.ws.rs.core.Context;
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;

@Path("/api")
public class ApiService {
    @Inject
    private Dependency1 dependency1;

    @Inject
    private Dependency2 dependency2;

    @GET
    @Path("hello")
    @Produces(MediaType.TEXT_PLAIN)
    public String getHelloWorld() {
        return "Hello, world!";
    }

    @GET
    @Path("test/{param}")
    @Produces(MediaType.APPLICATION_JSON)
    public Response testApi(
            @Context HttpServletRequest request,
            @PathParam("param") String parameter1
            @QueryParam("param") String parameter2)
            throws IOException, BadRequestExceedsException {
        try {
            TestResponseVO result =
                dependency1.doSomething(
                    parameter,
                    dependency2.doSomething(parameter2));

            return Response.status(200).entity(result).build();
        } catch (IOException e) {
            throw e;
        } catch (Exception e) {
            throw new BadRequestExceedsException("Something went wrong.");
        }
    }
}
~~~

# ExceptionMapper
The method <code>testApi</code> can throw <code>Exceptions</code>. When it happens, Jersey maps the exception to the most relevant <code>Response</code>.
~~~ java
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.ExceptionMapper;
import javax.ws.rs.ext.Provider;

@Provider
public class GenericExceptionMapper implements ExceptionMapper<Throwable> {
    @Override
    public Response toResponse(Throwable e) {
        return Response.status(Response.Status.INTERNAL_SERVER_ERROR)
                .entity(new Error(e.getMessage()))
                .type(MediaType.APPLICATION_JSON)
                .build();
    }
}
~~~

The above one was for any Exception. If you want to specify a certain Exception? It's easy.
~~~ java
import javax.ws.rs.core.MediaType;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.ExceptionMapper;
import javax.ws.rs.ext.Provider;

@Provider
public class BadRequestExceedsMapper implements ExceptionMapper<BadRequestExceedsException> {
    @Override
    public Response toResponse(BadRequestExceedsException e) {
        return Response.status(Response.Status.SERVICE_UNAVAILABLE)
                .entity(new Error(e.getMessage()))
                .type(MediaType.APPLICATION_JSON)
                .build();
    }
}
~~~ 

# Dependency Binding
As I mentioned, it's needed to inject instances which contain business logic to API components. Here, the **Binder** which is called **HK2** is used for injection. As if Spring does, HK2 provides Inversion of Control(IoC) and dependency injection(DI). In the following code, it maps the classes to the instances directly. But, it provides more way of injection, and they are still quite straightforward.
~~~ java
import org.glassfish.hk2.utilities.binding.AbstractBinder;

public class DependencyBinder extends AbstractBinder {
    private Dependency1 dependency1;
    private Dependency2 dependency2;

    public DependencyBinder(Dependency1 dependency1, Dependency2 dependency2) {
        this.dependency1 = dependency1;
        this.dependency2 = dependency2;
    }

    @Override
    protected void configure() {
        bind(dependency1).to(Dependency1.class);
        bind(dependency2).to(Dependency2.class);
    }
}
~~~

# Conclusion
This article only deals with the very essential features to implement a simple HTTP API server. But, **JAX-RS** specifies various features, and even **Jersey** provides more features than the specification. Its compliance with **JAX-RS specification** allows developers to plug-in various components. Once a best practice is written, it can be re-used repeatedly. Consequently, it will lead a productive way of working.


[Jersey main]: https://jersey.github.io/ "Jersey main"
[JAX-RS]: https://github.com/jax-rs "JAX-RS"
[Jetty main]: https://www.eclipse.org/jetty/ "Jetty main"
[REST Architectural Constraints]: https://restfulapi.net/rest-architectural-constraints/ "REST Architectural Constraints"