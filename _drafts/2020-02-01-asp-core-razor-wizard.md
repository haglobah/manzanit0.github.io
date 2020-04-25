---
layout: post
title: ".NET Core: Building a server-rendered wizard"
author: Javier García
category: .NET
tags: dotnet, asp, server-rendering, forms
---

As you can see based on my most recent posts, I've been focusing on some
ASP.NET Core development.  Recently, one of the things I've been working on is
a snappy form with a wizard-like interface, which would allow the users of an
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
a frontend interface which consumes the API.  Sometimes, you just don't need
all the RESTful endpoints, nor the JSON serialising and de-serialising, dealing
with CORS, or many of the other inconvenients of the architecture. You just
need some SIMPLE.

In the particular case of the application I'm working on, I have discovered
that simply exposing a set of sites is more than enough. And then, maybe, if
tomorrow I do end up needing an API... I can always start working on it and
adapt my architecture as needed. It's all about evolving architecture.

## Designing the solution

Now, being a sucker for simple tech and simple design, the MS offering that
made the most sense for this case was Razor Pages. You simply create a _Page_,
a _PageModel_ with some event handlers and done. No MVC, no SPA, nothing. Just
a simple page and some event handlers for it.

As an extra bonus, while playing around with Razor Pages, I discovered them to
be not just incredibly simple and easy to use but also extremely fast (although
it does make sense... simple is usually fast!).  This is a quick snapshot of
how fast going back and forward through the wizard was: ~120ms in average.

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

First of all the entity which we will be making this for is `Person`:

```cs
public class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public string Phone { get; set; }
}
```

Since my intention was to have a multi-step wizard from the begining, I figured
I could start by defining an abstract class and a couple of different steps
that would inherit from it. That would make it much easier to deal with
different steps that would have different kinds of data.

```cs
public abstract class StepViewModel { }


public class Step1ViewModel : StepViewModel
{
    [Required, Display(Name = "First Name")]
    public string FirstName { get; set; }

    [Required, Display(Name = "Last Name")]
    public string LastName { get; set; }
}

public class Step2ViewModel : StepViewModel
{
    public string Email { get; set; }

    public string Phone { get; set; }
}
```

Those model the different steps of my wizard. What about the actual page that
is going to contain the different steps and the handlers, to decide which to
render when, and what to do with the data? Let's start with a Page that
contains that list and initialises it.

```cs
public class WizardPage : PageModel
{
    [BindRequired]
    public int CurrentStepIndex { get; set; }
    public List<StepViewModel> Steps { get; set; }

    public WizardPage()
    {
      Initialize();
    }

    public void Initialize()
    {
        Steps = typeof(StepViewModel)
            .Assembly
            .GetTypes()
            .Where(t => !t.IsAbstract && typeof(StepViewModel).IsAssignableFrom(t))
            .Select(t => (IStepViewModel) Activator.CreateInstance(t))
            .ToList();
    }
}
```

### Loading the information for an existing record

Next, it would be most convenient to load the data if what we're doing is
editing a record.  We can do that with our first handler, in `WizardPage`. In
all the following snippets you will notice a reference to `_service`. I don't
want to show the code for that because I find it is not relevant for the
purpouse of this blog post, so simply imagine it is a Stateless class which
allows us to find, save and overall interact with _persons_, it's injected via
DI in the Page's controller, and that's all that's relevant.

```cs
public async Task<IActionResult> OnGetAsync(int? id)
{
    if (id != null)
    {
        var person = await _service.FindById((int) id);
        if (person != null)
        {
            // Save the ID to TempData so that we can keep track of the actual
            // record we're modifying.
            LoadWizardData();
        }
        else
        {
            return NotFound();
        }
    }
    else
    {
      // If no person was found by the given ID, reset all the information.
      ResetTempData();
    }

    return Page();
}

public void LoadWizardData(Person person)
{
    TempData["PersonId"] = person.ID;

    Steps = WizardMapper.ToWizardSteps(person);
    for (var i = 0; i < Steps.Count; i++)
    {
        TempData.Set($"Step{i}", Steps[i]);
    }
}

public void ResetTempData()
{
    TempData.Remove("PersonId");
    for (var i = 0; i < Steps.Count; i++)
    {
        TempData.Set($"Step{i}", Steps[i]);
    }
}
```

