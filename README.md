# nuget-multi-platform

## Feature exercised

Multi-targeting with platform-conditional `PackageReference`: a single `.csproj` targets both `net8.0` and `net8.0-windows`, with three unconditional cross-platform dependencies and two Windows-only dependencies gated behind an `ItemGroup Condition` on the active `TargetFramework`.

## Probe metadata

| Field         | Value                                              |
|---------------|----------------------------------------------------|
| Pattern       | multi-target-framework with platform-conditional refs |
| Package manager | NuGet (SDK-style csproj)                         |
| Target frameworks | `net8.0`, `net8.0-windows`                    |
| Lockfile      | `packages.lock.json` (version 2, one TFM section per framework) |
| Generated     | 2026-04-22                                         |

## File layout

```
nuget-multi-platform/
├── src/
│   └── MultiPlatformProbe/
│       ├── MultiPlatformProbe.csproj
│       └── packages.lock.json
├── README.md
└── expected-tree.json
```

## Dependencies declared

### Cross-platform (unconditional — both TFMs)

| Package                         | Version  |
|---------------------------------|----------|
| Newtonsoft.Json                 | 13.0.3   |
| Serilog                         | 3.1.1    |
| Microsoft.Extensions.Logging    | 8.0.0    |

### Windows-only (Condition: `'$(TargetFramework)' == 'net8.0-windows'`)

| Package                                  | Version  |
|------------------------------------------|----------|
| CommunityToolkit.Mvvm                    | 8.2.2    |
| Microsoft.Extensions.DependencyInjection | 8.0.0    |

## Expected dependency tree

Mend must detect two separate TFM-scoped dependency sets. Key expectations:

### Under `net8.0`

- Direct: `Newtonsoft.Json 13.0.3` (no children)
- Direct: `Serilog 3.1.1` (no children)
- Direct: `Microsoft.Extensions.Logging 8.0.0`
  - Transitive: `Microsoft.Extensions.DependencyInjection.Abstractions 8.0.0`
  - Transitive: `Microsoft.Extensions.Logging.Abstractions 8.0.0`
- **Not present**: `CommunityToolkit.Mvvm`, `Microsoft.Extensions.DependencyInjection`

### Under `net8.0-windows7.0`

All `net8.0` entries, plus:
- Direct: `CommunityToolkit.Mvvm 8.2.2`
  - Transitive: `Microsoft.Extensions.DependencyInjection.Abstractions 8.0.0`
  - Transitive: `System.Runtime.CompilerServices.Unsafe 6.0.0`
- Direct: `Microsoft.Extensions.DependencyInjection 8.0.0`
  - Transitive: `Microsoft.Extensions.DependencyInjection.Abstractions 8.0.0`

### Critical Mend detection assertions

1. Total distinct packages across both TFMs: 8 unique packages (`Newtonsoft.Json`, `Serilog`, `Microsoft.Extensions.Logging`, `Microsoft.Extensions.DependencyInjection.Abstractions`, `Microsoft.Extensions.Logging.Abstractions`, `CommunityToolkit.Mvvm`, `Microsoft.Extensions.DependencyInjection`, `System.Runtime.CompilerServices.Unsafe`).
2. `CommunityToolkit.Mvvm` and `Microsoft.Extensions.DependencyInjection` must appear **only** in the `net8.0-windows7.0` project, not in `net8.0`.
3. Detection must be driven from `packages.lock.json` (lockfile-driven mode). The lockfile has two top-level TFM keys matching the two TargetFrameworks.
4. `Microsoft.Extensions.DependencyInjection.Abstractions 8.0.0` is a transitive shared by three direct packages on Windows — Mend must not deduplicate it away from child lists.
5. Source: all packages resolve from `nuget.org` (no private feed).
