# Upgrading Optimizely to CMS 12 and Commerce 14

[[_TOC_]]

## Background

There are many great resources for learning CMS12 and Commerce14:

- The official developer documentation has been updated:
  https://world.optimizely.com
- The official user guild has been updated:
  https://webhelp.optimizely.com
- Mark Price and Scott Reed host a masterclass on .NET5:
  https://www.optimizely.com/support/education/product/migrating-to-optimizely-cms-12-and-commerce-14

...and so on.
But there aren’t many resources for upgrading from CMS 11 / Commerce 13

## Prerequisites

Phase 0: Stuff to do before you begin

### Read the official documentation

1.  Upgrading to Content Cloud (CMS 12)
    https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/upgrading-to-content-cloud-cms-12

2.  Breaking changes in Content Cloud (CMS 12)
    https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/breaking-changes-in-content-cloud-cms-12

3.  System requirements for Optimizely
    https://docs.developers.optimizely.com/content-cloud/v12.0.0-content-cloud/docs/system-requirements-for-optimizely

### Be on .NET Framework 4.7.2 or higher

CMS11 only requires .NET Framework 4.6.1, but Microsoft recommends being on 4.7.2 or higher when using Upgrade-Assistant

- https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/system-requirements-for-optimizely
- https://docs.microsoft.com/en-us/dotnet/core/porting/premigration-needed-changes

### Update to the latest version of CMS11 (Commerce13) before upgrading

The official documentation doesn’t explicitly say to do this, but I can’t think of any reason not to.

### Check the status of add-on packages

- Official CMS/Commerce add-on platform compatibility:
  https://docs.developers.optimizely.com/integrations/v1.1.0-apps-and-integrations/docs/add-ons-platform-compatibility

- Official big list of package migration status:
  https://world.optimizely.com/resources/net5/add-ons

### Check the status of add-on packages

Give yourself time to check the status of your favorite third party add-ons.

Having no workaround for unsupported add-ons could derail your whole upgrade project. Know what you’re getting into.

Note: Some old, .NET Framework add-ons will still work, just with a warning.

Example: Authorize.Net.
There is no .NET Core+ package, but it still compiles and runs.

## Upgrade-Assistant

Phase 1: How to use Microsoft's CLI tool for upgrading to .NET 5+

### Delete Commerce Manager

- As of Commerce 14, Commerce Manager is no more.
- 👏
- Remove the Commerce Manager project before getting started with Upgrade-Assistant.
- Note that some of Commerce Manager’s functionality hasn’t been ported over to the CMS yet and can only be done with APIs:
  - Importing and exporting catalogs
  - Adding countries and regions
  - Adding currencies
  - Working with business objects
  - Working with catalog and order meta classes and fields

### Delete `node_modules`

As a first step, Upgrade-Assistant copies all files in your solution/project folder into a Backup directory.

This can be disabled, but to play it safe:
If you have `node_modules` in the project folder you are upgrading, then delete it before running UA.

(don’t forget to stop any local watch)

### Use Opti’s Upgrade-Assistant-Extensions

Optimizely provides Upgrade-Assistant extensions with some Opti-specific capabilities:

- String Replacement
- Remove Default Argument for the TemplateDescriptor Attribute
- Base Class Mapping
- Replace IFindUIConfiguration with FindOptions
- Remove PropertyData ParseToObject method overrides
- Remove obsolete using statements like Mediachase.BusinessFoundation
- Type Mapping like EPiServer.Web.Routing to EPiServer.Core.Routing

Additionally, NuGet packages can be specified and templates for Program.cs and Startup.cs as they are required by .NET 5.0 can be added as well.

Read how it works (there are a couple gotchas):<br/>
https://github.com/episerver/upgrade-assistant-extensions

Learn how to configure UAE:<br/>
https://github.com/episerver/upgrade-assistant-extensions/releases

Ugrade-Assistant-Extensions will do some nice things for you out of the box (i.e., replace `BlockController` with `BlockComponent`\*), but do consider spending some time customizing the config for string/type/class replacements.

\*It does not, e.g., change `PageController` `Index` methods into `IndexAsync` methods, though.

How to get it ready:

1. Download the latest release (Epi.Source.Updater.X.Y.Z.zip):
   https://github.com/episerver/upgrade-assistant-extensions/releases
