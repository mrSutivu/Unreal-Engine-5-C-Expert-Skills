<div align="center">
  <h1>🚀 Unreal Engine 5 C++ Expert Skills</h1>
  <p><b>The Ultimate "Swiss Army Knife" for AI Agents and Senior Developers</b></p>
  
  <p>
    <img alt="Unreal Engine" src="https://img.shields.io/badge/Unreal%20Engine-5.4%2B-blue?logo=unrealengine&logoColor=white&style=for-the-badge" />
    <img alt="Language" src="https://img.shields.io/badge/Language-C%2B%2B20-00599C?logo=c%2B%2B&logoColor=white&style=for-the-badge" />
    <img alt="Antigravity Ready" src="https://img.shields.io/badge/Agent-Antigravity%20Ready-blueviolet?style=for-the-badge" />
    <img alt="Zero Hallucination" src="https://img.shields.io/badge/Status-Zero%20Hallucination-success?style=for-the-badge" />
  </p>
</div>

<br/>

## 📖 Overview

Welcome to the **Unreal Engine 5 C++ Expert Skills Library**. 

This repository contains nearly **100 hyper-specialized Markdown files (`SKILL.md`)** designed to act as the ultimate architectural and technical brain for AI Agents (like Claude, Gemini, Antigravity) and human developers. 

Unlike standard tutorials, these skills are written as **strict operational protocols**. They are optimized for AI ingestion (using the "Caveman Lite" linguistic format) to drastically reduce token consumption while maximizing technical density.

### 🛡️ The "Zero Hallucination" Philosophy
Every file in this repository is designed to prevent AI coding errors:
- **BAD vs GOOD Snippets**: Explicitly shows what *not* to do (e.g., using `Cast<T>` in a Tick loop) vs the standard (e.g., using `IsA<T>()`).
- **Verification Checklists**: Each skill ends with a binary checklist for the Agent to self-verify its code before pushing a PR.
- **Safety First**: Deep focus on pointers (`TWeakObjectPtr`), memory leaks, and Network Authority (`HasAuthority()`) to prevent silent crashes in multiplayer environments.

---

## 🗂️ The 10 Architectural Pillars

This library covers the entire spectrum of game development in UE5, from basic setup to low-level engine hacking.

### 1. 🏗️ Core C++ & Memory Management
- The migration to `TObjectPtr<T>`.
- Native Smart Pointers (`TSharedRef`, `TUniquePtr`) vs `std::`.
- High-performance memory stacks (`FMemStack`, `FInstancedStruct`).

### 2. ⚡ Performance & Optimization
- Eradicating the `Tick` function (Event-Driven architecture).
- CPU Cache Locality and Struct Padding optimization.
- Object Pooling for rapid-fire actors.
- Math & String optimization (`FStringBuilderBase`, `DistSquared`).

### 3. 🌐 Multiplayer & Network Replication
- `AGameMode` vs `AGameState` responsibilities.
- The `Push Model` (`MARK_PROPERTY_DIRTY_FROM_NAME`) for massive CPU savings.
- `FFastArraySerializer` for large inventory replication.
- Safe RPC implementation and RepNotify state synchronization.

### 4. ⚔️ Gameplay Ability System (GAS)
- `UAttributeSet` clamping and `ATTRIBUTE_ACCESSORS`.
- Instancing policies for `UGameplayAbility`.
- `UGameplayEffectExecutionCalculation` for complex damage math.
- Synchronizing targeting info via `FGameplayAbilityTargetData`.

### 5. 🧠 Artificial Intelligence (Mass Entity & BTs)
- C++ Behavior Tree Tasks (`UBTTaskNode`), Services, and Decorators.
- `FBlackboardKeySelector` for safe memory access.
- `Mass Entity` (ECS) architecture for 100,000+ simulated entities.
- The new `State Trees` hybrid system.

### 6. 🏃 Animation & Physics
- UE5's `NativeThreadSafeUpdateAnimation` for multi-threaded crowds.
- Executing `AnimMontage` and `AnimNotifies` from C++.
- Advanced Raycasts (`SweepMulti`, `FCollisionQueryParams`).
- Safely extracting `Physical Materials` for surface-specific VFX.

### 7. 🎮 Gameplay Systems & Blueprint Interop
- `BlueprintNativeEvent` vs `BlueprintImplementableEvent`.
- C++ Interfaces (`Execute_`).
- Enhanced Input System (`InputMappingContext`) in UE 5.1+.
- Native `FGameplayTag` integration.

### 8. 🖌️ UI Development (Slate & UMG)
- Pure native C++ UI via `Slate` (`SNew`, `TAttribute`).
- Invalidation Boxes for 120+ FPS UMG widgets.
- Virtualized lists (`SListView` / `STileView`).
- Proper `NativeDestruct` garbage collection to prevent Zombie UI.

### 9. 🖥️ Shaders, Rendering & RDG
- Writing HLSL in `usf`/`ush` files.
- Building passes with the `Render Dependency Graph` (`FRDGBuilder`).
- `Compute Shaders` and `UAV` manipulation.
- Modifying the UE5 `Tonemapper` and reading G-Buffer data.

### 10. 🛠️ Editor Tooling & Distribution
- Extending the Editor via `IDetailCustomization` and `FAssetTypeActions_Base`.
- Creating custom Blueprint Nodes (`UK2Node`).
- Passing Fab Marketplace submission requirements (HAL, cross-platform safety).

---

## 📂 The `references/` Directory (Memory Dumps)

While standard skills are fast and concise, the `references/` folder contains complete, copy-pasteable C++ boilerplate templates for Unreal's most complex "Final Boss" systems:
- **`rdg-pipeline-template.md`**: Complete Compute/Pixel Shader setup.
- **`gas-attribute-execution.md`**: Boilerplate for AttributeSets and ExecCalcs.
- **`async-multithreading-tasks.md`**: Syntax for `UE::Tasks`, `AsyncTask`, and `UE5Coro`.
- **`multiplayer-seamless-travel-code.md`**: Data persistence via `CopyProperties`.
- **`slate-ui-macro-lexicon.md`**: Deep dive into `SAssignNew` and Slate bindings.

---

## 🤖 How to Use (For AI Agents)

1. **Ingest the Context**: When a user assigns you a task (e.g., "Create a multiplayer gun"), scan the `SKILL.md` files related to your task (e.g., `rpc-server-client-multicast.md`, `raycast-line-trace.md`).
2. **Follow the Rules**: Treat the instructions as absolute Engine Law. Never assume an API implementation without checking the skill.
3. **Verify**: Before outputting code, run through the `Verification Checklist` at the bottom of each skill.

## 👨‍💻 How to Use (For Humans)

Read these files as a quick-reference "Senior Dev Cheat Sheet". If you forget the exact syntax for a `DOREPLIFETIME_CONDITION` or a `DECLARE_GLOBAL_SHADER`, you will find the most modern, optimized UE 5.4+ approach documented here without the fluff of a 30-minute tutorial.

---

<div align="center">
  <i>Built for the future of AI-assisted Game Development.</i>
</div>
