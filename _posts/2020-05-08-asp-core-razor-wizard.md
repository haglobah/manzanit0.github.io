---
layout: post
title: ".NET Core: Building a server-rendered wizard"
author: Javier García
category: .net
tags: .net, asp, server-rendering, forms
---

As you can see based on my most recent posts, I've been focusing on some
ASP.NET Core development. Recently, one of the things I've been working on is
a snappy form with a wizard-like interface which would allow the users of an
application to input the information without being overwhelmed with the amount
data required.

Most of the times, to develop an interactive web interface, I would immediately
turn my head towards Javascript, potentially React, simply because it provides
what most stakeholders expect nowadays from an application: a snappy feel.
However, this time I decided to try my luck with a server-rendered approach.
Before jumping into explaining how I did it, I'll try to outline the benefits I
considered.

## A single mind space

For me, the main benefit of choosing server-rendered web interfaces with stacks
like .NET or Elixir is that whenever I process data back and forth from the
interface, I don't have to change from C# to Javascript, but even further, I
don't have to make the jump from _UI-land_ to _backend-land_. Everything lives
in the same place. I may be using [view models][1], but I'm still in the
backend, meaning I have access to the database, to my services... or to
anything that lives in the backend. Being able to swap from the UI to the
server code without having to do the context switch enables me with a
productivity that otherwise I would not have.

## It's no longer an _API and API-Consumer_ architecture

Another of the main benefits I have found when going with a server-rendered
approach is that I don't necessarily have to go with creating a backend API and
a frontend interface which consumes the API. Sometimes, you just don't need
all the RESTful endpoints, nor the JSON serialising and de-serialising, dealing
with CORS, or many of the other inconvenients of the architecture. You just
need something SIMPLE.

In the particular case of the application I'm working on, I have discovered
that simply exposing a set of pages is more than enough. And then, maybe, if
tomorrow I do end up needing an API... I can always start working on it and
adapt my architecture as needed. Not yet though.It's all about evolving
architecture.

## Designing the solution

Now, being a sucker for simple tech and simple design, the MS offering that
made the most sense for this case was Razor Pages. You simply create a _Page_,
a _PageModel_ with some event handlers and done. No MVC, no SPA, nothing. Just
a simple page and some event handlers for it.

As an extra bonus, while playing around with Razor Pages, I discovered them to
be not just incredibly simple and easy to use but also extremely fast (although
it does make sense... simple is usually fast!). This is a quick snapshot of
how fast going back and forward through the original wizard was: ~120ms in
average, with form steps of over 10 fields and the full wizard being 6 steps.

