# Introduction to API Development

## Conventions

This document includes references to interface and class names. These names are given without their respective namespaces for brevity. IDEs typically will discover these classes automatically and offer to include the appropriate `using` directives.

Most of the sample code is designed to be used in a `IApiProvider.Build` method. For space reasons, some of the samples omit the class and method context. 

# API plugins

The API Server is extended using plugins. Any change to the functionality must be defined in plugin DLLs that are loaded when the API Server starts up.

An example plugin assembly is provided in the [SamplePlugin folder](../SamplePlugin).

# API Server Start

As the API Server starts up, it goes through the following phases.

- **ASP.NET Core Initialization**. The basic services of ASP.NET Core are configured in this phase. Plugins implementing `IBuilderPlugIn` are run.
- **API initialization**. Once the ASP.NET Core app is configured, services are added to the app. Use the `IPlugIn` interface to add API plugins.
- **ASP.NET Core Routing initialization**. In this phase, all available API projects are discovered along with their associated API methods. To add API projects and methods, use the `IPlugInMethodSetProvider` interface. To add API methods, use the `IApiBuilder` interface.
- **API validation**. The API is checked for consistency and errors. If an API project contains errors, then the server will still start and respond to requests, but any requests to the failing API projects will result in a 404 error.
- **Routing table generation**. The API Server configures an ASP.NET COre `EndpointRoutingMiddleware` and builds the routing table according to the discovered API methods.
- **Start**: The server starts responding to requests.

# API providers

## Declaring an API provider

Declare an API provider by adding a class to the API plugin. The easiest way to add an API provider to an API project is to implement the project-specific interface, such as `IApiProviderFor<PortalApiProject>`. When using this approach, the class must be marked as `public` and it must have a public parameterless constructor, so that the API builder can create instances of the class.

``` csharp
public class HelloWorldApi : IApiProviderFor<PortalApiProject>
{
    public void Build(IApiBuilder builder)
    {
        builder.AddMethod(Method.Define("helloworld")
            .HandleGet(request => "Hello world!")
        );
    }
}
```

## Dependency injection

While ASP.NET Core provides a dependency injection container which is fully available for use, the API Server provides its own dependency injection mechanism.

Services are resolved by their type `T` and by calling the `Resolve<T>` method on the appropriate dependency injection container (which implements the `IResolve` interface).

- _Resolving services at build time_: Server-level services are resolved by calling the `IApiBuilder.Resolve<T>` method. These services can be used for the lifetime of the server.
- _Resolving services at request time_: Request-level services are resolved by calling the `IRequest.Resolver.Resolve<T>` method. These services should only be used for the lifetime of one request.
- _Registering services at build time_: To register a new service, resolve the `IServices` object and register the new service on it:

```csharp
public void Build(IApiBuilder builder)
{
    var myService = new MyServiceImplementation();
    builder.Resolver.Resolve<IServices>().Register<IMyService>(myService);
}
```

## Order of execution

You can influence the order in which the API providers are called. You can add the `ApiProviderSort` attribute to the API provider class, or to the plugin assembly. All plugin assemblies and API providers are sorted by their `ApiProviderSort` attribute value. If no attribute is declared, the value defaults to 0.

You may need to set the order if you add services in one API provider, then resolve the service in another API provider:

``` csharp
// Use a higher sort order to ensure this API provider is called after IMyService is registered.
[ApiProviderSort(SortOrder = 10)]
public class ApiProvider2 : ApiProvider
{
    public void Build(IApiBuilder builder) 
    {
        var myService = builder.Resolver.Resolve<IMyService>();
        // ... then use the service
    }
}
```

# API methods

*can be of different types*
*can define many routes*, for example with entity methods a consistent route schema is applied
*routes of a method share a common URL prefix*
*can be assigned to one or more API projects*

## Method.Define

Methods are added to the API by adding classes that implement the `IApiProvider` interface, and adding API methods to the API builder:

```csharp
public void Build(IApiBuilder builder)
{
    builder.AddMethod(
        Method.Define("helloworld")
        .HandleGet(request => "Hello world")
    );
}
```

In this case, "helloworld" is the URL of the API method, but an API method is always called in the context of an API project.

For more information about the valid URL syntax, please refer to this page: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing?view=aspnetcore-8.0#route-templates

## How a request to an API method is processed

When the API Server receives a request, it goes throw the following steps to process it.

1. ASP.NET Core finds the API route that best matches the request URI.
1. If the server is in the Unavailable state, the request is aborted. (This is typically the case when a software update is about to be applied.)
1. The API Server associates the request with a session group. 
It tries to find an existing session group in the database (see: Session state), or it creates a new session group if it cannot find one.
1. The XSRF protection token header is checked. If the token does not match, the request is rejected.
1. The user's preferred language settings are determined, then applied to the asynchronous call context, All subsequent code will automatically use the correct language settings.
1. The validators for the API method are run. If any validation errors occur, the request is rejected.
1. The API method handler code runs.

Note that this list is not exhaustive, and additional plugins can hook into the lifecycle of the request.