2. Unzip it to your local file system. E.g.,
   ` C:\Temp\Epi.Source.Updater\`
3. Make your preferred configuration changes

### Make a plan-of-attack before running Upgrade-Assistant

Upgrade-Assistant can run against a Solution file (`.sln`) or Project file (`.csproj`).

If you run it against the Solution, it will execute against the child Projects in sequence according to the project dependency tree.

Example (RE: onion architecture):

1. `MySolution.Domain.csproj`
2. `MySolution.Application.csproj`
3. `MySolution.Web.csproj`
4. (done)

### Consider doing one CSPROJ at a time

Upgrade-Assistant will track progress and start where it left off if you cancel it at any time. But...

Do figure out the dependency sequence first and then run UA manually against each Project. This will allow you to resolve code issues in isolation on a per-Project basis without getting confused about where you are with UA.

1. `MySolution.Domain.csproj`
   1. Run Upgrade-Assistant
   2. Fix code issues
   3. Commit to source control
2. `MySolution.Application.csproj`
   1. Run Upgrade-Assistant
   2. Fix code issues
   3. Commit to source control
3. `MySolution.Web.csproj`
   1. Run Upgrade-Assistant
   2. Fix code issues
   3. Commit to source control

### Think about what flags to use

Basic syntax if your terminal is at the solution root:

```
upgrade-assistant upgrade MySolution.Web/MySolution.Web.csproj --flags-go-here
```

Flags:

`--extension "c:\temp\epi.source.updater"`<br/>`--target-tfm-support LTS`
<br/>Enables Opti’s Upgrade-Assistant-Extensions

`--ignore-unsupported-features`
<br/>This is required for upgrading the web application CSPROJ.

`--skip-backup`
<br/>Without this, UA will copy all solution files into /Backup/ first (RE: delete node_modules). But don’t you have source control?

`--non-interactive`
<br/>Officially: Microsoft’s documentation says that Upgrade-Assistant is meant to be interactive and to think twice about using this flag.
<br/>Unofficially: If you don’t use this flag, you will be sitting at your keyboard, pressing Enter repeatedly, for hours.

### Install and update Upgrade-Assistant

Open a terminal from anywhere:

```
dotnet tool install -g upgrade-assistant
dotnet tool update -g upgrade-assistant
```

https://docs.developers.optimizely.com/content-cloud/v12.0.0-content-cloud/docs/upgrade-assistant

### Run Upgrade-Assistant

From a terminal in your solution root (recommended):

`set DefaultTargetFrameworks__LTS=net5.0`
<br/>(this is required by Upgrade-Assistant-Extensions)

```
upgrade-assistant upgrade MySolution.Web/MySolution.Web.csproj
    --ignore-unsupported-features
    --skip-backup
    --non-interactive
    --extension "c:\temp\epi.source.updater"
    --target-tfm-support LTS
```

### Wait

(this can take several minutes up to hours)

### Review the code changes

`+ Properties/launchSettings.json`
<br/>Local server/IIS Express settings. Note that .NET5+ runs on HTTPS by default!

`+ appsettings.Development.json`
<br/>`+ appsettings.json`
<br/>Where your Web.config appSettings and connectionStrings went. TBD on guidance from the DXP team…

`- packages.config`
<br/>Packages are now referenced in the CSPROJ files.

`+ Program.cs`
<br/>`+ Startup.old.cs`
<br/>Program.cs and Startup.cs will need to be ported over. Look at Foundation for inspiration:
<br/>https://github.com/episerver/Foundation/tree/main/src/Foundation

### Commit the broken code

Do this so that if the code fixes go sideways, you can easily go back to the state immediately after running the Upgrade-Assistant

Consider checking in `.upgrade-assistant`. This is where UA tracks what steps it has already done.

Commit frequently from this point on.

### Delete leftover assembly dependencies

Some .NET Framework System assemblies will not have corresponding packages and get orphaned in the Dependencies > Assemblies node.

Unless any of these were explicitly added by your implementation, you should be free to delete them.

### Uninstall obsolete NuGet packages

Some EPiServer packages will need to be replaced entirely (i.e., removed and replaced with something else):
<br/>https://docs.developers.optimizely.com/content-cloud/v12.0.0-content-cloud/docs/breaking-changes-in-content-cloud-cms-12

Uninstall:

- `EPiServer.CMS.AspNet`
- `EPiServer.Framework.AspNet`
- `EPiServer.ServiceLocation.StructureMap`
- `EPiServer.Logging.Log4Net`

NuGet Package Manager can help. Example:

- The most recent version of `EPiServer.CMS.AspNet` is 11.x, so you know this one must be replaced.
- But the most recent version of `EPiServer.CMS.UI.AspNetIdentity` is 12+, so you know this can be updated.

### Manually resolve package errors

```
NU1177: Version conflict detected for Xyz. Install/reference Xyz.1.2.3 directly to project MySolution.Web to resolve this issue.
```

https://docs.microsoft.com/en-us/nuget/reference/errors-and-warnings/nu1107

Address this by:

1. Open your new CSPROJ file (just double-click the project in Solution
2. Find where all the `<PackageReference />` elements are.
3. Manually add the package reference it is complaining about, e.g.,
   <br/>`<PackageReference Include="Xyz" Version="1.2.3" />`

Keep track of which packages you add manually. Once you're finished, if they are redundant then you can remove them.

### Update NuGet packages

`EPiServer.CMS.AspNet` should be replaced with the following:

- `EPiServer.CMS.AspNetCore`
- `EPiServer.CMS.AspNetCore.Templating`
- `EPiServer.CMS.AspNetCore.Routing`
- `EPiServer.CMS.AspNetCore.Mvc`
- `EPiServer.CMS.AspNetCore.HtmlHelpers`

`EPiServer.Framework.AspNet` should be replaced with `EPiServer.Framework.AspNetCore`.

### Address known breaking changes

Go through the official breaking changes documentation:
<br/>https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/breaking-changes-in-content-cloud-cms-12

It’s dense but worth it.

## Code Fixes

Phase 2: Code issues that are commonly encountered

### Replace `HttpContextHelper` with `IHttpContextAccessor`

Upgrade-Assistant gives you the `HttpContextHelper` static helper class and replaces all instances of `HttpContext.Current` with it. But we want to use DI, right?

.NET Core introduced `IHttpContextAccessor`, which you can inject.

```cs
// .NET Framework
string myCookie = HttpContext.Current.Request.Cookies[CookieNames.PostalCode]?.Value;

