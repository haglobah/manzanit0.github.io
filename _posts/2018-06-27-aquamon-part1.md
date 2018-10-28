---
layout: post
title: "Interacting with Salesforce's Tooling API"
author: Javier Garcia
description: "An application to refresh Salesforce sandboxes."
category: salesforce
apprenticeship: false
tags: salesforce, .NET, automation
---

Since I started working at Frontiers, we've had many technical tasks that were monotonous and repetitive, typical in most technical jobs. One of them was refreshing our Salesforce Sandboxes every time we wanted to start working on a new feature.

Our *modus operandi* usually is to first refresh a sandbox, copy from Production, then branch our repository's Test branch, push all the metadata to the sandbox, and the finally start working on it until it's finished and a pull request is made.

If you've worked for some time with Salesforce, you are probably aware of how tedious it is to refresh a sandbox from Production and, once it is finished, set all the environment variables in Custom Settings or Custom Metadata. Here is where **Aquamon** comes into play.

After having to do this for a couple of months, I came to the conclusion that this had to be automated. To achieve this, I made a small CLI application which simply connected to Salesforce via the Tooling API and sent a refresh process order with a [PostSandboxCopy](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_interface_System_SandboxPostCopy.htm) execution script being executed after.

The application is separated in 4 different modules: the first one and most simple is the post-copy script, developed in Apex, which lives in Salesforce. This is executed every time a Sandbox is refreshed and simply sets some default values to the custom settings, remote site settings, etc. It can also seed the DB to have some records to work with without having to manually create them.

The post-copy script can look something like this, from the docs:

```csharp
global class PrepareMySandbox implements SandboxPostCopy {
    global void runApexClass(SandboxContext context) {
        System.debug("Org ID: " + context.organizationId());
        System.debug("Sandbox ID: " + context.sandboxId());
        System.debug("Sandbox Name: " + context.sandboxName());

        // Insert logic here to prepare the sandbox for use.
    }
}
```

The rest of the application, I developed in C# with .NET core. It has 3 modules: one for configuration, another one which manages all the connections/requests with the Salesforce Tooling API and a third for the CLI. Since I don't want to make this post extremely long, I'll dig deeper into the Tooling client and skip quickly over the others.

While the Tooling API is not specially complicated to use, there are some perks to it. It has a REST API and a SOAP API, but since we can do everything we need to with the REST API, I took that path.

## Authenticating in Salesforce

Now, first is first, and to be able talk to the Salesforce APIs we first need to authenticate. In order to do this, I created a DTO which contains all the authentication data and a small client to do so.

Salesforce's APIs use OAuth, so we only need to POST to their auth URL and get the authentication token which we will later use in all future calls: `https://login.salesforce.com/services/oauth2/token`

```csharp
public class ForceClient
{
    public string AuthToken { get; set; }
    public string ServiceUrl { get; set; }

    public void Login(LoginInfo info)
    {
        HttpContent content = new FormUrlEncodedContent(new Dictionary<string, string>
            {
                {"grant_type", "password"},
                {"client_id", info.ClientId},
                {"client_secret", info.ClientSecret},
                {"username", info.Username},
                {"password", info.Password + info.SecurityToken}
            }
        );

        using (var client = new HttpClient())
        {
            var message = client.PostAsync(new Uri("https://login.salesforce.com/services/oauth2/token"), content)
                .Result;

            if (!message.IsSuccessStatusCode)
            {
                throw new AuthenticationException();
            }

            var responseString = message.Content.ReadAsStringAsync().Result;

            var obj = JObject.Parse(responseString);
            AuthToken = (string) obj["access_token"];
            ServiceUrl = (string) obj["instance_url"];

            Console.WriteLine($"\n:: Logged in {ServiceUrl} ::");
        }
    }
}
```

## Connecting to the Tooling API

Once we're authenticated, in order to create, refresh and poll sandboxes, Salesforce exposes two different entities: the [SandboxInfo](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_sandboxinfo.htm) and the [SandboxProcess](https://developer.salesforce.com/docs/atlas.en-us.api_tooling.meta/api_tooling/tooling_api_objects_sandboxprocess.htm). To create or refresh sandboxes we would have to create a SandboxInfo record, to check the status, we would poll the SandboxProcess. 

All our development is going to oscillate between these two entities: do we want a new sandbox? Post a SandboxInfo. Do we want to refresh it? Patch it. Do we want to get the status? Then Get it.

To achieve this, I simply created a DTO for the SandboxInfo, based on the documentation and a SandboxManager class which would make the different calls to the different endpoints. If you notice, we inject the previous ForceClient which has already authenticated us.

```csharp
public class SandboxInfo
{
    public SandboxInfo()
    {
        AutoActivate = true;
        LicenseType = "DEVELOPER";
        Description = "Sandbox created via API";
    }

    public string SandboxName { get; set; }
    public string Status { get; set; }
    public string Description { get; set; }
    public string LicenseType { get; set; }
    public string ApexClassId { get; set; }
    public bool AutoActivate { get; set; }

    public string ToJSON()
    {
        return $@"
        {
            ""AutoActivate"": {AutoActivate.ToString().ToLower()},
            ""SandboxName"": ""{SandboxName}"",
            ""Description"": ""{Description}"",
            ""LicenseType"": ""{LicenseType}"",
            ""ApexClassId"": ""{ApexClassId}""
        }";
    }
}
```

