
# The netstandard2.0 Anchor: How the NuGet Ecosystem's Greatest Compatibility Promise Became Its Biggest Fragmentation Problem

You've seen this a hundred times. You pull up a NuGet package, check its dependencies, and there it is — `netstandard2.0`. That little target framework moniker has been the backbone of .NET package compatibility for nearly a decade. And that's precisely the problem.

## The Promise

When Microsoft shipped netstandard2.0 in 2017, it was genuinely elegant. One target framework. 32,000+ APIs. Write your library once, and it runs on .NET Framework 4.6.1+, .NET Core 2.0+, Xamarin, Mono, and Unity. For the first time, library authors didn't have to ship a dozen different TFMs or maintain labyrinthine `#if` directives. The ecosystem unified around a single compatibility surface.

And it worked. Spectacularly. Today, over 65% of the top 1,000 NuGet packages still target netstandard2.0. It's the safest bet in the game. Ship on netstandard2.0 and you cover everything from legacy ASP.NET on Windows Server to containerized .NET 9 microservices on ARM64 Linux.

## The Trap

That 65% number isn't a triumph. It's a symptom.

netstandard2.0 is frozen. Permanently. It will never get `Span<T>`. It will never get `IAsyncEnumerable<T>`. It will never get `System.Text.Json`, pipeline-based HTTP, or half the performance primitives that make modern .NET what it is. The standard was the bridge from Framework to Core, and that bridge has a hard stop.

Library authors are now stuck in a vise. You can target netstandard2.0 and reach everyone — but you're coding against a 2017 API surface, shipping polyfills, and explaining why your performance numbers look nothing like the benchmarks blog posts promise. Or you can target `net8.0` / `net9.0`, use Span-ified APIs, drop your allocations by 40%, and then field GitHub issues from the 30% of your users still on .NET Framework 4.8 who get `NU1202: Package is not compatible`.

## The Real Fragmentation Nobody Talks About

We obsess over the Framework-to-Core migration as if it were a binary transition. It wasn't. What actually happened is the ecosystem fractured into at least four compatibility zones:

- **Zone 1: netstandard2.0 purists.** Libraries that target _only_ netstandard2.0. Maximum reach, frozen feature set. These packages will compile against anything and take advantage of nothing.

- **Zone 2: Multi-targeters.** The `netstandard2.0;net8.0` crowd. They `#if NET8_0_OR_GREATER` their way to partial modernity, shipping two implementations in one package. It works, but it doubles the test matrix, complicates CI, and produces subtle behavioral differences between targets that surface as "works on my machine" bugs.

- **Zone 3: Forward-only.** Packages targeting `net8.0` (or `net9.0`) exclusively. Clean code, modern APIs, no polyfills. They've accepted that .NET Framework users won't be customers — a defensible position in 2026, but one that fragments the ecosystem every time someone forks a popular Zone 1 library into a Zone 3 rewrite.

- **Zone 4: The .NET Framework long tail.** Enterprise packages still targeting `net48` because their customers run on-prem IIS and upgrading isn't on the 2026 roadmap. These libraries often dual-target netstandard2.0, but the Framework-specific code paths are where the bugs live.

Four zones, one ecosystem. And the anchor that created this fragmentation was the very thing designed to prevent it.

## What This Costs You

If you're building a new .NET library in 2026, you'll spend your first hour not writing code but answering the targeting question. That's architectural overhead that didn't exist before netstandard — back when everyone just targeted a single framework version and moved on.

If you're consuming packages, you've learned to check the TFM column on NuGet.org before installing. You've memorized which of your dependencies are netstandard2.0 vs. net8.0 and mentally traced compatibility chains. You've debugged `MissingMethodException` at runtime because a netstandard2.0 library got loaded into a net8.0 process with a different `System.Memory` resolution path.

This is not theoretical. This is Tuesday.

## The Path Forward (No, Really)

The solution isn't "everyone upgrade to .NET 9." That's a fantasy that ignores enterprise procurement cycles, regulatory lock-in, and the fact that some .NET Framework applications run factories, hospitals, and payment systems.

Here's what I actually recommend:

**For library authors shipping today:** Multi-target `netstandard2.0` and `net8.0`. Yes, it's more work. But `#if NET8_0_OR_GREATER` blocks let you use Span-ified paths for modern consumers while maintaining Framework compatibility. Just be disciplined about your `#if` surface — if more than 15% of your code is conditional, you're doing it wrong.

**For internal / greenfield libraries:** Target `net9.0` only. You don't need Framework compatibility for code that only runs inside your organization's Kubernetes cluster. Stop paying the netstandard tax on internal packages.

**For tooling:** Microsoft, give us a Roslyn analyzer that detects when a netstandard2.0 library could benefit from a `NET8_0_OR_GREATER` fast path and suggests the refactor. The fragmentation won't fix itself, but tooling can make multi-targeting less of a burden.

**For the community:** Treat netstandard2.0 as a compatibility floor, not a target. If your library is on netstandard2.0 and has no conditional code for modern TFMs, you're not "compatible" — you're leaving performance on the table for every single one of your users.

---

The netstandard2.0 anchor gave us a decade of unprecedented .NET ecosystem compatibility. The next decade's challenge is undoing the fragmentation that compatibility created — without breaking everything that still depends on it.

That's not a framework problem. That's an architecture problem. And architects solve architecture problems.
