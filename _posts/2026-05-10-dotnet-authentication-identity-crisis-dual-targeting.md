
# The Authentication Identity Crisis: How .NET's Dual-Target Promise Fractured the NuGet Ecosystem

On r/dotnet this week, someone asked the question every library author wrestles with: "Can I drop netstandard2.0 and net48 targets in 2026?" The top answer was brutal in its honesty — only if you're willing to lose 40% of your NuGet downloads overnight. That's not a migration timeline. That's a hostage situation.

## The Promise of Multi-Targeting

When .NET Core shipped, multi-targeting was sold as the bridge. Write your code once, add `<TargetFrameworks>netstandard2.0;net8.0</TargetFrameworks>` to your csproj, and the build system produces packages that work everywhere. .NET Framework 4.8 on Windows Server? Covered. .NET 9 on ARM64 Linux? Covered. The NuGet resolver picks the best asset for each consumer, and everyone wins.

It worked well enough that even the .NET Foundation packages themselves — `Microsoft.Extensions.*`, `System.Text.Json` — ship with dual targets. If it's good enough for Microsoft, it's good enough for you, right?

## The Reality

Wrong. Multi-targeting created a hidden cost center that compounds with every dependency you add.

When your application references Package A (which multi-targets netstandard2.0 and net8.0) and Package B (which targets only net8.0), NuGet's dependency resolver enters a combinatorial resolution path that produces error messages even senior engineers spend hours decoding. You'll see `NU1605: Detected package downgrade` when there is no downgrade. You'll get `NU1202` with a stack trace that implicates a transitive dependency four levels deep that you've never heard of.

The resolver isn't broken. It's doing exactly what it was designed to do — navigating a compatibility graph with explicit assets for each TFM. The problem is that the graph is fundamentally unresolvable in certain configurations because netstandard2.0 and net8.0 share API surface names but resolve to different assemblies at runtime.

## The Split Ecosystem Nobody Designed

What emerged isn't a bridge. It's a split. The .NET ecosystem now operates across four incompatible dependency zones:

- **Zone 1: netstandard2.0 only.** Maximum reach, frozen API surface. These packages run on everything and optimize for nothing. They're the compatibility tax that 65% of the top 1,000 NuGet packages pay.

- **Zone 2: netstandard2.0 + net8.0.** The multi-targeters. Two code paths, `#if NET8_0_OR_GREATER` blocks, doubled test matrices. These packages technically work everywhere but have subtle behavioral differences between targets. You will discover them in production.

- **Zone 3: net8.0/net9.0 only.** Forward-only libraries. Clean code, Span-ified APIs, zero polyfills. But the moment a Zone 1 package depends on a Zone 3 package, the entire dependency chain breaks for Framework consumers.

- **Zone 4: The Framework long tail.** Enterprise libraries still targeting `net48` because their customers run on-prem IIS and upgrading isn't on the 2026 roadmap. These packages often dual-target netstandard2.0, but the Framework-specific code paths are where the bugs live.

Four zones. One ecosystem. And every new library you introduce creates a transitive edge that crosses at least two of them.

## Why This Is a Senior-Level Problem

Junior developers add packages and move on. Senior developers know that every NuGet dependency is a compatibility bet placed against an unknown consumer's runtime environment.

That `Microsoft.Extensions.Logging` reference? If your consumer is on .NET Framework 4.8 with binding redirects, and one of your dependencies pulled in a net8.0-only logging abstraction through a transitive chain, the runtime will throw `FileLoadException` with a fusion log that spans 200 lines.

The real cost isn't the debugging session. It's that these problems don't surface at build time. They surface in staging, or worse, in production, when a consumer deploys to an environment with a different .NET runtime than your CI pipeline tested against.

## What Actually Works

**For library authors in 2026:** If your user base includes .NET Framework consumers, multi-target `netstandard2.0` and `net9.0`. Keep conditional code under 15% of your surface area. Add a CI leg that runs your test suite against both target frameworks. If you don't test both paths, you're shipping bugs you haven't met yet.

**For internal and greenfield libraries:** Target `net9.0` exclusively. You don't need Framework compatibility for code that only runs inside your organization's Kubernetes cluster. Stop paying the multi-target tax on internal packages.

**For application developers:** Audit your dependency tree before adding packages. Run `dotnet list package --include-transitive` and look for the TFM column. If a critical dependency is netstandard2.0-only with no signs of modernization, start planning your exit now — not when the `MissingMethodException` hits.

## The Bottom Line

Multi-targeting was supposed to prevent ecosystem fragmentation. Instead, it legitimized it. By making it possible to ship the same package to completely different runtimes, it gave library authors permission to defer the hard decision about which runtime to actually support — and pushed that cost downstream to every consumer.

That's not a tooling failure. That's an architecture failure. And in 2026, the architects who understand the dependency graph are the ones keeping production from catching fire.
