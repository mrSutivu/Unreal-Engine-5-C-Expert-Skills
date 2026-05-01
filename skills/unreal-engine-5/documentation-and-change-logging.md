---
name: documentation-and-change-logging
description: Maintaining automated RELEASE.md files, technical documentation, and .agent/workflows for commercial distribution.
---

# Documentation and Change-logging (UE5 Expert)

## Activation / When to Use
- Mandatory for professional distribution, team collaboration, and Fab Marketplace compliance.
- Trigger before tagging a new version or submitting a plugin update.

## 1. Core Principles
High-quality code is worthless if users (or other AI agents) don't know how to use it. Documentation must be structured, versioned, and machine-readable.

## 2. The RELEASE.md Standard
Maintain a `RELEASE.md` (or `CHANGELOG.md`) at the root of the plugin. Use Semantic Versioning (SemVer: Major.Minor.Patch) and the Keep a Changelog format.

```markdown
# Changelog

## [1.1.0] - 2026-05-01
### Added
- MegaLights support for the Spawner module.
- New `UEditorUtilityWidget` for batch renaming.

### Changed
- Refactored `TObjectPtr` migration logic.

### Fixed
- Memory leak in `NativeDestruct` of `UMyUserWidget`.
```

## 3. Technical Documentation (User Facing)
Fab requires hosted or included documentation.
- **Setup Guide**: How to enable and configure the plugin.
- **Blueprint API**: List of major nodes with screenshots or text descriptions.
- **C++ API**: Instructions on how to add the plugin to `PublicDependencyModuleNames`.

## 4. The .agent/workflows/ Directory
For tools designed to be used by Antigravity or Claude agents, provide explicit workflow files.

```yaml
# .agent/workflows/plugin-setup.yaml
name: Plugin Integration
description: Steps for the agent to install this plugin into a new project.
steps:
  - Add to Build.cs PublicDependencyModuleNames.
  - Inherit AMyPluginActor instead of AActor.
```

## 5. Impact on Safety & Trust
- **Oversight Demand**: Comprehensive docs drastically reduce support tickets and user frustration.
- **Agent Handoff**: A well-maintained changelog allows future AI sessions to instantly understand what was modified recently, preventing redundant work.

## Verification Checklist
- [ ] `RELEASE.md` exists and follows SemVer.
- [ ] Technical documentation explains BOTH Blueprint and C++ usage.
- [ ] Breaking changes are explicitly highlighted in the changelog.
- [ ] Plugin dependencies are documented for users who need to modify `Build.cs`.
