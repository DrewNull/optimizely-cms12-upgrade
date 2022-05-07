# Upgrading Optimizely CMS 11 to 12 and Commerce 13 to 14

[[_TOC_]]

## Background

There are many great resources for learning how to build a new solution using
CMS 12 and Commerce 14. The official [developer documentation](https://world.optimizely.com)
has been updated, the official [user guide](https://webhelp.optimizely.com) has
been updated, and an excellent [masterclass](https://www.optimizely.com/support/education/product/migrating-to-optimizely-cms-12-and-commerce-14)
is hosted by Mark Price and Scott Reed (to name a few). But there isn't much
information on how to take an existing CMS 11 / Commerce 13 solution and upgrade
it to .NET 5+.

This blog post is not intended to be the definitive guide for ugprading an
existing solution to .NET 5+, but rather a collection of learnings from
misadventures in upgrading two Commerce 13 solutions to date.

## Prerequisites

Make sure that your solution is ready to upgrade.

### 1. Read the official documentation

1. [Upgrading to Content Cloud (CMS 12)](https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/upgrading-to-content-cloud-cms-12)
2. [Breaking changes in Content Cloud (CMS 12)](https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/breaking-changes-in-content-cloud-cms-12)
3. [System requirements for Optimizely (CMS 12)](https://docs.developers.optimizely.com/content-cloud/v12.0.0-content-cloud/docs/system-requirements-for-optimizely)

### 2. Be on .NET Framework 4.7.2 or higher

CMS 11 only [requires](https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/system-requirements-for-optimizely)
.NET Framework 4.6.1, but Microsoft [recommends](https://docs.microsoft.com/en-us/dotnet/core/porting/premigration-needed-changes)
being on 4.7.2 or higher when using Upgrade-Assistant

### 3. Update to the latest version of CMS11 (Commerce13) before upgrading

The official documentation doesn't explicitly say to do this, but is there any
reason _not_ to?

### 4. Check the status of add-on packages

Optimizely maintains a list of the .NET 5 migration status of the official
platform and addon NuGet packages:

- [Add-ons platform compatibility (Optimizely Developer Docs)](https://docs.developers.optimizely.com/integrations/v1.1.0-apps-and-integrations/docs/add-ons-platform-compatibility)
- [Add-Ons Status (Optimizely.com)](https://world.optimizely.com/resources/net5/add-ons)

No such list exists for unofficial add-ons (as of this writing). So, when
planning the upgrade, give yourself time to check the status of your favorite
third party add-ons. Having no workaround for unsupported add-ons could derail
your whole upgrade project. Know what you’re getting into.

Note that some old, .NET Framework add-ons will still work, just with a warning.
For example: Authorize.Net. There is no .NET Core+ package, but it still compiles
and runs when you install it to your .NET 5+ solution via NuGet.

## Upgrade-Assistant

Once you have reviewed the prerequisites and your solution is ready, it's time
to start making changes. The [.NET Upgrade-Assistant](https://dotnet.microsoft.com/en-us/platform/upgrade-assistant)
is Microsoft's CLI tool for upgrading .NET Framework solutions to .NET 5+.

Read and bookmark the official Optimizely documentation: [Upgrade Assistant](https://docs.developers.optimizely.com/content-cloud/v12.0.0-content-cloud/docs/upgrade-assistant)

**Important**: The following steps, in this Upgrade-Assistant heading, should be
conducted in the sequence that they are listed below.

### 5. Delete Commerce Manager

As of Commerce 14, Commerce Manager is no more. 👏

Remove the Commerce Manager project before getting started with Upgrade-Assistant.
Take note that some of Commerce Manager’s functionality hasn’t been ported over
to the CMS yet and can only be done with APIs:

- Importing and exporting catalogs
- Adding countries and regions
- Adding currencies
- Working with business objects
- Working with catalog and order meta classes and fields

### 6. Delete `node_modules`

As a first step, Upgrade-Assistant copies all files in your solution/project
folder into a Backup directory. If you have NPM's `node_modules` directory in
the solution you are upgrading, you _probably_ want to delete it first so you
don't have to sit around waiting for it to be backed up. Upgrade-Assistant's
backup step can be disabled, but to play it safe, delete your `node_modules`
folder before moving forward.

### 7. Use Opti’s Upgrade-Assistant-Extensions

Upgrade-Assistant can be extended to automatically execute additional commands.
Optimizely has a public GitHub repo for their own Upgrade-Assistant extensions
which provides some Opti-specific capabilities:

- String Replacement
- Remove Default Argument for the TemplateDescriptor Attribute
- Base Class Mapping
- Replace IFindUIConfiguration with FindOptions
- Remove PropertyData ParseToObject method overrides
- Remove obsolete using statements like Mediachase.BusinessFoundation
- Type Mapping like EPiServer.Web.Routing to EPiServer.Core.Routing

Additionally, NuGet packages can be specified, and templates for `Program.cs` and
`Startup.cs` (required by .NET 5+) can be added as well.

Read how it works on GitHub (there are a couple gotchas): [Upgrade Assistant Extensions](https://github.com/episerver/upgrade-assistant-extensions).
Check the [Releases](https://github.com/episerver/upgrade-assistant-extensions/releases)
page to learn what the configuration options are and how to use them.

Note that, although Ugrade-Assistant-Extensions will do some nice things for you
out of the box (e.g., replace `BlockController`s with `BlockComponent`s), do
expect to spend time customizing the config for string/type/class replacements.

How to get it ready:

1. Download the latest release (Epi.Source.Updater.X.Y.Z.zip): [Releases](https://github.com/episerver/upgrade-assistant-extensions/releases)
2. Unzip it to your local file system, such as `C:\Temp\Epi.Source.Updater\`.
3. Make your preferred configuration changes.

### 8. Make a plan-of-attack before running Upgrade-Assistant

Upgrade-Assistant can run against a Solution file (`.sln`) or Project file (`.csproj`).
If you run it against the Solution, it is smart enough to analyze your project
dependency tree and execute against one project at a time, in order, starting with
those that has no project dependencies.

For example, consider a fictitious onion architecture inspired `MySolution.sln`.
If you run Upgrade-Assistant against the `.sln`, it will execute against each
project in this order:

1. `MySolution.Domain.csproj`
   - Depends on nothing
2. `MySolution.Application.csproj`
   - Depends on Domain
3. `MySolution.Web.csproj`
   - Depends on Application, which depends on Domain

### 9. Consider doing one CSPROJ at a time

Upgrade-Assistant will track progress and start where it left off if you cancel
it at any time. But&mdash;do figure out the dependency sequence first and
consider running UA manually against each Project. This will allow you to
resolve code issues in isolation on a per-Project basis without getting confused
about where you are with UA. Especially if you find yourself mindlessly jamming
that Enter key while it runs.

For example,

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

### 10. Think about which flags to use

Upgrade-Assistant has a number of flags that can modify execution behavior.

The UA basic syntax, if your terminal is at the solution root, is the following:

```cli
upgrade-assistant upgrade MySolution.Web/MySolution.Web.csproj --flags-go-here
```

Consider using the following flags:

`--extension "c:\temp\epi.source.updater"`<br/>`--target-tfm-support LTS`
<br/>These two flags enable Opti’s Upgrade-Assistant-Extensions.

`--ignore-unsupported-features`
<br/>This is required for upgrading the web application CSPROJ.

`--skip-backup`
<br/>Without this, UA will copy all solution files into `/Backup/` first (RE:
deleing `node_modules`). But... don’t you have source control?

`--non-interactive`
<br/>Officially: Microsoft’s documentation says that Upgrade-Assistant is meant
to be interactive, and that you should think twice about using this flag.
<br/>Unofficially: If you don’t use this flag, you will be sitting at your
keyboard, pressing Enter repeatedly, for hours.

### 11. Install and update Upgrade-Assistant

To install Upgrade-Assistant globally on your local machine, open a terminal
from anywhere and enter the following:

```cli
dotnet tool install -g upgrade-assistant
dotnet tool update -g upgrade-assistant
```

### 12. Run Upgrade-Assistant

If you've made it this far, you're finally ready to run Upgrade-Assistant.

From a terminal in your solution root (recommended):

```cli
set DefaultTargetFrameworks__LTS=net5.0
```

This ☝ is required by Upgrade-Assistant-Extensions.

Then, with the framework set, enter:

```cli
upgrade-assistant upgrade MySolution.Web/MySolution.Web.csproj
    --ignore-unsupported-features
    --skip-backup
    --non-interactive
    --extension "c:\temp\epi.source.updater"
    --target-tfm-support LTS
```

This ☝ is written on multiple lines for readability, not for copy-paste.

At this point, Upgrade-Assistant starts doing its thing.

### 13. Wait

Upgrade-Assistant can take from several minutes to, depending on the size of
your solution, hours.

### 14. Review the code changes

Here are some of the changes you should expect to see when upgrading your web
application solution project.

`+ Properties/launchSettings.json`
<br/>Local server/IIS Express settings. Note that .NET5+ runs on HTTPS by default!

`+ appsettings.Development.json`
<br/>`+ appsettings.json`
<br/>Where your Web.config appSettings and connectionStrings went. TBD on
guidance from the DXP team...

`- packages.config`
<br/>Packages are now referenced in the CSPROJ files.

`+ Program.cs`
<br/>`+ Startup.old.cs`
<br/>Program.cs and Startup.cs will need to be ported over. Look at Foundation
for inspiration: [GitHub](https://github.com/episerver/Foundation/tree/main/src/Foundation).

### 15. Commit the broken code

Be sure to commit the code at this stage, even though it is broken. This way, if
your code fixes go sideways, you can easily go back to the state immediately
after running the Upgrade-Assistant.

Do check in `.upgrade-assistant`. This is where UA internally tracks its own
progress, allowing it to pick up where it left off if you need to shut down along
the way. Jot down a reminder to delete this file once the upgrade is complete.
It is not needed by the solution in any way.

Make a mental note to commit frequently from this point on. Comitting progress on
code fixes along the way can be a lifesaver.

### 16. Delete leftover assembly dependencies

Some .NET Framework System assemblies will not have corresponding packages and
will get orphaned in the Dependencies > Assemblies node.Unless any of these were
explicitly added by your implementation, you should be free to delete them.

### 17. Uninstall obsolete NuGet packages

Some `EPiServer` packages will need to be replaced entirely (i.e., removed and
replaced with something else). These are listed in the official documentation:
[Breaking changes in Content Cloud (CMS 12)](https://docs.developers.optimizely.com/content-cloud/v12.0.0-content-cloud/docs/breaking-changes-in-content-cloud-cms-12).

In summary, the following `EPiServer` packages must be uninstalled:

- `EPiServer.CMS.AspNet`
- `EPiServer.Framework.AspNet`
- `EPiServer.ServiceLocation.StructureMap`
- `EPiServer.Logging.Log4Net`

NuGet Package Manager reveals, to a sharp eye, which packages must be removed.
For example:

- The latest version of `EPiServer.CMS.AspNet` is `11.x`, so you know this one
  must be replaced.
- But the latest version of, say, `EPiServer.CMS.UI.AspNetIdentity` is `12`+, so
  you know this can be updated.

### 18. Manually resolve package errors

If you've made it this far, the following error will have probably started
plaguing your attempts to build the solution:

> NU1177: Version conflict detected for Xyz. Install/reference Xyz.1.2.3 directly
> to project MySolution.Web to resolve this issue.

It is unclear to me _why_ this happens, but

Here is what Microsoft says about this error: [NuGet Error NU1107](https://docs.microsoft.com/en-us/nuget/reference/errors-and-warnings/nu1107).

> **Issue**<br/>
> Unable to resolve dependency constraints between packages. Two different
> packages are asking for two different versions of 'PackageA'. The project
> needs to choose which version of 'PackageA' to use.
>
> **Solution**<br/>
> Install/reference 'PackageA' directly (in the project file) with the exact
> version that you choose. Generally, picking the higher version is the right choice.

In other words, this error can be addressed by doing the following for each
package that Visual Studio complains about:

1. Open your new CSPROJ file (double-click the project in Solution).
2. Find where all the `<PackageReference />` elements are.
3. Manually add the package reference it is complaining about, e.g.,
   <br/>`<PackageReference Include="Xyz" Version="1.2.3" />`

This manual process might result in your Project(s) taking on dependencies that
aren't actually needed. When the upgrade is complete, go through each Project's
dependencies and clean out the ones that are unused.

[ReSharper](https://www.jetbrains.com/resharper/)
has an Optimize References tool that can help you explore whether and how each
dependency is used. Right-click on a Project's Packages node (under its
Dependencies node) to access this tool.

When doing this, becareful not to delete the central `EPiServer` product pacakges
from your web application Project, such as `EPiServer.CMS`, `EPiServer.Commerce`,
`EPiServer.Find`, `EPiServer.Forms`, etc.

### 19. Update NuGet packages

At this point, the Project should be ready for updating its NuGet packages.

As mentioned above, there are a couple `EPiServer` packages that must be _replaced_:

- `EPiServer.Framework.AspNet` should be replaced with `EPiServer.Framework.AspNetCore`.
- `EPiServer.CMS.AspNet` should be replaced with the following:
  - `EPiServer.CMS.AspNetCore`
  - `EPiServer.CMS.AspNetCore.Templating`
  - `EPiServer.CMS.AspNetCore.Routing`
  - `EPiServer.CMS.AspNetCore.Mvc`
  - `EPiServer.CMS.AspNetCore.HtmlHelpers`

### 20. Address known breaking changes

There are too many `EPiServer` breaking changes to list here. If you haven't
already, go through the official breaking changes documentation and make sure
each is taken care of before moving on: [Breaking Changes in Content Cloud (CMS 12)](https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/breaking-changes-in-content-cloud-cms-12).
It is a dense read, but worth it.

## Code Fixes

The following section is a list of commonly-encountered code fixes. **This is not
an exhaustive list** (obviously). Much of this content is about replacing
`System.Web`, which was removed in .NET Core. But there are some Opti-specific
topics too.

### 21. Replace `HttpContextHelper` with `IHttpContextAccessor`

`HttpContext.Current`, which `EPiServer` solutions tend to use liberally, was
removed in .NET Core. Because this is such a common issue, Upgrade-Assistant
automatically creates and adds an `HttpContextHelper` static helper class, which
provides a static means of accessing the current `HttpContext.

Although better than nothing, this does not obey [SOLID](https://en.wikipedia.org/wiki/SOLID).
In .NET Core, the `IHttpContextAccessor` abstraction was introduced, which can
be injected by default in ASP.NET Core, and has an `HttpContext` member
property that provides best-practice access to the current request's context.

For example:

```cs
// .NET Framework
string myCookie = HttpContext.Current.Request.Cookies[CookieNames.PostalCode]?.Value;

// .NET Core
string myCookie = _httpContextAccessor.HttpContext?.Request.Cookies["MyCookie"];
```

### 22. Give yourself time to replace HttpRequest

Microsoft reimagined the `HttpRequest` concept in .NET Core. Most of the legacy
`System.Web` capability is still present, but in many cases has been reorganized
to better conform to web and HTTP standards. Because use of the `HttpRequest`
object is critical to ASP.NET solutions, expect to spend a nontrivial amount of time
fixing compiler errors due to `HttpRequest` changes.

For example:

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

### 23. Use `IHttpClientFactory`

We need to talk about `HttpClient`.

Managing the lifecycle of `HttpClient` in .NET Framework was always a pain. The
central point of confusion is that `HttpClient` implements `IDisposable`, but
that putting it in a `using` statement can lead to SNAT port exhaustion (i.e.,
when your web server runs out of outgoing connections and stops processing
incoming requests until they free up) and bring your entire application to its knees.

Much has been written on this:

- [You're using HttpClient wrong and it is destabilizing your software](https://www.aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/)
- [You're (probably still) using HttpClient wrong and it is destabilizing your software](https://josef.codes/you-are-probably-still-using-httpclient-wrong-and-it-is-destabilizing-your-software/)
- [Singleton HttpClient? Beware of this serious behaviour and how to fix it](http://byterot.blogspot.com/2016/07/singleton-httpclient-dns.html)
- [Issues with the original HttpClient class available in .NET](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests)
  (Microsoft)

In practice, most .NET Framework solutions that use `HttpClient` either new up
`HttpClient`s on-demand, or _carefully_ roll their own DI-friendly management of
the `HttpClient` lifecycle (more specifically, the underlying request handler
object which is the true source of the problem).

Fortunately, .NET Core introduced `IHttpClientFactory`, which makes these
problems go away: [Use IHttpClientFactory to implement resilient HTTP requests](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/implement-resilient-applications/use-httpclientfactory-to-implement-resilient-http-requests).
Once configured, `IHttpClientFactory` can be injected and used to access a safe
and reliable instance of `HttpClient`. Multiple `HttpClients` can be registered
per application by giving them names.

This is a two-step process:

1. Register the `HttpClient`(s) as application middleware in `Startup.cs`
2. Inject `IHttpClientFactory` where ever `HttpClient` is needed

Example: Say we depend on a custom API that requires a client certificate...

In .NET Framework, the `HttpClient` might be newed-up on demand, like this:

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

But in .NET Core, one must first register the `HttpClient` as middleware first,
and then access it via `IHttpClientFactory`, like this:

```cs
// .NET Core
// Register a named HttpClient as middleware in Startup.cs:
public static void AddHttpClientForCustomApi(this IServiceCollection services)
{
    var certificate = LoadX509Certificate2ForCustomApi(); // from file, blob, etc.
    var handler = new HttpClientHandler();
    handler.ClientCertificates.Add(certificate);
    services.AddHttpClient("CustomAPI", httpClient => { })
            .ConfigurePrimaryHttpMessageHandler(() => handler);
}
// Then you can get the custom API HttpClient by name:
public static HttpClient GetHttpClientForCustomApi(IHttpClientFactory factory) =>
    factory.CreateClient("CustomAPI");
```

### 24. Replace `HostingEnvironment` with `IWebHostEnvironment`

`IWebHostEnvironment`, introduced in .NET Core, should be used to replace
`HostingEnvironment`. One common use case is for accessing the file system:

```cs
// .NET Framework
string myFilePath = HostingEnvironment.MapPath("~/App_Data/MyFile.zip");

// .NET Core
string myFilePath = Path.Combine(_webHostEnvironment.WebRootPath, "App_Data/MyFile.zip");
```

### 25. Replace Output Caching

Microsoft's `[OutputCache]` and Optimizely's `[ContentOutputCache]` have been
removed. Opti no longer has its own wrapper around output caching, and instead
recommends to use the new .NET Core types: [Output Caching](https://docs.developers.optimizely.com/content-cloud/v12.0.0-content-cloud/docs/caching#output-caching).

It is best practice to replace .NET Framework output caching with the
server-side Response Caching Middleware new in ASP.NET Core:
[Response Caching Middleware in ASP.NET Core](https://docs.microsoft.com/en-us/aspnet/core/performance/caching/middleware).
The `[ResponseCache]` attribute should feel familiar.

- Note that this is different than .NET Core's plain-vanilla "Response Caching" concept
- RCM does not account for whether the user is authenticated, unlike .NET
  Framework output caching. This is a significant departure from how output
  caching worked before. Do consider this when rendering content that could vary
  by user.
- Be careful about the sequence in which you call `app.UseResponseCaching()`. It
  cannot be invoked before `app.UseCors()`.
- ASP.NET Core introudces a new caching tool called [Cache Tag Helpers](https://docs.microsoft.com/en-us/aspnet/core/mvc/views/tag-helpers/built-in/distributed-cache-tag-helper).
  This is represented as two new Razor elements, `<cache>` and `<distributed-cache>`,
  which facilitates the caching of server-rendered markup from within Razor, and
  can be used to achieve donut caching. Although Optimizely does not have
  official helpers to manage `vary-by`, it's easy to imagine an extension for
  `<distributed-cache>` that uses Opti's `ISynchronizedObjectInstanceCache`
  under the hood.

### 26. Replace `RouteTable` with `UseEndpoints`

`RouteTable` is from `System.Web` and no longer exists. Optimizely controllers
are automatically routed by the CMS, and custom API controllers _should_ use attribute
routing. But there are some scenarios where custom routes will need to be manually
registered. ASP.NET Core introduces `app.UseEndpoints()` to register custom routes
as middleware.

Example:

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

### 27. Use async controller methods

Although there probably aren't many use cases for it, `PageControllers` and
`ContentControllers` (Commerce) can now be "async all the way". Note that
partial content controllers, such `BlockComponents` (f.k.a. `BlockControllers`),
must still be synchronous.

```cs
// .NET Framework
public ActionResult Index(HomePage currentPage) {}
public ActionResult Index(StandardProduct currentContent) {}

// .NET Core
public async Task<ActionResult> IndexAsync(HomePage currentPage) {}
public async Task<ActionResult> IndexAsync(StandardProduct currentContent) {}
```

### 28. Delete `SessionStateBehavior.Disabled`

In the .NET Framework, MVC controllers would&mdash;by default&mdash;handle
multiple incoming requests within a single session synchronously. That is, if a
user’s browser where to issue, say, 3 requests at the same time, ASP.NET would
execute them one at a time in a FIFO sequence. At scale, this would lead to
performance issues because the browser would be held in limbo as ASP.NET took its
time processing one request at a time.

This could be mitigated using the `SessionState` attribute on controllers, which
explicitly tells ASP.NET to process requests asynchronously:

```cs
[SessionState(SessionStateBehavior.Disabled)]
public class MyController : Controller
```

In .NET Core, however, this asynchronous behavior is the default. So, the
`SessionState` attribute is no longer needed&mdash;in fact, it no longer exists&mdash;
and must be removed.

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
        if (visitorGroup != null
            && helper.IsPrincipalInGroup(PrincipalInfo.CurrentPrincipal, visitorGroup.Name))
        {
            yield return visitorGroup.Id.ToString();
        }
    }
}

// CMS 12
public IEnumerable<string> GetVisitorGroupIds()
{
    foreach (var visitorGroup in _visitorGroupRepository.List()?.ToList())
    {
        _visitorGroupRoleRepository.TryGetRole(visitorGroup.Name, out var visitorGroupRole);
        if (visitorGroupRole != null
            && visitorGroupRole.IsMatch(PrincipalInfo.CurrentPrincipal, _httpContextAccessor.HttpContext))
        {
            yield return visitorGroup.Id.ToString();
        }
    }
}
```

### Replace ImageProcessor with ImageSharp

TBD

## CMS 12 on DXP

Phase 3: Upgrading the service environment

### To be continued...

> Once the codebase is upgraded to .NET 5, and everything works locally, DXP customers will need to migrate their service environment to the latest version using migration tool that will soon be available in [the portal](https://paasportal.episerver.net) (paasportal.episerver.net).

https://docs.developers.optimizely.com/content-cloud/v11.0.0-content-cloud/docs/upgrading-to-content-cloud-cms-12

## The End
