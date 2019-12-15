---
layout: post
title: ".NET Core: Bootstraping an application in Heroku"
author: Javier García
category: .NET
tags: dotnet, asp, docker, heroku, devops
---

Recently I had a client which requested the development of an MVP for his personal business. Since
the application was not going to go live in the short term I figured Heroku would be a good platform
to start the development in. I find it allows me to skip all the ceremony that other clouds like
Azure or AWS need. The issue is, while Heroku is usually very well documented, it doesn't have much
information for .NET applications. My intention is to write this post as some aid to that.

**Note:** I will be using **.NET Core 3.0** for this blogpost

## Creating the most basic ASP.NET Core example

Whenever I start a new project, one of my obssesions is to try to start with the single most barebones
structure I can. It makes me uneasy to start off with a lot of scaffolded code which you don't
understand and/or potentially don't need.

After checking out the different offerings that ASP.NET Core has, I figured that going with a [Razor Pages][0]
project could be a good decision, specially since the client won't need to expose any kind of public
API, and the flows won't be too complex.

To get started, I went ahead and ran:

```bash
dotnet new webapp -o lazarus
cd lazarus
```

Once the main application is bootstraped, to run the tests, it's as easy as `dotnet test` and to run
the application, go with `dotnet run`. I don't want to get too crazy with .NET because the point of
the blog post is to introduce how I automated the build, deployed to Heroku and the different devop-sy
aspects, so I'll skip towards that.

Just as a very generic introduction, you'll find that `Pages/` has the different Razor Pages which
.NET will render, and the entry point of the application is in `Program.cs`. For more information, check
the [official tutorials][1].

## Adding Docker to the mix

Moving forward, to have any .NET Core application run on Heroku, we need to first run them in a container.
We'll be working with Heroku's [container registry and runtime][2]. For this, cutting most corners,
I went ahead and copied a `Dockerfile` from [Dockerhub][3].

After adapting it a little bit, I ended up with the following:

```Dockerfile
FROM mcr.microsoft.com/dotnet/core/sdk:3.0 AS build-env
WORKDIR /app

# Copy csproj and restore as distinct layers
COPY lazarus/*.csproj ./
RUN dotnet restore

# Copy everything else and build
COPY lazarus/ ./
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/core/aspnet:3.0 AS runtime
WORKDIR /app
COPY --from=build-env /app/out ./

# Set ASPNETCORE_URLS to run the app on the port Heroku exposes.
# Kestrel run by default on 5000/1 and Heroku doesn't allow that.
CMD ASPNETCORE_URLS=http://*:$PORT dotnet lazarus.dll
```

As you'll notice it's a multi-step Docker image, where the first step is the building of the
application, and the second step is simply putting that together into an image that simply contains
the runtime, not the SDK.

Now, even though it's commented, I'll be even more explicit and further explain the container's entry point.
With the out-of-the-box generated ASP project, Kestrel runs on ports 5000 for HTTP and 5001 for HTTPS.
The issue is that Heroku can't guarantee which port it will expose in the dyno once it spins it up -
it's something that we can only know at runtime via [the `$PORT` environment variable][4].
Hence, the reason for starting up the application on the port exposed via `$PORT`.

Now, this is a Heroku quirk. If you are to run the application on any other platform, the most likely
thing is that you will be able to use whatever port you wish. But this is important for this scenario.

To start the docker container locally, we would just run `docker run` like always, but explicitly
stating the port as an env variable:

```bash
docker build -t lazarus .
docker run -p 5000:5000 -e PORT=5000 lazarus
```

So far, so good - we have an ASP Core application which runs, it's wrapped in a Docker container and
it all seems breezy.

## Deploying to Heroku

The next step in our story would be to deploy our app to Heroku, and I won't lie to you folks,
it's as easy as it gets. Create an account, then create an application and run the following
commands in the shell:

```bash
heroku container:login
heroku container:push web
heroku container:release web
heroku open
```

## Automating all things ops

As you may have realised, with the current setup, as we further develop our application, every time we
want to deploy a new version we would just have to run:

```bash'
heroku container:push web
heroku container:release web
```

It's not too bad, but we might want to further automate the deployment in our CI pipeline. To do
this, I decided to try out [Github Actions][8], and it worked out just fine. To get aquainted with Actions
I went ahead and created an initial job to build and test the application every time I pushed to the
remote repository. This looked like this:

```yaml
name: Build and Test

on: [push]

jobs:
  build_and_test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1
    - name: Setup .NET Core
      uses: actions/setup-dotnet@v1
      with:
        dotnet-version: 3.0.100
    - name: Build with dotnet
      run: dotnet build ./lazarus/lazarus.csproj --configuration Release
    - name: Test with dotnet
      run: dotnet test ./lazarusTest/lazarusTest.csproj --configuration Release
```

