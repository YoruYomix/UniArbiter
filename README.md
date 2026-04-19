# UniArbiter

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A Unity-focused **Arbiter Pattern** library that solves the "pick one from many candidates" problem through composable strategy pipelines.

## Table of Contents

- [Motivation](#motivation)
- [Core Concept](#core-concept)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
  - [IStrategy\<T\>](#istrategyt)
  - [ArbiterContext](#arbitercontext)
  - [Strategy Types](#strategy-types)
  - [Arbiter\<T\>](#arbitert)
  - [Runtime Modifications](#runtime-modifications)
- [Examples](#examples)
- [Advanced Usage](#advanced-usage)
- [Design Principles](#design-principles)
- [License](#license)

## Motivation

Games constantly face the "multiple candidates, one choice" problem:

- **Souls-like:** Multiple lock-on targets — which enemy should the camera focus on?
- **RPG:** NPC, chest, and door overlap — which one does the interact key activate?
- **Auto-battle:** Healer has multiple wounded allies — who gets healed first?
- **3D Action:** Wall-climb, ledge-drop, and ladder are all available — which action triggers?
- **TCG:** Multiple effects can activate simultaneously — which one resolves?
- **Resource Management:** Memory pressure — which loaded asset gets unloaded first?

The typical approach — `if-else` branches, hardcoded behavior tree leaves, or state machine transitions — breaks down quickly. Every new condition touches existing logic, and the number of edge cases grows exponentially.

UniArbiter replaces all of that with a **strategy pipeline**.

## Core Concept

```
Candidates → Strategy[0] → Strategy[1] → ... → Strategy[N] → Final Selection
```

An arbiter simply runs an array of strategies in order. Each strategy filters, scores, or reshapes the candidate list — the arbiter doesn't care how.

## Installation

Add via Unity Package Manager using the Git URL:

```
https://github.com/YoruYomix/UniArbiter.git
```

## Quick Start

### 1. Define a candidate type

```csharp
public interface IEnemy
{
    float HpRatio { get; }
    bool HasTaunt { get; }
    bool IsHidden { get; }
    bool IsInvincible { get; }
    int Row { get; }
}
```

### 2. Write strategy steps

```csharp
public class ExcludeHidden : IRule<IEnemy>
{
    public IReadOnlyList<IEnemy> Apply(IReadOnlyList<IEnemy> candidates, ArbiterContext ctx)
    {
        var result = candidates.Where(e => !e.IsHidden).ToList();
        return result.Count > 0 ? result : candidates; // fallback to all if none remain
    }
}

public class BackRowFilter : IRule<IEnemy>
{
    public IReadOnlyList<IEnemy> Apply(IReadOnlyList<IEnemy> candidates, ArbiterContext ctx)
    {
        var maxRow = candidates.Max(e => e.Row);
        return candidates.Where(e => e.Row == maxRow).ToList();
    }
}

public class HpRatioScorer : IScorer<IEnemy>
{
    public float Score(IEnemy enemy, ArbiterContext ctx)
    {
        return 1f - enemy.HpRatio; // lower HP = higher score
    }
}
```

### 3. Create an arbiter

```csharp
// Define strategy types as empty structs
public struct ArcherBaseRule { }
public struct ArcherBaseScoring { }

var arbiter = new Arbiter<IEnemy>(
    new RuleStrategy<ArcherBaseRule>(
        new ForceTauntTarget(),
        new ExcludeHidden(),
        new BackRowFilter()
    ),
    new ScoringStrategy<ArcherBaseScoring>(
        (new HpRatioScorer(), 0.5f),
        (new ThreatScorer(), 0.5f)
    )
);
```

### 4. Resolve

```csharp
IReadOnlyList<IEnemy> targets = arbiter.Resolve(enemies, context);

// Never returns null — returns an empty list if all candidates are filtered out.
if (targets.Count > 0)
    targets[0].TakeDamage(damage);
```

## API Reference

### IStrategy\<T\>

The single interface every strategy implements.

```csharp
public interface IStrategy<T>
{
    IReadOnlyList<T> Execute(IReadOnlyList<T> candidates, ArbiterContext context);
}
```

- **Rule strategies** — remove candidates that don't meet conditions, return the rest.
- **Scoring strategies** — assign scores to candidates, return the highest.
- Regardless of behavior, the signature is identical. The arbiter makes no distinction.

### ArbiterContext

A context object carrying situational information (current turn, active scene, combat state, etc.). Populate it with whatever your strategies need.

```csharp
public class ArbiterContext
{
    // Add fields relevant to your game:
    // bool IsInCombat, int CurrentTurn, string ActiveScene, etc.
}
```

### Strategy Types

Strategies are identified by their type parameter — an empty struct that acts as a compile-time key. All runtime modifications target strategies by type, eliminating string-based lookups.

```csharp
public struct ArcherBaseRule { }
public struct ArcherBaseScoring { }
public struct ArcherCharmRule { }
```

### Arbiter\<T\>

#### Creation

```csharp
// Rule + Scoring
var arbiter = new Arbiter<IEnemy>(
    new RuleStrategy<ArcherBaseRule>(new ExcludeHidden(), new BackRowFilter()),
    new ScoringStrategy<ArcherBaseScoring>(
        (new HpRatioScorer(), 0.5f),
        (new ThreatScorer(), 0.5f)
    )
);

// Rule only
var arbiter = new Arbiter<IEnemy>(
    new RuleStrategy<ArcherBaseRule>(new ForceTauntTarget(), new ExcludeHidden(), new BackRowFilter())
);

// Scoring only
var arbiter = new Arbiter<IEnemy>(
    new ScoringStrategy<ArcherBaseScoring>(
        (new HpRatioScorer(), 0.5f),
        (new ThreatScorer(), 0.5f)
    )
);
```

#### Execution Flow

```
Resolve(candidates, context)
    → Strategy[0].Execute(candidates) → narrows or reorders
    → Strategy[1].Execute(remaining)  → narrows or reorders
    → ...
    → returns filtered result list
```

### Runtime Modifications

All runtime modifications return an `IDisposable` handle. Call `Dispose()` to revert. Every modification targets a strategy by type. When multiple modifications compete for the same target, the one with the highest **weight** wins.

#### API Overview

| Method | Purpose |
|---|---|
| `Override<T>(strategy, weight)` | Temporarily replace an entire strategy |
| `InsertFirst<T>(step)` | Insert a step at the beginning of a strategy |
| `InsertLast<T>(step)` | Insert a step at the end of a strategy |
| `InsertBefore<TStrategy, TStep>(step)` | Insert before a specific step |
| `InsertAfter<TStrategy, TStep>(step)` | Insert after a specific step |
| `Suppress<TStrategy, TStep>(weight)` | Temporarily disable a specific step |
| `InsertFirstAll(step)` | Insert a step at the beginning of all strategies |
| `InsertLastAll(step)` | Insert a step at the end of all strategies |
| `SuppressAll<TStep>(weight)` | Disable a step across all strategies |

All methods return `IDisposable`.

#### Override

Temporarily replace an entire strategy. The target is identified by strategy type.

```csharp
// Replace ArcherBaseRule with a charm rule
var charm = arbiter.Override<ArcherBaseRule>(
    new RuleStrategy<ArcherCharmRule>(new AllyFilter(), new BackRowFilter()),
    weight: 50
);

// Revert to original
charm.Dispose();
```

Works the same for scoring strategies:

```csharp
var frenzy = arbiter.Override<ArcherBaseScoring>(
    new ScoringStrategy<ArcherFrenzyScoring>(
        (new ThreatScorer(), 1.0f)
    ),
    weight: 30
);

frenzy.Dispose();
```

There is no separate API for rules vs. scoring — `Override<T>` handles both uniformly.

#### Insert

Temporarily inject steps into a specific strategy.

```csharp
// Insert at the front of ArcherBaseRule
var taunt = arbiter.InsertFirst<ArcherBaseRule>(new ForceTauntTarget());

// Insert before BackRowFilter inside ArcherBaseRule
var shield = arbiter.InsertBefore<ArcherBaseRule, BackRowFilter>(new ExcludeShielded());

// Insert at the front of ALL strategies
var tauntAll = arbiter.InsertFirstAll(new ForceTauntTarget());

// Revert
taunt.Dispose();
shield.Dispose();
tauntAll.Dispose();
```

#### Suppress

Temporarily disable a specific step within a strategy.

```csharp
// Disable ExcludeHidden inside ArcherBaseRule (e.g., "True Sight" buff)
var trueSight = arbiter.Suppress<ArcherBaseRule, ExcludeHidden>(weight: 10);

trueSight.Dispose(); // step re-enabled
```

#### Weight-Based Competition

When multiple modifications target the same slot, the highest weight wins. All handles remain in a priority queue — `Dispose()` removes a handle from the queue, and the next-highest automatically promotes.

If weights are equal, the most recently added handle takes priority.

```
Queue state: [Charm w:20 active] [Confuse w:10 waiting] [MildDaze w:5 waiting]

Charm.Dispose()     → [Confuse w:10 active] [MildDaze w:5 waiting]
Confuse.Dispose()   → [MildDaze w:5 active]
MildDaze.Dispose()  → default strategy restored
```

```csharp
var confuse = arbiter.Override<ArcherBaseRule>(
    new RuleStrategy<ConfuseRule>(new RandomFilter()), weight: 10);
var charm = arbiter.Override<ArcherBaseRule>(
    new RuleStrategy<CharmRule>(new AllyTargetFilter()), weight: 20);

// Dispose charm → confuse promotes
charm.Dispose();

// Dispose confuse → default strategy restored
confuse.Dispose();
```

#### Idempotent Dispose

Calling `Dispose()` on an already-disposed handle is a no-op. Handles can be disposed in any order regardless of their active/waiting state.

```csharp
var confuse = arbiter.Override<ArcherBaseRule>(
    new RuleStrategy<ConfuseRule>(new RandomFilter()), weight: 10);
var charm = arbiter.Override<ArcherBaseRule>(
    new RuleStrategy<CharmRule>(new AllyTargetFilter()), weight: 20);

// Dispose waiting handle first
confuse.Dispose();

// Dispose charm → confuse already gone, default restores
charm.Dispose();

// No-op
confuse.Dispose();
```

#### Strategy Lifecycle

Steps added at construction time and steps inserted at runtime are treated identically during execution. The only difference is whether you hold a handle to remove them.

- **Construction-time steps:** Permanent. No handle, cannot be removed.
- **Inserted steps:** `IDisposable` handle returned. Removable via `Dispose()`.

When an inserted step is removed, any overrides or suppressions targeting it are also disposed automatically.

```csharp
var taunt = arbiter.InsertFirst<ArcherBaseRule>(new ForceTauntTarget());

// Suppress the inserted taunt step
var suppressTaunt = arbiter.Suppress<ArcherBaseRule, ForceTauntTarget>(weight: 10);

// Removing the taunt step also cleans up its suppression
taunt.Dispose();
// suppressTaunt.Dispose() is now a no-op (idempotent)
```

## Examples

### Auto-Battle: Archer Target Selection

```csharp
var archerArbiter = new Arbiter<IEnemy>(
    new RuleStrategy<ArcherBaseRule>(
        new ForceTauntTarget(),
        new ExcludeHidden(),
        new ExcludeInvincible(),
        new BackRowFilter()
    ),
    new ScoringStrategy<ArcherBaseScoring>(
        (new HpRatioScorer(), 0.5f),
        (new DebuffCountScorer(), 0.3f),
        (new ThreatScorer(), 0.2f)
    )
);
```

### Auto-Battle: Tank Target Selection

```csharp
var tankArbiter = new Arbiter<IEnemy>(
    new RuleStrategy<TankBaseRule>(
        new ForceTauntTarget(),
        new ExcludeHidden(),
        new ExcludeInvincible(),
        new FrontRowFilter()
    ),
    new ScoringStrategy<TankBaseScoring>(
        (new HpRatioScorer(), 0.3f),
        (new ThreatScorer(), 0.7f)
    )
);
```

### Souls-Like: Lock-On Target

```csharp
var lockOnArbiter = new Arbiter<IEnemy>(
    new RuleStrategy<LockOnRule>(
        new ExcludeOffScreen(),
        new ExcludeOutOfRange()
    ),
    new ScoringStrategy<LockOnScoring>(
        (new CameraDirectionScorer(), 0.4f),
        (new DistanceScorer(), 0.3f),
        (new HpRatioScorer(), 0.3f)
    )
);
```

### 3D Action: Context Action Selection

```csharp
var contextArbiter = new Arbiter<IContextAction>(
    new RuleStrategy<ContextActionRule>(
        new ExcludeUnavailableActions(),
        new ExcludeOnCooldown()
    ),
    new ScoringStrategy<ContextActionScoring>(
        (new DistanceScorer(), 0.4f),
        (new AngleScorer(), 0.3f),
        (new PriorityScorer(), 0.3f)
    )
);
```

### Turn-Based: Camera Focus Selection

```csharp
var cameraArbiter = new Arbiter<IUnit>(
    new RuleStrategy<CameraFocusRule>(
        new ActionableUnitsOnly(),
        new PrioritizeEventUnits()
    ),
    new ScoringStrategy<CameraFocusScoring>(
        (new DangerScorer(), 0.5f),
        (new HpRatioScorer(), 0.3f),
        (new FrontLineDistanceScorer(), 0.2f)
    )
);
```

### Asset Unload Priority

```csharp
var assetArbiter = new Arbiter<LoadedAsset>(
    new RuleStrategy<AssetUnloadRule>(
        new ExcludeActiveSceneAssets(),
        new ExcludePreloadedAssets()
    ),
    new ScoringStrategy<AssetUnloadScoring>(
        (new LastAccessTimeScorer(), 0.4f),
        (new MemorySizeScorer(), 0.3f),
        (new ReferenceCountScorer(), 0.3f)
    )
);
```

## Advanced Usage

### Grouping Strategies with Interfaces

Since strategies are identified by type, you can leverage C# interface hierarchies to group and target them.

```csharp
public interface IBaseRule { }

public struct ArcherBaseRule : IBaseRule { }
public struct TankBaseRule : IBaseRule { }
public struct MageBaseRule : IBaseRule { }
```

```csharp
// Insert taunt into ALL base-rule strategies
var taunt = arbiter.InsertFirst<IBaseRule>(new ForceTauntTarget());

// Insert into archer only
var buff = arbiter.InsertFirst<ArcherBaseRule>(new ExcludeHidden());
```

Interfaces act as strategy groups — no need for separate `All` APIs when a single type parameter controls scope.

### Multi-Interface Tagging

A single strategy type can implement multiple interfaces, belonging to several categories at once.

```csharp
public interface IBaseRule { }
public interface IBackRowRule { }
public interface IPhysicalRule { }

public struct ArcherBaseRule : IBaseRule, IBackRowRule, IPhysicalRule { }
public struct MageBaseRule : IBaseRule, IBackRowRule { }
public struct TankBaseRule : IBaseRule, IPhysicalRule { }
```

```csharp
// Taunt for all base rules
arbiter.InsertFirst<IBaseRule>(new ForceTauntTarget());

// Suppress stealth detection for back-row characters only
arbiter.Suppress<IBackRowRule, ExcludeHidden>(weight: 10);

// Add armor scorer for physical characters only
arbiter.InsertLast<IPhysicalScoring>(new ArmorScorer(0.3f));
```

No custom tag system needed — the C# type system handles it, with full IDE autocomplete, compile-time verification, and refactoring safety.

### Same Buff, Different Effects Per Class

Interface tags let the same buff automatically branch without any `if` statements.

```csharp
// "Frenzy" buff activates:
// Physical characters → switch to front-row targeting
arbiter.InsertFirst<IPhysicalRule>(new FrontRowFilter());
// Back-row characters → ignore stealth
arbiter.Suppress<IBackRowRule, ExcludeHidden>(weight: 10);

// Archer (IPhysicalRule + IBackRowRule) → both effects apply
// Tank (IPhysicalRule only) → only front-row targeting
// Mage (IBackRowRule only) → only stealth ignore
```

### DI Integration (VContainer)

The first type parameter of an arbiter serves as its identity, enabling generic-based DI registration without string keys.

```csharp
builder.Register<Arbiter<ArcherBaseRule, IEnemy>>(Lifetime.Singleton)
    .WithParameter(new IStrategy<IEnemy>[]
    {
        new RuleStrategy<ArcherBaseRule>(new ExcludeHidden(), new BackRowFilter()),
        new ScoringStrategy<ArcherBaseScoring>(
            (new HpRatioScorer(), 0.5f),
            (new ThreatScorer(), 0.5f))
    });
```

```csharp
public class ArcherAI
{
    readonly Arbiter<ArcherBaseRule, IEnemy> _arbiter;

    public ArcherAI(Arbiter<ArcherBaseRule, IEnemy> arbiter)
    {
        _arbiter = arbiter;
    }
}
```

## Design Principles

- **Single concept** — an arbiter runs an array of strategies in order. That's it.
- **Single interface** — every strategy is `IStrategy<T>`.
- **Types as identifiers** — strategy types replace string keys. Compile-time safe.
- **Composition over inheritance** — build strategies from small, reusable steps.
- **IDisposable + weight** — the only mechanism for all runtime modifications.
- **Zero dependencies** — no external packages required.
- **IL2CPP safe** — fully compatible with AOT compilation.

## License

[MIT](LICENSE)
