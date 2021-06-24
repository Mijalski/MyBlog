+++
title = "Auto Registering dependencies with Scrutor"
date = "2021-06-24T14:26:47+02:00"
author = ""
authorTwitter = "kyrcooler"
cover = ""
tags = ["scrutor", "dependency-injection"]
keywords = ["scrutor", "transient dependency", "dependency injection", "di", "dependency injection container", "service collection", "services", "registering dependencies"]
description = ""
showFullContent = false
+++

![Dependency injection](https://i.imgur.com/HXCKL5l.png)

Dependency injection (DI) software design pattern is one of the most used ones and requires no introduction, but if you need any [check out MDNS article about this topic](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection). 

To achieve Inversion of Control we can use DI Containers, in which we define services and the abstractions that they fulfill so that they can be injected (created) at runtime. .NET Core and .NET5+ has a simple container built-in that allows registering dependencies one by one, but the application can quickly outgrow this approach, plus it creates an additional responsibility of remembering to register our new service each time we create one.

Here's an example of registering dependencies one by one, which is fine up to a certain point.

{{< highlight csharp >}}

services.AddTransient<IMyService, MyService>()
                .AddTransient<IOtherService, OtherService>()
                .AddTransient<IExampleService, ExampleService>();

{{< / highlight >}}

Developers tend to lean towards other containers, that support registering by convention (by some rules, eg. by name), but they can create an unnecessary complexity (some even abstract from IServiceCollection) and they all seem to have a steep learning curve.

How can you register all your dependencies without writing lines and lines of code? [Scrutor](https://www.nuget.org/packages/Scrutor/)! With it, we can scan our assemblies and perform certain actions on the types we find - it's a perfect tool to find all our classess and interfaces and register them by convention (the convention being marker interfaces).

Add the package to your project and then  create three interfaces to mark our dependencies with, like so:

{{< highlight csharp >}}

public interface ISingletonDependency { }

public interface ITransientDependency { }

public interface IScopedDependency { }

{{< / highlight >}}

If you need some information on the different type of lifetimes, [check the MDNS for more information about this topic](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/dependency-injection?view=aspnetcore-5.0#lifetime-and-registration-options).

Decorate those marker interfaces with comments describing that they are responsible for registering dependencies implementing them. You may want to discuss the naming with your team and agree to drop the `Dependency` suffix or name them with Domain in mind (eg. `IUseCase` or `IService`).

After that, we need all our services to implement the selected service (`ISingletonDependency` for our singletons, `ITransientDependency` for our transient services and `IScopedDependency` for scoped ones).

{{< highlight csharp >}}

public class MyService : IMyService, ITransientDependency 
{
    // ...
}
{{< / highlight >}}

Now we need to do the actual injection of our services, so in our `Startup.cs` navigate to `ConfigureServices` method (or create a new extensions class named something like `DependencyInjectionExtensions.cs` to store our setup separately):

{{< highlight csharp >}}

// Dependency Injection with Scrutor
services.Scan(scan => scan
    .FromAssemblyDependencies(Assembly.GetExecutingAssembly())
    .AddClasses(classes => 
        classes.AssignableTo<ITransientDependency>())
    .AsImplementedInterfaces()
    .WithTransientLifetime());

services.Scan(scan => scan
    .FromAssemblyDependencies(Assembly.GetExecutingAssembly())
    .AddClasses(classes => 
        classes.AssignableTo<ISingletonDependency>())
    .AsImplementedInterfaces()
    .WithSingletonLifetime());

services.Scan(scan => scan
    .FromAssemblyDependencies(Assembly.GetExecutingAssembly())
    .AddClasses(classes => 
        classes.AssignableTo<IScopedDependency>())
    .AsImplementedInterfaces()
    .WithScopedLifetime());

{{< / highlight >}}

We scan all our executing assemblies for classes assignable to our three marker interfaces and register them as implementations of those interfaces (with proper scopes set by each of them). 

Now each time we create a new service we don't need to navigate to a `Startup` class and register our service by hand, but we can do it automatically with one interface. If we have a base class for our services, we can implement it there and don't need to worry about it later, just like so:

{{< highlight csharp >}}

public abstract class ServiceBase : ITransientDependency 
{
    // ...
}

{{< / highlight >}}