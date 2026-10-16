# .NET Ecosystem

## Overview of Terms

| Term | What is it? | What is it for? |
|---|---|---|
| C# | Programming language | To write source code |
| .NET (formerly .NET Core; .NET 5 and later) | Cross-platform development platform | To compile, run, and host applications |
| .NET Framework | The original, Windows-only platform (1.0 – 4.8.1) | Maintaining existing Windows apps |
| ASP.NET Core | Web framework within .NET | To create APIs, websites, and web applications |
| Visual Studio / VS Code | IDEs and editors | To program with features like IntelliSense |
| NuGet | Package manager | To add external libraries to your project |
| Entity Framework Core | .NET ORM | To work with databases in an OO fashion |

## .NET (formerly .NET Core; .NET 5 and later)

### What is it?
A runtime + library + SDKs. It is the platform that runs applications written in C# and other languages such as F#, VB.NET.

### Tools it provides:
- Compile code
- Manage memory
- Run applications
- Access files, network, databases, etc.

### Supported application types:
- **ASP.NET Core** - Web and APIs
- **WPF/WinForms** - Desktop (Windows)
- **Blazor** - Web with C#
- **.NET MAUI** - Mobile and desktop (successor to Xamarin, whose support ended May 1, 2024)
- **Console apps**
- **Azure Functions** - Serverless
- **Unity** - Games (C# scripting, but on Unity's own Mono/IL2CPP runtime today, not .NET; Unity has announced a move to CoreCLR)

## ASP.NET Core

### What is it?
A **modern, cross-platform web framework**, part of .NET.

### Allows you to create:
- REST APIs
- MVC web applications
- Web apps with Razor Pages
- SignalR (real-time apps)

### Features:
- Fast, lightweight, middleware-based
- Runs on Windows, Linux, and macOS
- Integrated with dependency injection, logging, configuration, etc.

## .NET Framework vs .NET (formerly .NET Core)

**The one-liner:** .NET Framework is the original, Windows-only runtime, now in maintenance mode as a Windows component. .NET (originally ".NET Core") is the cross-platform, open-source successor where all new development happens.

### The naming, untangled

| Milestone | What happened |
|---|---|
| .NET Framework 1.0 – 4.8.1 | The original Windows platform. *".NET Framework 4.8.1 is the latest version of .NET Framework and will continue to be distributed with future releases of Windows."* |
| June 2016: .NET Core 1.0 | A from-scratch, cross-platform rewrite, *"available on Windows, OS X and Linux"* |
| .NET 5 | *"This release was named .NET 5 instead of .NET Core 4 for two reasons: Version numbers 4.x were skipped to avoid confusion with .NET Framework 4.x. 'Core' was dropped from the name to emphasize that this is the main implementation of .NET going forward."* |
| Every November since | A new major version. *"Even numbered releases are LTS releases that get free support and patches for three years. Odd numbered releases are STS releases that get free support and patches for two years."* (STS was 18 months before .NET 9.) |

> **ASP.NET Core and EF Core kept "Core"** on purpose. ASP.NET Core 5.0 *"retains the name 'Core' to avoid confusing it with ASP.NET MVC 5."*

### Side by side

| | .NET Framework | .NET |
|---|---|---|
| Platforms | Windows only | Windows, Linux, macOS |
| Install model | A Windows component. *"only one version of .NET Framework 4 can exist on a machine at a time"*. 4.5 → 4.8 are in-place updates. | Side-by-side runtimes on one machine, plus framework-dependent or **self-contained** deployment |
| Support | 4.7 and later follow the **Windows OS lifecycle**. 4.6.2 ends Jan 12, 2027; 3.5 SP1 ends Jan 9, 2029; 4.5.2–4.6.1 were retired Apr 26, 2022. | LTS 3 years / STS 2 years per release |
| Evolution | Servicing only (*"security and reliability bug fixes"*). Microsoft said in 2019 that 4.8 *"will be the last major version"*. | New major version every year |
| Web | ASP.NET (Web Forms, MVC 5, Web API 2) | ASP.NET Core. *"ASP.NET Core 3.0 or later only run on .NET Core."* |
| Desktop | WPF, Windows Forms | WPF, Windows Forms, still Windows-only: *"Even though .NET is a cross-platform technology, WPF only runs on Windows."* |
| Performance | — | *"When your system needs the best possible performance and scalability, .NET and ASP.NET Core are your best options."* |

### What does NOT exist on modern .NET

From Microsoft's porting docs. This is the list that decides how hard a migration is:

| .NET Framework technology | On .NET | Replacement |
|---|---|---|
| Multiple **AppDomains** | Creating more isn't supported | Separate processes or containers; `AssemblyLoadContext` for loading/unloading assemblies |
| **.NET Remoting** | Not supported | Pipes, `MemoryMappedFile`, StreamJsonRpc, HTTP, sockets |
| **Code Access Security** / security transparency | *"no longer treated as a security boundary"* | OS boundaries: *"virtualization, containers, or user accounts"* |
| **System.EnterpriseServices** (COM+) | Not supported | — |
| **Workflow Foundation** | Not supported | CoreWF (porting docs) or Elsa Workflows (.NET 5 docs) |
| **ASP.NET Web Forms** | *"only available in .NET Framework"* | Blazor or Razor Pages |
| **WCF server** | Not in the box | **CoreWCF**: *"As of April 2022, CoreWCF is officially supported by Microsoft."* |

> **Hidden landmine:** delegate `BeginInvoke()` / `EndInvoke()` relied on Remoting. On .NET they *"will throw PlatformNotSupportedException"*. Replace them with `Task.Run`.

### .NET Standard: the bridge between them

*".NET Standard is a formal specification of .NET APIs that are available on multiple .NET implementations."* It still matters for one reason: **sharing a library with .NET Framework consumers**.

- *"if you want to share code between .NET Framework and any other .NET implementation... your library should target .NET Standard 2.0."*
- *".NET Framework doesn't support .NET Standard 2.1"*, and *"There will be no new .NET Standard versions after 2.1."*
- Practical shape for a shared library: multi-target, so modern apps get the optimized build and .NET Framework apps fall back to `netstandard2.0`:

```xml
<PropertyGroup>
  <TargetFrameworks>net10.0;netstandard2.0</TargetFrameworks>
</PropertyGroup>
```

### When to stay on .NET Framework, and how to leave

Microsoft's server guidance: *"The latest .NET version (currently .NET 10) is the preferred version of .NET to use for server development. The reasons to continue using .NET Framework are specific and limited."* Those reasons:

- The app already runs on .NET Framework. The docs: *"In most cases, you don't need to migrate"* a working app.
- It depends on third-party libraries or NuGet packages with no .NET version.
- It uses technologies from the table above that have no replacement you can adopt yet.
- It runs on a platform that doesn't support .NET.

Migration tooling changes fast. **.NET Upgrade Assistant is officially deprecated** in favor of a GitHub Copilot upgrade agent in Visual Studio. Its product name has changed more than once, so check the current docs. The older `try-convert` tool is archived. A common incremental path for web apps:

1. Move shared code to `netstandard2.0` or multi-target.
2. Migrate leaf services first.
3. Run old and new side by side behind a proxy.

> **Interview answer in 30 seconds:** ".NET Framework is Windows-only, installed as part of Windows, and only gets fixes. 4.8.1 is the latest version, and Microsoft said 4.8 would be the last major one. .NET (Core) is cross-platform, open source, deploys side-by-side or self-contained, ships every November, and is faster. New code goes on .NET. Existing Framework apps stay there unless they need what's new, and the hard parts of migrating are AppDomains, Remoting, WCF server and Web Forms."

---

[Back to index](README.md) | [Next: Namespace →](02-namespace.md)