![screenshot of razor pages rendering times](http://i.imgur.com/NFFMbx3.png)

As I started, I realised that the challenge of going with a completely
server-rendered wizard was that it was new - It was my first time implementing
something like this. The most important question I had to answer was **how to
have access to the wizard's data throughout the whole form**, regardless to
which section the user navigates to. In order words: every time the user clicks
the _next_ or _previous_ buttons and a request is made to the backend to load
the relevant page - where do I save the data so that it's available when he
comes back to that section or simply saves the full form?

The first thing I thought was to keep track of the data in a temporal database
table, every time I navigated back and forward through the wizard, persist the
changes... but that would make each request slower, because it has to go all
the way over to the database to fetch it, even if it's an extra 1ms. It also
has the inconvenience that it would be fairly tricky to handle the scenario
when the user simply closes the form - how would I clean up the data? I didn't
like it.

Another solution was to keep the data within the HTML, in hidden fields, and
send it back and forth all the time, certainly more appealing that the latter,
but there was still something that didn't click. Then I thought about cookies.
Ah, cookies! It was perfect - just update them every time the user goes back
and forth and reset them every time the user begins the form again. Also, the
requests to the server would simply have to contain the current step's data,
not everything, so there would be less data being sent over the wire, that is a
plus.

Reading about how to handle this in ASP, I found what is called
[`TempData`][2]. It's cookies, but handled from the server using the
[`set-cookie`][3] header in the response. Sold. Razor Pages with cookies it is.

## Creating the wizard

Having explained all my train of thought, we can dive right in the code without
too many explanations now.

_For those who just want a TLDR and check out the code, there is a full working
example right [here](https://github.com/Manzanit0/ServerWizardExample). We'll
be explaining it throughout this blogpost._

### The business layer

First of all the entity which we will be making this for is `Contact`, and the
wizard will attempt to cover some CRUD operations from the `ContactService`.
For simplicity, I've kept the business layer to the minimum:

```cs
public class Contact
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }
}

public class ContactService
{
    private IList<Contact> Contacts { get; set; }

    public ContactService()
    {
        Contacts = new List<Contact>();
    }

    public void Save(Contact contact) => Contacts.Add(contact);
    public Contact FindById(int id) => Contacts.FirstOrDefault(x => x.Id == id);
    public IList<Contact> All() => Contacts;
}
```

### The wizard steps view models

Since my intention was to have a multi-step wizard from the begining, I figured
I could start by defining an abstract class and a couple of different steps
that would inherit from it as viewmodels. I usually prefer scratching in C#
instead of HTML to get an idea of how the UI is going to look, and viewmodels
are perfect for that—once you have them figured out, you only have to map their
properties to the relevant template.

```cs
public abstract class StepViewModel
{
    /// <summary>
    /// Allows to control the order of a list of steps.
    /// </summary>
    public int Position { get; protected set; }
}

public class PersonalInformationStep : StepViewModel
{
    [Display(Name = "First Name")]
    public string FirstName { get; set; }

    [Required, Display(Name = "Last Name")]
    public string LastName { get; set; }

    public PersonalInformationStep()
    {
        Position = 0;
    }
}

public class ContactInformationStep : StepViewModel
{
    public string Email { get; set; }
    public string Phone { get; set; }

    public ContactInformationStep()
    {
        Position = 1;
    }
}
```

### The Razor Page

The latter classes model the view of the different steps of the wizard.
However, we still haven't talked about the actual page that is going to contain
the different steps and the handlers, in order to decide which step to render
when, and what to do with the data. Let's start with a Page that orchestrates
our viewmodels.

```cs
public class Index : PageModel
{
    [BindRequired]
    [BindProperty(SupportsGet = true)]
    public int CurrentStepIndex { get; set; }

    public IList<StepViewModel> Steps { get; set; }

    private readonly ContactService _service;

    public Index(ContactService service)
    {
        _service = service;
        InitializeSteps();
    }

    private void InitializeSteps()
    {
        Steps = typeof(StepViewModel)
            .Assembly
            .GetTypes()
            .Where(t => !t.IsAbstract && typeof(StepViewModel).IsAssignableFrom(t))
            .Select(t => (StepViewModel) Activator.CreateInstance(t))
            .OrderBy(x => x.Position) // Notice how we make sure that they're in the right order.
            .ToList();
    }
}
```

Notice how we've also created a `CurrentStepIndex` property. This will be the
flag which will keep track of which step users are in.

### The _complete_ Razor Page

As you've noticed, the above isn't very useful yet, but I wanted to show you
the properties of the page first because, at the end of the day, that's the
core of all we're going to do. That's the necessary state for our wizard. Now
all we have to do is just play around with those, create some Razor handlers to
shuffle the data around, etc, but it's just data-juggling like I say.

Next I'm going to dump the code of the full Razor Page. It might seem a little bit
daunting at first, but I'll cover all the details.

```cs
public class Index : PageModel
{
    // Regarding cleaning the ModelState:
    // https://stackoverflow.com/questions/54356921/razor-views-bounded-property-not-updating-after-post

    [BindRequired]
    [BindProperty(SupportsGet = true)]
    public int CurrentStepIndex { get; set; }

    public IList<StepViewModel> Steps { get; set; }

    private readonly ContactService _service;

    public Index(ContactService service)
    {
        _service = service;
        InitializeSteps();
    }

    private void InitializeSteps()
    {
        Steps = typeof(StepViewModel)
            .Assembly
            .GetTypes()
            .Where(t => !t.IsAbstract && typeof(StepViewModel).IsAssignableFrom(t))
            .Select(t => (StepViewModel) Activator.CreateInstance(t))
            .OrderBy(x => x.Position)
            .ToList();
    }

    public IActionResult OnGetAsync(int? id)
    {
        if (id != null)
        {
            var client = _service.FindById((int) id);
            if (client != null)
            {
                LoadWizardData(client);
            }
            else
            {
                return NotFound();
            }
        }
        else
        {
            SetEmptyTempData();
        }

        return Page();

    }

    public PageResult OnPostNext(StepViewModel currentStep)
    {
        if (ModelState.IsValid) MoveToNextStep(currentStep);
        return Page();
    }

    public PageResult OnPostPrevious(StepViewModel currentStep)
    {
        if (ModelState.IsValid) MoveToPreviousStep(currentStep);
        return Page();
    }

    public IActionResult OnPostFinish(StepViewModel currentStep)
    {
        if (!ModelState.IsValid) return Page();

        var client = ProcessSteps(currentStep);
        _service.Save(client);
        return RedirectToPage("./../Index", new {id = client.Id});
    }

    private void LoadWizardData(Contact client)
    {
        TempData["ClientId"] = client.Id;

        Steps = StepMapper.ToSteps(client).OrderBy(x => x.Position).ToList();

        for (var i = 0; i < Steps.Count; i++)
        {
            TempData.Set($"Step{i}", Steps[i]);
        }
    }

    private void SetEmptyTempData()
    {
        TempData.Remove("ClientId");
        for (var i = 0; i < Steps.Count; i++)
        {
            TempData.Set($"Step{i}", Steps[i]);
        }
    }

    private void MoveToNextStep(StepViewModel currentStep) => JumpToStep(currentStep, CurrentStepIndex + 1);

    private void MoveToPreviousStep(StepViewModel currentStep) => JumpToStep(currentStep, CurrentStepIndex - 1);

    private void JumpToStep(StepViewModel currentStep, int nextStepPosition)
    {
        TempData.Set($"Step{CurrentStepIndex}", currentStep);
        CurrentStepIndex = nextStepPosition;
        JsonConvert.PopulateObject((string) TempData.Peek($"Step{CurrentStepIndex}"), Steps[CurrentStepIndex]);
        ModelState.Clear();
    }

    private Contact ProcessSteps(StepViewModel finalStep)
    {
        for (var i = 0; i < Steps.Count; i++)
        {
            var data = TempData.Peek($"Step{i}");
            JsonConvert.PopulateObject((string) data, Steps[i]);
        }

        Steps[CurrentStepIndex] = finalStep;

        var contact = new Contact();
        if (TempData.Peek("ClientId") != null)
        {
            contact.Id = (int) TempData["ClientId"];
        }

        StepMapper.EnrichClient(contact, Steps);
        return contact;
    }
}
```

If you've tried to copy paste the above, you will notice that some things don't
seem to exist. The first one is `TempData::Set` and the second one the Mapper.
Regarding TempData, I created a little extension for convenience. I figured
that serialising the data into JSON and saving it as a cookie could be the
simplest possible thing, so that's what I went with:

```cs
public static class TempDataExtensions
{
    public static void Set<T>(this ITempDataDictionary tempData, string key, T value) where T : class
    {
        tempData[key] = JsonConvert.SerializeObject(value);
    }
    public static T Get<T>(this ITempDataDictionary tempData, string key) where T : class
    {
        tempData.TryGetValue(key, out object o);
        return o == null ? null : JsonConvert.DeserializeObject<T>((string) o);
    }
}
```

As for the mapper, this is it:

```cs
internal static class StepMapper
{
    public static void EnrichClient(Contact contact, IEnumerable<StepViewModel> steps)
    {
        foreach (var step in steps)
        {
            switch (step)
            {
                case PersonalInformationStep s:
                    contact.FirstName = s.FirstName;
                    contact.LastName = s.LastName;
                    break;
                case ContactInformationStep s:
                    contact.Email = s.Email;
                    contact.Phone = s.Phone;
                    break;
            }
        }
    }

    public static IEnumerable<StepViewModel> ToSteps(Contact contact)
    {
        return new List<StepViewModel>
        {
            new PersonalInformationStep {FirstName = contact.FirstName, LastName = contact.LastName},
            new ContactInformationStep {Email = contact.Email, Phone = contact.Phone}
        };
    }
}
```

For mapping there are a ton of different approaches. I personally like to go
with static functions whenever it's a pure-ish function, but if you want to go
with a more DI-heavy approach in order to be able to mock everything in your
tests, then that's super-valid too.

### Moving back and forth in the wizard

At this point, we've seen all the relevant code except the Razor templates, so
let's try to explain a few things.

As you'll have noticed, the code is fairly self-explanatory. The most important
thing is to not loose sight of what happens every time we move back and forth
in the wizard:

```cs
private void JumpToStep(StepViewModel currentStep, int nextStepPosition)
{
    TempData.Set($"Step{CurrentStepIndex}", currentStep);
    CurrentStepIndex = nextStepPosition;
    JsonConvert.PopulateObject((string) TempData.Peek($"Step{CurrentStepIndex}"), Steps[CurrentStepIndex]);
    ModelState.Clear();
}
```

1. We save the current step to TempData (cookies)
2. We load the next step from TempData into memory, so it can then be sent in
   the response to the request of the next step.
3. We clear the ModelState

You'll be wondering: `ModelState.Clear()`? What? Well I'll be damned if there
aren't gotchas in every technology I've used! Basically we just gotta get rid
of the previous value, or else ASP get's that over the new one we're binding.
Read more [here][4].

Also, I'll very briefly comment on why I decided to use `TempData::Peek` over
`TempData::Get`. The main reason is because `Get` removes the value once read,
while `Peek` doesn't. That kind of gives me more jiggle space, if you know what
I mean. I couldn't come up with any bad reason not to do it :-)

## Model binding

If you've gotten to this section, it's probably because you're interested in
how it wraps up. So far, we've built all the little blocks that put together a
server rendered wizard except one, one of the most crucial, actually. Without
it it won't work. It's the model binder.

In ASP.NET Core, to avoid having to type all the verbose as well as error-prone
code that is mapping the HTML form result to C# entities, the team which worked
on it created what are called as [model binders][5]. It's basically a piece of
Middleware which does that work for us—tries to understand the format of the
incoming payload and maps it to a C# entity.

In our specific case, what we need is what's defined as a [_polymorphic model
binder_][6]. That's a binder that gives us a different instance based on
whatever it actually is. It's not the prettiest bit of code, nor the easiest to
understand, but here it is:

```cs
public class StepModelBinder : IModelBinder
{
    private readonly Dictionary<Type, (ModelMetadata, IModelBinder)> _binders;

    public StepModelBinder(Dictionary<Type, (ModelMetadata, IModelBinder)> binders)
    {
        _binders = binders;
    }

    public async Task BindModelAsync(ModelBindingContext bindingContext)
    {
        var modelTypeValue = bindingContext.ValueProvider.GetValue("StepType").FirstValue;
        var modelType = Type.GetType(modelTypeValue, true);

        IModelBinder modelBinder;
        ModelMetadata modelMetadata;
        if (modelTypeValue.Contains("Step"))
        {
            (modelMetadata, modelBinder) = _binders[modelType];
        }
        else
        {
            bindingContext.Result = ModelBindingResult.Failed();
            return;
        }

        var newBindingContext = DefaultModelBindingContext.CreateBindingContext(
            bindingContext.ActionContext,
            bindingContext.ValueProvider,
            modelMetadata,
            bindingInfo: null,
            bindingContext.ModelName);

        await modelBinder.BindModelAsync(newBindingContext);
        bindingContext.Result = newBindingContext.Result;

        if (newBindingContext.Result.IsModelSet)
        {
            // Setting the ValidationState ensures properties on derived types are correctly
            bindingContext.ValidationState[newBindingContext.Result] = new ValidationStateEntry
            {
                Metadata = modelMetadata,
            };
        }
    }
}

public class StepModelBinderProvider : IModelBinderProvider
{
    public IModelBinder GetBinder(ModelBinderProviderContext context)
    {
        if (context.Metadata.ModelType != typeof(StepViewModel))
        {
            return null;
        }

        var subclasses = new[]
        {
            typeof(PersonalInformationStep),
            typeof(ContactInformationStep),
        };

        var binders = new Dictionary<Type, (ModelMetadata, IModelBinder)>();

        foreach (var type in subclasses)
        {
            var modelMetadata = context.MetadataProvider.GetMetadataForType(type);
            binders[type] = (modelMetadata, context.CreateBinder(modelMetadata));
        }

        return new StepModelBinder(binders);
    }
}
```

Take into account that you will need to register the provider in the
`Startup.cs` class so the runtime can generate the binder when a request
comes in.

```cs
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    services.AddRazorPages();
    services.AddSingleton(new ContactService());

    services.AddMvc(options =>
    {
        // Adding your provider to the end of the collection may result in a built-in
        // model binder being called before your custom binder has a chance.
        options.ModelBinderProviders.Insert(0, new StepModelBinderProvider());
    });

}
```

## The importance of View Models

One last thing I wanted to bring forward is the usage of view models. In this
example, view models have been particularly useful because they allow us to
encapsulate the steps, but there are many other benefits to it. This is a
simplified example, but the real wizard I developed had 6 steps and over 20
fields per step. At a certain point I was asking my self why I didn't reuse the
Domain model instead of creating view models for everything.

On the one hand, while using the domain model used by the database and the
services does provide with a significant speed boost initially, I have
discovered in my experience that it's only in the beginning. The true benefit
of following a domain-driven design and stablishing the correct boundaries is
that we can then make changes freely without having to do _full-stack_ changes.
In this case, if I want to change anything related to the view, I only have to
tamper with the HTML and the view model, and not worry about all the underlying
code. The same applied otherwise—I was able to refactor the domain-side of things
reckleslly without having to wonder too much about the view. The boundary is the
mapper.

Another benefit of using view models is that we only play with the data we
need. Sometimes domain models have way more data than what the actual view
requires. That allows us to keep the views much dumber and focused than
otherwise. It allows our colleagues to not have to worry about the properties
that aren't being displayed. Did we forget to display them or are they
supposed to not be displayed? Not to mention if we're ALSO sending that to
the frontend. Does our user need to know our full contact? Or just its name?

While it is also true that in this specific piece of work using view models
also greatly simplifies the code, sometimes it actually complicates it. Those
are the times when it's important to keep all this in mind. It's all about
domain-driven design.C# is verbose and sometimes it's easy to not see the
benefits of things, until it's late and we miss it.

## Final thoughts

You might have noticed that I haven't shown any of the actual Razor code. I
made the decision not to because the blog post was already being too long and
the Razor code was fairly irrelevant. Most of it was simply displaying the
actual primitives:

```
<div class="form-group">
    <label asp-for="FirstName" class="control-label required"></label>
    <input asp-for="FirstName" class="form-control" />
    <span asp-validation-for="FirstName" class="text-danger"></span>
</div>
```

However, please feel free to check the github [repository][7] with all the
code. You will find all the Razor templates in there.

On a different note, I have found this to be an extremely insightful experience
and I really recommend anybody to try and develop a piece of code like this in
any stack they wish. It has given me the insight to understand that sometimes
it makes sense to try the newest and shiniest framework that has come out, but
other times some plain old server html is more than enough, actually even
better than enough, because it allows us to keep the application simpler, the
development workflow faster and the headspace smaller.

[1]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel
[2]: https://www.learnrazorpages.com/razor-pages/tempdata
[3]: https://developer.mozilla.org/es/docs/Web/HTTP/Headers/Set-Cookie
[4]: https://stackoverflow.com/questions/54356921/razor-views-bounded-property-not-updating-after-post
[5]: https://docs.microsoft.com/en-us/aspnet/core/mvc/models/model-binding?view=aspnetcore-3.1
[6]: https://docs.microsoft.com/en-us/aspnet/core/mvc/advanced/custom-model-binding?view=aspnetcore-3.0#polymorphic-model-binding
[7]: https://github.com/Manzanit0/ServerWizardExample