```csharp
public class SandboxManager
{
    private const string SANDBOX_INFO_ENDPOINT = 
        "/services/data/v42.0/tooling/sobjects/SandboxInfo/";

    private const string SANDBOX_PROCESS_ENDPOINT = 
        "/services/data/v42.0/tooling/sobjects/SandboxProcess/";

    private const string QUERY_SANDBOX_INFO =
        "/services/data/v42.0/tooling/query?q=SELECT+Id,SandboxName+FROM+SandboxInfo+WHERE+SandboxName+in+('{name}')";

    private const string QUERY_SANDBOX_PROCESS =
        "/services/data/v42.0/tooling/query?q=SELECT+Id,Status+FROM+SandboxProcess+WHERE+SandboxName+in+('{name}')";

    public SandboxManager(ForceClient client)
    {
        Client = client;
    }

    private ForceClient Client { get; }

    public void CreateSandbox(SandboxInfo info)
    {
        var response = Client.Post(SANDBOX_INFO_ENDPOINT, info.ToJSON());
        var stringResponse = response.Content.ReadAsStringAsync().Result;

        Console.WriteLine("\n:: Sandbox created :: \n" + stringResponse);
    }

    public void RefreshSandbox(SandboxInfo info)
    {
        var response = Client.Get(QUERY_SANDBOX_INFO.Replace("{name}", info.SandboxName));
        var stringResponse = response.Content.ReadAsStringAsync().Result;

        Console.WriteLine("\n:: Queried Sandbox :: \n" + stringResponse);

        dynamic queriedSandbox = JObject.Parse(stringResponse);
        string id = queriedSandbox.records[0].Id;

        response = Client.Patch(SANDBOX_INFO_ENDPOINT + $"{id}/", info.ToJSON());
        stringResponse = response.Content.ReadAsStringAsync().Result;

        Console.WriteLine("\n:: Refreshed Sandbox :: \n" + stringResponse);
    }

    public string GetSandboxStatus(SandboxInfo info)
    {
        var response = Client.Get(QUERY_SANDBOX_PROCESS.Replace("{name}", info.SandboxName));
        var stringResponse = response.Content.ReadAsStringAsync().Result;

        dynamic sandboxes = JObject.Parse(stringResponse);
        foreach (var record in sandboxes.records)
            if (record.Status == info.Status)
                return Client.Get("" + record.attributes.url).Content.ReadAsStringAsync().Result;

        return $@"{""Message"": ""A sandbox with the name {info.SandboxName} and status {
                info.Status
            } does not exist.""}";
    }
}
```

Just as a heads up, I added a couple of extra methods to `ForceClient.cs` in order to encapsulate all the `HttpClient` instances.

``` csharp
public class ForceClient
{
    // ... Further methods in the Force client.
    
    public HttpResponseMessage Get(string endpoint)
    {
        return Callout(HttpMethod.Get, endpoint, "{}");
    }

    public HttpResponseMessage Post(string endpoint, string payload)
    {
        return Callout(HttpMethod.Post, endpoint, payload);
    }

    public HttpResponseMessage Patch(string endpoint, string payload)
    {
        return Callout(new HttpMethod("PATCH"), endpoint, payload);
    }

    private HttpResponseMessage Callout(HttpMethod method, string endpoint, string payload)
    {
        using (var client = new HttpClient())
        {
            var request = new HttpRequestMessage(method, new Uri(ServiceUrl + endpoint));

            request.Headers.Add("Authorization", "Bearer " + AuthToken);
            request.Headers.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json"));
            request.Headers.Add("X-PrettyPrint", "1");

            request.Content = new StringContent(payload, Encoding.UTF8, "application/json");

            return client.SendAsync(request).Result;
        }
    }
}
```
    
## What about the rest of the application?

Now, all the latter summarizes the interaction with Salesforce. To wire that up in a CLI application, I used the out-of-the-box `Microsoft.Extensions.Configuration` to read the configuration and bind it to different POCOs and `Microsoft.Extensions.CommandLineUtils` to set up the commands and argument parsing.

If you want to see the whole code, please do visit the github repository [here](https://github.com/Manzanit0/Aquamon).

On a side note, `Microsoft.Extensions.Configuration` is no longer being actively developed, so if I had to re-develop the application I would maybe go for a different option, but if you're interested in it, Nate McMaster has forked it [here](https://github.com/natemcmaster/CommandLineUtils), which is nice.

## TL;DR;

As you have seen, interacting with the Salesforce APIs is not very difficult, but it does require getting to know them. At the start it took me a little time to figure out how to query the system, and some other perks related to it like putting together the actual string. Nonetheless, once you get it rolling, it's very easy to keep pumping more stuff out.

Furthermore, I encourage you guys to download it, test it and feel free to give me any feedback! It's a tool I currently use on a daily basis and even though it is still not perfect, it does save me a bunch of time.