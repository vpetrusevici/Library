---

## ✨ Looking For Sponsors ✨

FastEndpoints needs sponsorship to [sustain the project](https://github.com/FastEndpoints/FastEndpoints/issues/449). Please help out if you can.

---

[//]: # (<details><summary>title text</summary></details>)

## New 🎉

<details><summary>New generic attribute [Group&lt;T&gt;] for attribute based endpoint group configuration</summary>

When using attribute based endpoint configuration, you can now use the generic 'Group<TEndpointGroup>' attribute to specify the group which the endpoint belongs to like so:

```csharp
//group definition class
sealed class Administration : Group
{
    public Administration()
    {
        Configure(
            "admin",
            ep =>
            {
                ep.Description(
                    x => x.Produces(401)
                          .WithTags("administration"));
            });
    }
}

//using generic attribute to associate the endpoint with the above group
[HttpPost("login"), Group<Administration>]
sealed class MyEndpoint : EndpointWithoutRequest
{
    ...
}
```

</details>

<details><summary>Specify a label, summary & description for Swagger request examples</summary>

When specifying multiple swagger request examples, you can now specify the additional info like this:

```csharp
Summary(
    x =>
    {
        x.RequestExamples.Add(
            new(
                new MyRequest { ... },
                "label",
                "summary",
                "description"));
    });
```

</details>

<details><summary>Automatic type inference for route params from route constraints for Swagger</summary>

Given route templates such as the following that has type constraints for route params, it was previously only possible to correctly infer the type of the parameter (for Swagger spec generation) if the parameters are being bound to a request DTO and that DTO has a matching property. The following will now work out of the box and the generated Swagger spec will have the respective parameter type/format.

```csharp
sealed class MyEndpoint : EndpointWithoutRequest
{
    public override void Configure()
    {
        Get("test/{id:int}/{token:guid}/{date:datetime}");
        AllowAnonymous();
    }

    public override async Task HandleAsync(CancellationToken c)
    {
        var id = Route<int>("id");
        var token = Route<Guid>("token");
        var date = Route<DateTime>("date");

        await SendAsync(new { id, token, date });
    }
}
```

You can register your own route constraint types or even override the default ones like below by updating the Swagger route constraint map:

```csharp
FastEndpoints.Swagger.GlobalConfig.RouteConstraintMap["nonzero"] = typeof(long);
FastEndpoints.Swagger.GlobalConfig.RouteConstraintMap["guid"] = typeof(Guid);
FastEndpoints.Swagger.GlobalConfig.RouteConstraintMap["date"] = typeof(DateTime);
```

</details>

<details><summary>Form related exception transformer function setting</summary>

When accessing Form data there are various cases where an exception would be thrown internally by ASP.NET such as in the case of the incoming request body size exceeding the default limit or whatever you specify like so:

```csharp
bld.WebHost.ConfigureKestrel(
    o =>
    {
        o.Limits.MaxRequestBodySize = 30000000;
    });
```

If the incoming request body size is more than `MaxRequestBodySize`, Kestrel would automatically short-circuit the response with a `413 - Content Too Long` response, which may not be what you want. You can instead specify a `FormExceptionTrasnformer` func to transform the exception in to a regular 400 error/problem details JSON response like so:

```csharp
app.UseFastEndpoints(
       c =>
       {
           c.Errors.UseProblemDetails(); //this is optional
           c.Binding.FormExceptionTransformer =
               ex => new ValidationFailure("GeneralErrors", ex.Message);
       })
```

Which would result in a JSON response like so:

```json
{
    "type": "https://www.rfc-editor.org/rfc/rfc7231#section-6.5.1",
    "title": "One or more validation errors occurred.",
    "status": 400,
    "instance": "/upload-file",
    "traceId": "0HN39MGSS8QDA:00000001",
    "errors": [
        {
            "name": "generalErrors",
            "reason": "Request body too large. The max request body size is 30000000 bytes."
        }
    ]
}
```

</details>

[//]: # (## Improvements 🚀)

## Fixes 🪲

<details><summary>Contention issue resulting in random 415 responses</summary>

There was a possible contention issue that could arise in and extremely niche edge case where the WAFs could be instantiated in quick succession which results in tests failing due to 415 responses being returned randomly. This has been fixed by moving the necessary work to be performed at app startup instead of at the first request for a particular endpoint. More info: #661

</details>

<details><summary>Eliminate potential contention issues with 'AppFixture'</summary>

`AppFixture` abstract class has been improved to use an Async friendly Lazy initialization technique to prevent any chances of more than a single WAF being created per each derived `AppFixture` type in high concurrent situations. Previously we were relying solely on `ConcurrentDictionary`'s thread safety features which did not always give the desired effect. Coupling that with Lazy initialization seems to solve any and all possible contention issues.

</details>

<details><summary>Correct exception not thrown when trying to instantiate response DTOs with required properties</summary>

When the response DTO contains required properties like this:

```csharp
public class MyResponse
{
    public required string FullName { get; set; }
}
```

If an attempt was made to utilize the auto response sending feature by setting properties of the `Response` object, a 400 validation error was being thrown instead of a 500 internal server error. It is now correctly throwing the `NotSupportedException` as it should, because FE cannot automatically instantiate objects that have required properties and the correct usage is for you to instantiate the object yourself and set it to the `Response` property, which is what the exception will now be instructing you to do.

</details>

## Breaking Changes ⚠️

<details><summary>The way multiple Swagger request examples are set has been changed</summary>

Previous way:

```csharp
Summary(s =>
{
    s.RequestExamples.Add(new MyRequest {...});
});
```

New way:

```csharp
s.RequestExamples.Add(new(new MyRequest { ... })); // wrapped in a RequestExample class
```

</details>

<details><summary>'PreSetupAsync()' trigger behavior change in `AppFixture` class</summary>

Previously the `PreSetupAsync()` virtual method was run per each test-class instantiation. That behavior does not make much sense as the WAF instance is created and cached just once per test run. The new behavior is more in line with other virtual methods such as `ConfigureApp()` & `ConfigureServices()` that they are only ever run once for the sake of creation of the WAF. This change will only affect you if you've been creating some state such as a `TestContainer` instance in `PreSetupAsync` and later on disposing that container in `TearDownAsync()`. From now on you should not be disposing the container yourself if your derived `AppFixture` class is being used by more than one test-class. See [this gist](https://gist.github.com/dj-nitehawk/04a78cea10f2239eb81c958c52ec84e0) to get a better understanding.

</details>

<details><summary>Consolidate Jwt key signing algorithm properties into one</summary>

`JwtCreationOptions` class had two different properties to specify `SymmetricKeyAlgorithm` as well as `AsymmetricKeyAlgorithm`, which has now been consolidated into a single property called `SigningAlgorithm`.

Before:

```csharp
var token = JwtBearer.CreateToken(
    o =>
    {
        o.SymmetricKeyAlgorithm = SecurityAlgorithms.HmacSha256Signature;
        //or
        o.AsymmetricKeyAlgorithm = SecurityAlgorithms.RsaSha256;
    });
```

After:

```csharp
var token = JwtBearer.CreateToken(
    o =>
    {
        o.SigningStyle = TokenSigningStyle.Symmetric;
        o.SigningAlgorithm = SecurityAlgorithms.HmacSha256Signature;
    });
```

</details>