---
name: ubt-integration
description: Managing the Unreal Build Tool (UBT) via Target.cs and Build.cs files. Configuring cross-module dependencies and project prefixes.
---

# Unreal Build Tool (UBT) Integration (UE5 Expert)

## Activation / When to Use
- Mandatory when creating new plugins or adding major external libraries.
- Trigger when UBT throws linker errors (LNK2019) or missing header errors.
- Use to configure project-specific build settings (e.g., enabling RTTI, optimizing build times).

## 1. The Build Hierarchy
- **Target.cs**: Configures the overall build for a specific target (Game, Editor, Client, Server). It dictates *how* the program is built.
- **Build.cs**: Configures individual modules. It dictates *what* is included in the module (Dependencies, Includes).

## 2. Target.cs Best Practices
Enable strict includes and IWYU (Include What You Use) to force good coding standards.

```csharp
// MyProject.Target.cs
public class MyProjectTarget : TargetRules
{
    public MyProjectTarget(TargetInfo Target) : base(Target)
    {
        Type = TargetType.Game;
        DefaultBuildSettings = BuildSettingsVersion.V5;
        IncludeOrderVersion = EngineIncludeOrderVersion.Unreal5_4;

        // Force strict IWYU (Catches hidden include dependencies)
        bEnforceIWYU = true;
        
        ExtraModuleNames.Add("MyProject");
    }
}
```

## 3. Build.cs Best Practices
Manage dependencies cleanly.

```csharp
// MyModule.Build.cs
public class MyModule : ModuleRules
{
    public MyModule(ReadOnlyTargetRules Target) : base(Target)
    {
        PCHUsage = PCHUsageMode.UseExplicitOrSharedPCHs;

        // Expose public include paths for other modules to use
        PublicIncludePaths.Add(ModuleDirectory + "/Public");
        
        // Private include paths (hidden from dependents)
        PrivateIncludePaths.Add(ModuleDirectory + "/Private");

        PublicDependencyModuleNames.AddRange(new string[] { "Core" });
        PrivateDependencyModuleNames.AddRange(new string[] { "CoreUObject", "Engine" });
    }
}
```

## 4. Impact on Safety
- **Cross-Module Integrity**: Clean `Build.cs` files prevent circular dependencies which can completely break the UBT process.
- **Build Times**: Enabling IWYU and keeping Public dependencies small ensures that modifying a single file does not trigger a massive 20-minute recompile.

## Common Mistakes (BAD vs GOOD)

**BAD (Target.cs - Legacy settings)**:
```csharp
DefaultBuildSettings = BuildSettingsVersion.V2; // Outdated, disables new UE5 optimizations.
```

**GOOD (Target.cs - Modern UE5)**:
```csharp
DefaultBuildSettings = BuildSettingsVersion.V5; // Enforces UE5.4+ standards.
```

## Verification Checklist
- [ ] `Target.cs` uses `BuildSettingsVersion.V5`.
- [ ] `bEnforceIWYU = true` is enabled in `Target.cs`.
- [ ] `Public/PrivateIncludePaths` are correctly mapped in `Build.cs`.
- [ ] Dependencies are separated strictly into Public and Private.