If you've tried to copy paste the above, you will notice that some things don't
seem to exist.  The first one is `TempData::Set` and the second one the Mapper.
Regarding TempData, I created a little extension for convenience. I figured
that serialising the data into JSON and saving it could be the simplest
possible thing, so that's what I went with:

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

As for the wizard, this is more or less it:

```cs
public static class WizardMapper
{
    public static List<StepViewModel> ToWizardSteps(Person person)
    {
        var step1 = new Step1ViewModel { FirstName = person.FirstName, LastName = person.LastName };
        var step2 = new Step2ViewModel { Phone = person.Phone, Email = person.Email };
        return new List<StepViewModel> { step1, step2};
    }
}
```

### Moving back and forth in the wizard

So far, so good. At this point, we just need a couple of handlers to move back
and forth, and the model binder so the form actually works. I'll cover model
binding later, so let's go with the remaining handlers.

In my world, handlers should be as simple as possible, so that's what I tried:

```cs
public IActionResult OnPostNext(IStepViewModel currentStep)
{
    if (ModelState.IsValid)
    {
        MoveToNextStep(currentStep);
    }

    return Page();
}

public IActionResult OnPostPrevious(IStepViewModel currentStep)
{
    if (ModelState.IsValid) MoveToPreviousStep(currentStep);

    return Page();
}
```

Now, the actual detail is in the `MoveToNextStep` and `MoveToPreviousStep`:

```cs
private void MoveToNextStep(IStepViewModel currentStep)
{
    // Save the step we just finished editing.
    TempData.Set($"Step{CurrentStepIndex}", currentStep);

    // Increment the step of the wizard we want to be in
    CurrentStepIndex++;

    // Read the cookies so we load the information of the next step, if there is any.
    JsonConvert.PopulateObject((string) TempData.Peek($"Step{CurrentStepIndex}"), Steps[CurrentStepIndex]);
    ModelState.Remove("CurrentStepIndex");
}

// Same comments as above apply :-)
private void MoveToPreviousStep(IStepViewModel currentStep)
{
    TempData.Set($"Step{CurrentStepIndex}", currentStep);

    CurrentStepIndex--;
    JsonConvert.PopulateObject((string) TempData.Peek($"Step{CurrentStepIndex}"), Steps[CurrentStepIndex]);
    ModelState.Remove("CurrentStepIndex");
}
```

But you'll be wondering `ModelState.Remove("CurrentStepIndex")`? What? Well
I'll be damned if there aren't gotchas in every technology I've used! Basically
we just gotta get rid of the previous value, or else ASP get's that over the
new one we're binding. Read more [here][4]. Also, I'll very briefly comment on
why I decided to use `TempData::Peek` over `TempData::Get`. The main reason is
because `Get` removes the value once read, while `Peek` doesn't. That kind of
gives me more jiggle space, if you know what I mean. I couldn't come up with
any bad reason not to do it :-)

### Saving and closing the wizard

Lastly, we want to have a `save & close` button in our wizard. Let's get over
with the last handler for that:

```cs
public async Task<IActionResult> OnPostFinish(StepViewModel currentStep)
{
    if (!ModelState.IsValid) return Page();

    var person = ProcessSteps(currentStep);
    await _service.Save(person);

    // Once the user clicks on save, take him elsewhere. Maybe.
    return RedirectToPage("./some-other-page?");
}

private Person ProcessSteps(StepViewModel finalStep)
{
    // Read every step in the cookies and load them into
    // the in-memory list.
    for (var i = 0; i < Steps.Count; i++)
    {
        var data = TempData.Peek($"Step{i}");
        JsonConvert.PopulateObject((string) data, Steps[i]);
    }

    // Also include the last step we just received
    Steps[CurrentStepIndex] = finalStep;

    // Build our Person instance so we can save it afterwards.
    var person = new Person();
    if (TempData.Peek("PersonId") != null)
    {
        person.ID = (int) TempData["PersonId"];
    }

    // This simply maps the steps to the person instance.
    return WizardMapper.EnrichPerson(person, Steps);
}
```

