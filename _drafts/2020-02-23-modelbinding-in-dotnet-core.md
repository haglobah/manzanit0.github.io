---
layout: post
title: ".NET Core: Model binding complex types"
author: Javier García
category: dotnet
tags: dotnet, c#
---

> A complex type must have a public default constructor and public writable
> properties to bind. When model binding occurs, the class is instantiated
> using the public default constructor.

Prefixes are the property name, say `instructor.ID`, `instructor` would be
the prefix.

> For each property of the complex type, model binding looks through the
> sources for the name pattern prefix.property_name. If nothing is found, it
> looks for just property_name without the prefix.

You can, however, specify a custom prefix with the attribute:
`[Bind(Prefix = "custom_prefix")]`

[0]: https://docs.microsoft.com/en-us/aspnet/core/mvc/models/model-binding?view=aspnetcore-3.1