// .NET Core
string myCookie = _httpContextAccessor.HttpContext?.Request.Cookies["MyCookie"];
```

### Give yourself time to replace HttpRequest

```cs
// .NET Framework
string userIp = httpRequest.ServerVariables["HTTP_X_FORWARDED_FOR"]
    ?? httpRequest.UserHostAddress;
string userAgent = httpRequest.UserAgent;
string host = httpRequest.Url.Host;
string url = httpRequest.Url.ToString();
string anonymousId = httpRequest.AnonymousID;

// .NET Core
string userIP = httpRequest.HttpContext.GetServerVariable("HTTP_X_FORWARDED_FOR")
    ?? request.HttpContext.Connection.RemoteIpAddress?.ToString();
string userAgent = httpRequest.Headers["User-Agent"];
string host = httpRequest.Host.ToString();
string url = httpRequest.GetDisplayUrl(); // or GetEncodedUrl()
// There is no AnonymousID. Roll your own!
```

### We need to talk about HttpClient.

Managing the lifecycle of `HttpClient` in .NET Framework was always a pain. Even though `HttpClient` implements `IDisposable`, putting it in a `using` statement can lead to SNAT port exhaustion (i.e., when your web server runs out of outgoing connections) and bring your entire application to its knees.

Much has been written on this:

- [You're using HttpClient wrong and it is destabilizing your software](https://www.aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/)
- [You're (probably still) using HttpClient wrong and it is destabilizing your software](https://josef.codes/you-are-probably-still-using-httpclient-wrong-and-it-is-destabilizing-your-software/)
- ]Singleton HttpClient? Beware of this serious behaviour and how to fix it](http://byterot.blogspot.com/2016/07/singleton-httpclient-dns.html)
- [Issues with the original HttpClient class available in .NET](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests) Microsoft

In practice, you probably either new up an `HttpClient` on-demand or roll your own `HttpClient` lifecycle management (beware of DNS refreshes).

### Use `IHttpClientFactory`

Microsoft: [Use IHttpClientFactory to implement resilient HTTP requests](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests)

TLDR: Register the `HttpClient`(s) as application middleware and then rely on `IHttpClientFactory` to get it for us.

Example: Say we depend on a custom API that requires a client certificate...

```cs
// .NET Framework
public static HttpClient GetHttpClientForCustomApi()
{
    var certificate = LoadX509Certificate2ForCustomApi(); // from file, blob, etc.
    var requestHandler = new WebRequestHandler();
    requestHandler.ClientCertificates.Add(certificate);
    var httpClient = new HttpClient(requestHandler);
    return httpClient;
}
```

(this code doesn't attempt to manage the request handler's lifecycle, but illustrates the point)

```cs
// .NET Core
// Register a named HttpClient as middleware in Startup.cs:
public static void AddHttpClientForCustomApi(this IServiceCollection services)
{
    var certificate = LoadX509Certificate2ForCustomApi();
    var handler = new HttpClientHandler();
    handler.ClientCertificates.Add(certificate);
    services.AddHttpClient("CustomAPI", httpClient => { })
            .ConfigurePrimaryHttpMessageHandler(() => handler);
}
// Then you can get the custom API HttpClient by name:
public static HttpClient GetHttpClientForCustomApi(IHttpClientFactory factory) =>
    factory.CreateClient("CustomAPI");
```

### Replace `HostingEnvironment` with `IWebHostEnvironment`

Example: Loading a file from the file system...

```cs
// .NET Framework
string myFilePath = HostingEnvironment.MapPath("~/App_Data/MyFile.zip");

