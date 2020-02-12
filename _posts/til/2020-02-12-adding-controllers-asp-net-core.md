---
layout: post
title: "Telling ASP.NET Core to map the controllers to actual endpoints"
author: Javier Garcia
category: .NET
tags: dotnet, c#, controllers
---

TIL that to add controllers to an ASP.NET Core application, it's not as straight forward as simply creating
a controller which inherits from `Controller` or even adding the `Route` attribute to it. It's also necessary
to explicitely tell the framework to map the controllers to endpoints!

In the `Startup.cs` class, you'll need to add the following:

```csharp
public void ConfigureServices(IServiceCollection services)
{
    // ...

    services.AddControllers();
}
```

As well as:

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // ...

    app.UseRouting();
    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
    });
}
```

Of course, don't forget to mark the Controller:

```csharp
[Route("tsunami")]
public class TsunamiController : Controller
{
    // ...
}
```