I found it to be fairly straightforward, so I created another Action for deployment everytime I merged
some code into `master`. For this I reused an already created action to [interact with Heroku][5].
Using that, my own YAML looked like this:

```yaml
name: Release to Heroku

on:
  push:
    branches:
      - master

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@master
      - name: login
        uses: actions/heroku@master
        env:
          HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
        with:
          args: container:login
      - name: push
        uses: actions/heroku@master
        env:
          HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
        with:
          args: container:push -a ${{ secrets.HEROKU_APP }} web
      - name: release
        uses: actions/heroku@master
        env:
          HEROKU_API_KEY: ${{ secrets.HEROKU_API_KEY }}
        with:
          args: container:release -a ${{ secrets.HEROKU_APP }} web
```

In this step there is the quirk where you have to configure the secrets (that's a Heroku API key
and the app name) through `Settings/Secrets` in the actual project's settings. It's the easier step,
but I had a little trouble figuring out where it was. It's the repository's settings, not the account's.
Lastly, to generate the API key, check [here][6].

With this, we now have two actions configured which
  1. Build and test every commit we push
  2. Deploy to Heroku every new version we merge to `master`


## Easter Egg: Running migrations

Lastly, I didn't plan to include how to automate migrations in this blogpost, but after a while
I figured it would make it a little bit more complete. Running migrations as an automated step if
our DB doesn't have distributed access, makes our development workflow so much more simple and in
.NET Core, it's not all that complicated.

When deciding where to automate that part of the process, I wasn't sure - usually I would expect
migrations to run as part of the deployment process, just before starting up the application, but
in this case, I decided that the most simple solution would be to include them within the actual
startup of the application, specially taking into consideration that the application would be a single
instance and we would also have a single database without replicas.

### Working with migrations

To get started with migrations in .NET Core, I highly recommend you pick [this tutorial][7] up and then
come back to the blog post to continue with the automation parts. I will assume you already have some
migrations created and a working connection with your database.

### Running migrations before starting the server

To run all pending migrations before starting the web server, I went ahead and created a
`HostExtensions` which would, for the current WebHost, pick the Database service run the migration, and
return the host so the startup code can then spin the server up. This lives in the `Program.cs` and it
looks like this:

```cs
public static class HostExtensions
{
    public static IHost MigrateDatabase<T>(this IHost host) where T : DbContext
    {
        using(var scope = host.Services.CreateScope())
        {
            var services = scope.ServiceProvider;
            try
            {
                var db = services.GetRequiredService<T>();
                db.Database.Migrate();
            }
            catch (Exception ex)
            {
                var logger = services.GetRequiredService<ILogger<Program>>();
                logger.LogError(ex, "An error occurred while migrating the database.");
            }
        }
        return host;
    }
}
```

Where our Main function would use it like this:

```cs
public static void Main(string[] args)
{
    CreateHostBuilder(args)
        .Build()
        .MigrateDatabase<ApplicationContext>()
        .Run();
}
```

And with this, my friends, we would finally have our Database migrations running automatically everytime
we run our application.

## Final thoughts

Wrapping up, I hope to have covered most of the aspects you will run into if you decide to start a new
application running in Heroku, from the little Heroku quirks, to the more broad questions like _how do
I automate this in my CI/CD pipeline?_. I have found this journey to be not painful at all, to my surprise
and I've been quite delighted to find .NET Core a very friendly technology to work with, specially compared
to .NET 4.5, which was my last contact with .NET.

If this were a different kind of project, maybe a distributed service part of a bigger picture or simply
with an expectation of higher usage, I would have gone differently about it, or maybe not. Regardless, I
still find it extremely useful to start most of my applications in Heroku, due to how easy it is to get started
and then move to more complicated setups as the project requires it. I always try to keep in mind the quote:
_"Simple things should be simple, complex things should be possible."_

Happy coding,
Javier


[0]: https://docs.microsoft.com/en-us/aspnet/core/razor-pages
[1]: https://docs.microsoft.com/en-gb/aspnet/core/getting-started
[2]: https://devcenter.heroku.com/articles/container-registry-and-runtime
[3]: https://hub.docker.com/_/microsoft-dotnet-core
[4]: https://devcenter.heroku.com/articles/dynos#common-runtime-networking
[5]: https://github.com/actions/heroku
[6]: https://help.heroku.com/PBGP6IDE/how-should-i-generate-an-api-key-that-allows-me-to-use-the-heroku-platform-api
[7]: https://docs.microsoft.com/en-us/aspnet/core/tutorials/razor-pages/model
[8]: https://github.com/features/actions