## Model binding

If you've gotten to this section, it's probably because you're interested in
how it wraps up.  So far, we've build all the little blocks that put together a
server rendered wizard except one, one of the most crucial, actually. Without
it it won't work. It's the model binder.

In ASP.NET Core, to avoid having to type all the verbose as well as error-prone
code that is mapping the HTML form result to C# entities, the team which worked
on it created what is called as [model binders][5]. It's basically a piece of
Middleware which does that work for us.

In our specific case, what we need is what's defined as a [_polymorphic model
binder_][6]. That's a binder that gives us a different instance based on
whatever it actually is. It's not the prettiest bit of code, nor the easiest to
understand, but here it is:

```cs
public class WizardModelBinder : IModelBinder
{
    private Dictionary <Type, (ModelMetadata, IModelBinder)> binders;

    public WizardModelBinder(Dictionary < Type, (ModelMetadata, IModelBinder) > binders)
    {
        this.binders = binders;
    }

    public async Task BindModelAsync(ModelBindingContext bindingContext)
    {
        var modelTypeValue = bindingContext.ValueProvider.GetValue("StepType").FirstValue;
        var modelType = Type.GetType(modelTypeValue, true);

        IModelBinder modelBinder;
        ModelMetadata modelMetadata;
        if (modelTypeValue.Contains("Step"))
        {
            (modelMetadata, modelBinder) = binders[modelType];
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
            bindingInfo : null,
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

public class WizardModelBinderProvider : IModelBinderProvider
{
    public IModelBinder GetBinder(ModelBinderProviderContext context)
    {
        if (context.Metadata.ModelType != typeof(StepViewModel))
        {
            return null;
        }

        // Here list all the subclasses. Basically, as we add more steps to the
        // wizard, we would only have to add further types below.
        // This could also be done with reflection.
        var subclasses = new []
        {
            typeof(Step1ViewModel),
            typeof(Step2ViewModel),
        };

        var binders = new Dictionary < Type,
            (ModelMetadata, IModelBinder) > ();

        foreach (var type in subclasses)
        {
            var modelMetadata = context.MetadataProvider.GetMetadataForType(type);
            binders[type] = (modelMetadata, context.CreateBinder(modelMetadata));
        }

        return new WizardModelBinder(binders);
    }
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
code.

Another benefit of using view models is that we only play with the data we
need. Sometimes domain models have way more data than what the actual view
requires. That allows us to keep the views much dumber and focused than
otherwise. It allows our colleagues to not have to worry about the properties
that aren't being displayed.  Did we forget to display them or are they
supposed to not be displayed?

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

I have found this to be an extremely insightful experience and I really
recommend anybody to try and develop a piece of code like this in any stack
they wish. It has given me the insight to understand that sometimes it makes
sense to try the newest and shiniest framework that has come out, but other
times some plain old server html is more than enough, actually even better than
enough, because it allows us to keep the application simpler, the development
workflow faster and the headspace smaller.


[1]: https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel
[2]: https://www.learnrazorpages.com/razor-pages/tempdata
[3]: https://developer.mozilla.org/es/docs/Web/HTTP/Headers/Set-Cookie
[4]: https://stackoverflow.com/questions/54356921/razor-views-bounded-property-not-updating-after-post
[5]: https://docs.microsoft.com/en-us/aspnet/core/mvc/models/model-binding?view=aspnetcore-3.1
[6]: https://docs.microsoft.com/en-us/aspnet/core/mvc/advanced/custom-model-binding?view=aspnetcore-3.0#polymorphic-model-binding