// .NET Core
string myFilePath = Path.Combine(_webHostEnvironment.WebRootPath, "App_Data/MyFile.zip");
```

### Replace Output Caching

`[OutputCache]` and `[ContentOutputCache]` are gone.
<br/>https://docs.developers.optimizely.com/content-cloud/v12.0.0-content-cloud/docs/caching#output-caching

It is recommended to replace these with the server-side Response Caching Middleware new in ASP.NET Core (`[ResponseCache]` should feel familiar):
<br/>https://docs.microsoft.com/en-us/aspnet/core/performance/caching/middleware

- Note that this is different than plain vanilla “Response Caching”
- You must handle caching for authenticated users yourself
- Be careful about the sequence in which you call app.UseResponseCaching()
- Consider using the `<cache>` or `<distributed-cache>` tag helpers:
  <br/> https://docs.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/built-in/distributed-cache-tag-helper

### Replace `RouteTable` with `UseEndpoints`

`RouteTable` is from `System.Web` and no longer exists. Use attribute routing if you can. But if you can't...

```cs
// .NET Framework
RouteTable.Routes.MapRoute(
    "RobotsTxtRoute", "robots.txt",
    new { controller = "RobotsTxt", action = "Index" });

// .NET Core
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllerRoute(
        "RobotsTxtRoute", "robots.txt",
        new  { controller = "RobotsTxt", action = "Index" });
});
```

### Refactor async controller methods

`PageControllers` and `ContentControllers` can now be "async all the way". `BlockComponents` (f.k.a. `BlockControllers`), not so much.

```cs
// .NET Framework
public ActionResult Index(HomePage currentPage)
public ActionResult Index(StandardProduct currentContent)

// .NET Core
public async Task<ActionResult> IndexAsync(HomePage currentPage)
public async Task<ActionResult> IndexAsync(StandardProduct currentContent)
```

### Delete `SessionStateBehavior.Disabled`

In .NET Framework, MVC controllers would&mdash;by default&mdash;handle multiple incoming requests within a single session synchronously.

That is, if a user’s browser issues 3 requests at the same time, ASP.NET would execute them one at a time.

We could get around this by using the `SessionState` attribute on our controllers:

```cs
[SessionState(SessionStateBehavior.Disabled)]
public class MyController : Controller
```

In .NET Core, this asynchronous behavior is the default. So, the `SessionState` attribute is no longer needed.

### Take care when replacing `Newtonsoft.Json` with `System.Text.Json`

.NET Core introduced a performant JSON toolkit with `System.Text.Json`. It isn’t as feature rich as Newtonsoft, but it is the default (and preferred) JSON de/serializer in ASP.NET Core.

Do test any serialization that is migrated from Newtonsoft to STJ.

Examples:

- API controller requests and response models
- Anything that is indexed or projected with Opti Search & Nav
- External API client requests and responses

### Don’t get confused by authorization action filters

Use case: Triggering custom behavior when an authentication check either succeeds or fails.

Do implement both `ActionFilterAttribute` and `IAuthorizationFilter`:

```cs
public class AuthenticationRequiredAttribute : ActionFilterAttribute, IAuthorizationFilter
```

But the `OnAuthorization` signature changed slightly:

```cs
// .NET Framework
public void OnAuthorization(AuthorizationContext filterContext)

// .NET Core
public void OnAuthorization(AuthorizationFilterContext filterContext)
```

### Check virtual roles in appsettings.json

If you cannot add a new user to the WebAdmin or Administrators group in CMS Admin, check your `appsettings.json` for virtual role definitions.

### Remove VisitorGroupHelper

```cs
// CMS 11
public IEnumerable<string> GetVisitorGroupIds()
{
        var helper = new VisitorGroupHelper(_visitorGroupRoleRepository);
        foreach (var visitorGroup in _visitorGroupRepository.List())
        {
            if (visitorGroup != null && helper.IsPrincipalInGroup(PrincipalInfo.CurrentPrincipal, visitorGroup.Name))
                yield return visitorGroup.Id.ToString();
        }
}

// CMS 12
public IEnumerable<string> GetVisitorGroupIds()
{
    foreach (var visitorGroup in _visitorGroupRepository.List()?.ToList())
    {
        _visitorGroupRoleRepository.TryGetRole(visitorGroup.Name, out var visitorGroupRole);
        if (visitorGroupRole != null && visitorGroupRole.IsMatch(PrincipalInfo.CurrentPrincipal, _httpContextAccessor.HttpContext))
            yield return visitorGroup.Id.ToString();
    }
}
```

## CMS 12 on DXP

Phase 3: Upgrading the service environment

### To be continued...

> Once the codebase is upgraded to .NET 5, and everything works locally, DXP customers will need to migrate their service environment to the latest version using migration tool that will soon be available in [the portal](https://paasportal.episerver.net) (paasportal.episerver.net).

https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/upgrading-to-content-cloud-cms-12

## The End
