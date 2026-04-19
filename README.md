<p align="center">
  <h1 align="center">UniArbiter</h1>
  <p align="center">
    A Unity-specific arbiter pattern library that solves the "choose one from multiple candidates" problem in games
    <br />
    <br />
    <a href="#installation">Installation</a>
    ·
    <a href="#quick-start">Quick Start</a>
    ·
    <a href="#api-reference">API Reference</a>
    ·
    <a href="#examples">Examples</a>
    ·
    <a href="#license">License</a>
  </p>
</p>

---

## Why Do You Need This?

When developing games, you repeatedly encounter the situation: "there are multiple candidates, but you need to pick just one."

| Genre | Situation |
|---|---|
| Souls-like | Multiple lock-on targets — which one to target? |
| RPG | NPC, chest, and door overlap — which one does the F key interact with? |
| Auto-battle | Healer has multiple allies — who to heal first? |
| 3D Action | Wall-cling, ledge-drop, and ladder-climb are all available — which to perform? |
| TCG | Multiple effects can trigger simultaneously — which one to apply? |
| Resource Management | Under memory pressure — which loaded asset to unload first? |

Currently, these problems are solved with if-else branching, hardcoded BT leaf nodes, or state machine transition conditions.
Every time a new condition is added, existing branches must be modified, and the number of cases grows exponentially.

UniArbiter solves this problem with a **strategy pipeline**.

## Features

- **Single Concept** — An arbiter does one thing: "execute an array of strategies sequentially."
- **Single Interface** — All strategies are unified under `IStrategy<T>`.
- **Type as Identifier** — Strategies are identified by their type parameter; no string keys.
- **Composition First** — Build small strategies and combine them as arrays.
- **IDisposable + Weight** — The sole principle for all runtime modifications.
- **Zero Dependencies** — Works without any external dependencies.
- **IL2CPP Safe** — Works safely in AOT environments.

## Installation

### Unity Package Manager (Git URL)

`Window → Package Manager → + → Add package from git URL...`

```
https://github.com/YoruYomix/UniArbiter.git
```

## Core Concepts

```
Candidate List → Strategy[0] → Strategy[1] → ... → Strategy[N] → Final Selection
```

The arbiter simply executes the strategy array sequentially. Whether each strategy filters candidates or scores them, the arbiter doesn't know or care.

### IStrategy\<T\>

The single interface for all strategies.

```csharp
public interface IStrategy<T>
{
    IReadOnlyList<T> Execute(IReadOnlyList<T> candidates, ArbiterContext context);
}
```

- **Filtering Strategy** — Removes candidates that don't meet the conditions and returns the rest.
- **Scoring Strategy** — Scores candidates and returns only the highest-scoring ones.
- Regardless of what a strategy does, the signature is the same. The arbiter does not distinguish between them.

### ArbiterContext

A context object that holds current situational information.

```csharp
public class ArbiterContext
{
    // Is combat active, whose turn is it, what is the current scene, etc.
    // Users store whatever information they need
}
```

### Strategy Types

Strategies are identified by their type parameter. Each strategy registered with an arbiter has a unique type, and all runtime modifications target strategies by type.

```csharp
// Strategy type definitions (empty structs)
public struct ArcherBaseRules { }
public struct ArcherBaseScoring { }
public struct ArcherCharmRules { }
```

## Quick Start

### 1. Define the Candidate Type

Define the interface for the candidates the arbiter will choose from.

```csharp
public interface IEnemy
{
    float HpRatio { get; }
    bool HasTaunt { get; }
    bool IsHidden { get; }
    bool IsInvincible { get; }
    int Row { get; }
    List<DebuffType> Debuffs { get; }
}

public class Enemy : MonoBehaviour, IEnemy
{
    public float HpRatio => _currentHp / _maxHp;
    public bool HasTaunt => _buffs.Contains(BuffType.Taunt);
    public bool IsHidden => _buffs.Contains(BuffType.Stealth);
    public bool IsInvincible => _buffs.Contains(BuffType.Invincible);
    public int Row => _position.row;
    public List<DebuffType> Debuffs => _debuffs;
}
```

### 2. Write Strategies

Implement `IStrategy<T>`. You can use the library-provided `RuleStrategy` and `ScoringStrategy`, or implement your own.

**Rule Example:**

```csharp
public class ExcludeHiddenEnemies : IRule<IEnemy>
{
    public IReadOnlyList<IEnemy> Apply(IReadOnlyList<IEnemy> candidates, ArbiterContext ctx)
    {
        var result = candidates.Where(e => !e.IsHidden).ToList();
        return result.Count > 0 ? result : candidates;
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
```

**Scorer Example:**

```csharp
public class HpRatioScorer : IScorer<IEnemy>
{
    public float Score(IEnemy enemy, ArbiterContext ctx)
    {
        return 1f - enemy.HpRatio; // Lower health = higher score
    }
}
```

### 3. Create the Arbiter

Build the arbiter with a strategy array. Both strategies and rules within them can have weights.

```csharp
var arbiter = new Arbiter<IEnemy>(
    new RuleStrategy<ArcherBaseRules>(
        new ForceTauntTarget(),
        new ExcludeHiddenEnemies(),
        new BackRowFilter()
    ),
    new ScoringStrategy<ArcherBaseScoring>(
        (new HpRatioScorer(), 0.5f),
        (new ThreatScorer(), 0.5f)
    )
);
```

Assigning a weight to a rule gives it resistance against Override/Suppress.

```csharp
var arbiter = new Arbiter<IEnemy>(
    new RuleStrategy<ArcherBaseRules>(
        (new ForceTauntTarget(), weight: 50),  // High weight → strong resistance to Override/Suppress
        new ExcludeHiddenEnemies(),             // Weight omitted → default 0
        new BackRowFilter()
    )
);
```

### 4. Execute

```csharp
IReadOnlyList<T> results = arbiter.Resolve(candidates, context);
```

```
Resolve(candidates, context)
    → Strategy[0].Execute(candidates) → narrows or sorts candidates
    → Strategy[1].Execute(remaining)  → narrows or sorts candidates
    → ...
    → returns the filtered result array
```

### 5. Use the Results

`Resolve` always returns `IReadOnlyList<T>`. It never returns null. If all candidates are filtered out, it returns an empty array.

```csharp
var targets = arbiter.Resolve(enemies, context);

switch (targets.Count)
{
    case 0:
        break;                                          // No candidates
    case 1:
        targets[0].TakeDamage(damage);                  // Single result
        break;
    default:
        targets[0].TakeDamage(damage);                  // Use the first
        targets[Random.Range(0, targets.Count)]...;     // Random selection
        foreach (var t in targets) t.TakeDamage(...);   // AoE attack
        break;
}
```

The arbiter doesn't hide decisions. It transparently shows how many results there are, and the user decides what to do.

## API Reference

### Runtime Modifications

All runtime modifications return an `IDisposable` handle and are released via `Dispose`. When multiple modifications compete for the same target, weight determines the winner. All modifications specify their target by strategy type.

| Method | Purpose |
|---|---|
| `OverrideStrategy<T>(strategy, weight)` | Temporarily replace an entire strategy |
| `Override<TStrategy, TRule>(rule, weight)` | Replace a specific rule within a specific strategy |
| `Suppress<TStrategy, TRule>(weight)` | Disable a specific rule within a specific strategy |
| `InsertFirst<T>(rule, weight)` | Insert a rule at the beginning of a specific strategy |
| `InsertLast<T>(rule, weight)` | Insert a rule at the end of a specific strategy |
| `InsertBefore<TStrategy, TRule>(rule, weight)` | Insert before a specific rule in a specific strategy |
| `InsertAfter<TStrategy, TRule>(rule, weight)` | Insert after a specific rule in a specific strategy |
| `OverrideStrategyAll(strategy, weight)` | Temporarily replace all strategies entirely |
| `OverrideAll<TRule>(rule, weight)` | Replace the matching rule across all strategies |
| `SuppressAll<TRule>(weight)` | Disable the matching rule across all strategies |
| `InsertFirstAll(rule, weight)` | Insert a rule at the beginning of all strategies |
| `InsertLastAll(rule, weight)` | Insert a rule at the end of all strategies |

> All return `IDisposable`. The default weight for Insert methods is 0.

### Strategy Type Matching

Rule-level modifications require strategy type matching. They are applied only when the specified strategy type matches the currently active strategy's type.

```
Default: RuleStrategy<ArcherBaseRules>(ExcludeHiddenEnemies, BackRowFilter)

InsertFirst<ArcherBaseRules>(Taunt)  → ArcherBaseRules active → Applied
InsertFirst<ArcherCharmRules>(Taunt) → ArcherCharmRules inactive → Ignored

OverrideStrategy activates ArcherCharmRules →

InsertFirst<ArcherBaseRules>(Taunt)  → ArcherBaseRules inactive → Ignored
InsertFirst<ArcherCharmRules>(Taunt) → ArcherCharmRules active → Applied
```

Once the strategy type is matched, rule-level modifications do not distinguish the strategy's origin. Whether it's the default strategy set at creation time or one replaced via `OverrideStrategy`, if the matching rule type exists inside, the modification is applied to all.

### Strategy Override (OverrideStrategy)

Temporarily replaces an entire strategy.

```csharp
// Replace ArcherBaseRules with Charm rules
var charm = arbiter.OverrideStrategy<ArcherBaseRules>(
    new RuleStrategy<ArcherCharmRules>(new AllyFilter(), new BackRowFilter()),
    weight: 50
);

charm.Dispose(); // Release → next-priority override becomes active
```

**Evaluation order during strategy replacement:** When a strategy is replaced, the arbiter processes in this order:

1. Collect all rule handles (Override, Insert, Suppress) that match the new strategy's type
2. Apply the collected rule handles to the new strategy
3. Activate the fully-applied strategy

`OverrideStrategy` only replaces the strategy's base configuration. Rule modification handles remain alive independently, and if the replaced strategy's type matches, they are applied as-is.

```csharp
// Default: RuleStrategy<ArcherBaseRules>(ExcludeHiddenEnemies, BackRowFilter)

// 1. Insert Taunt
var taunt = arbiter.InsertFirst<ArcherBaseRules>(new ForceTauntTarget());
// Current: Taunt → ExcludeHiddenEnemies → BackRowFilter

// 2. Replace ArcherBaseRules entirely (replacement also has ArcherBaseRules type)
var enhanced = arbiter.OverrideStrategy<ArcherBaseRules>(
    new RuleStrategy<ArcherBaseRules>(new ExcludeHiddenEnemies(), new FrontRowFilter()),
    weight: 20);
// Current: Taunt → ExcludeHiddenEnemies → FrontRowFilter
// Taunt matches ArcherBaseRules, so it's still applied

// 3. Release enhanced
enhanced.Dispose();
// Current: Taunt → ExcludeHiddenEnemies → BackRowFilter
```

### Rule Override (Override)

Temporarily replaces a specific rule within a specific strategy with a different rule.

```csharp
var improved = arbiter.Override<ArcherBaseRules, ExcludeHiddenEnemies>(
    new ImprovedExcludeHiddenEnemies(), weight: 10);

improved.Dispose(); // Release → next-priority override applied, or falls back to original
```

### Insert

Temporarily inserts a rule into a specific strategy. Inserted rules also have weights. A higher weight means that Override/Suppress targeting that rule slot must use an even higher weight to take effect.

```csharp
// Default weight 0
var taunt = arbiter.InsertFirst<ArcherBaseRules>(new ForceTauntTarget());

// With explicit weight
var taunt = arbiter.InsertFirst<ArcherBaseRules>(new ForceTauntTarget(), weight: 30);

// Insert before a specific rule
var buff = arbiter.InsertBefore<ArcherBaseRules, BackRowFilter>(new ExcludeShielded(), weight: 10);

// Insert at the beginning of all strategies
var tauntAll = arbiter.InsertFirstAll(new ForceTauntTarget(), weight: 20);
```

### Suppress

Temporarily disables a specific rule within a specific strategy.

```csharp
var trueVision = arbiter.Suppress<ArcherBaseRules, ExcludeHiddenEnemies>(weight: 10);

trueVision.Dispose(); // Release → if no suppression, rule executes
```

## Weight Competition

When multiple modifications compete for the same target, weight determines the winner.

**Core Principle: Overwriting ≠ Releasing.** All handles remain alive in a priority queue, and only the highest one is active. `Dispose` removes from the queue, and if the active handle is removed, the next in line is automatically promoted. When weights are equal, the later one takes priority.

### Strategy-Level Competition (OverrideStrategy)

```
Queue state: [Charm w:20 active] [Confuse w:10 waiting] [WeakConfuse w:5 waiting] [Creation strategy w:0 waiting]

Charm.Dispose()      → [Confuse w:10 active] [WeakConfuse w:5 waiting] [Creation strategy w:0 waiting]
Confuse.Dispose()    → [WeakConfuse w:5 active] [Creation strategy w:0 waiting]
WeakConfuse.Dispose() → [Creation strategy w:0 active]
```

```csharp
var confuse = arbiter.OverrideStrategy<ArcherBaseRules>(
    new RuleStrategy<ConfuseRules>(new RandomFilter()), weight: 10);
var charm = arbiter.OverrideStrategy<ArcherBaseRules>(
    new RuleStrategy<CharmRules>(new AllyTargetFilter()), weight: 20);

charm.Dispose();   // Confuse promoted
confuse.Dispose(); // Creation strategy (w:0) becomes active
```

### Rule-Level Competition (Separate Suppress + Override Queues)

Each rule slot has **separate** Suppress and Override queues. Suppress/Override must have a weight higher than the rule slot's weight to be applied.

> **When Suppress and the rule slot have equal weight, Suppress wins.**
> When Override and the rule slot have equal weight, the later one takes priority (same as normal rules).

```
1. Suppress queue has an active entry (weight ≥ rule weight) → entire slot skipped (Override ignored)
2. No Suppress + Override queue has an active entry (weight ≥ rule weight) → highest-weight Override executes
3. Neither → original executes
```

Suppress means "turn this slot off," while Override means "change the contents of this slot." Suppress gets tie-breaking priority because "turning off" represents a stronger intent than "changing." To break through a Suppress, the rule weight must be higher than the Suppress weight.

```
ExcludeHiddenEnemies slot:
  Suppress queue: [TrueVision w:30 active]
  Override queue: [Improved w:20 active] [Weak w:10 waiting]

→ Suppress active → slot skipped (Override ignored)
→ TrueVision.Dispose() → Suppress queue empty → check Override queue → Improved executes
→ Improved.Dispose() → Weak promoted
→ Weak.Dispose() → original ExcludeHiddenEnemies (w:0) becomes active
```

```csharp
var improved = arbiter.Override<ArcherBaseRules, ExcludeHiddenEnemies>(
    new ImprovedExcludeHiddenEnemies(), weight: 20);
var trueVision = arbiter.Suppress<ArcherBaseRules, ExcludeHiddenEnemies>(weight: 30);

// Suppress active → slot skipped (Improved doesn't execute either)

trueVision.Dispose();
// No Suppress → check Override → Improved executes

improved.Dispose();
// Neither → original ExcludeHiddenEnemies executes
```

### Idempotency

`Dispose` on an already-released handle is ignored. Whether active or waiting, `Dispose` safely removes the handle.

### Strategy Lifecycle

Strategies and rules defined at creation time are treated identically to inserted ones. The only difference is **whether they can be removed via a handle**.

- **Creation-time strategies/rules** — Registered in the override queue with their weight. Cannot be removed since there is no handle.
- **Runtime strategies/rules** — An `IDisposable` handle is returned. Can be removed from the queue via `Dispose`. If a weight is specified, it becomes the default weight for that rule slot.

Even when an inserted rule is removed, Override/Suppress handles targeting that rule do not expire. Handles remain alive independently, and if the target reappears, they are type-matched and automatically reapplied.

```csharp
var taunt = arbiter.InsertFirst<ArcherBaseRules>(new ForceTauntTarget());
var suppressTaunt = arbiter.Suppress<ArcherBaseRules, ForceTauntTarget>(weight: 10);
// Current: (Taunt skipped) → ExcludeHiddenEnemies → BackRowFilter

taunt.Dispose();
// Current: ExcludeHiddenEnemies → BackRowFilter (suppressTaunt is alive but has no target)

var taunt2 = arbiter.InsertFirst<ArcherBaseRules>(new ForceTauntTarget());
// Current: (Taunt skipped) → ExcludeHiddenEnemies → BackRowFilter (suppressTaunt auto-reapplied)

suppressTaunt.Dispose();
// Current: Taunt → ExcludeHiddenEnemies → BackRowFilter
```

## Examples

### Auto-Battle Archer Character Target Selection

```csharp
var archerArbiter = new Arbiter<IEnemy>(
    new RuleStrategy<ArcherBaseRules>(
        new ForceTauntTarget(),
        new ExcludeHiddenEnemies(),
        new ExcludeInvincibleEnemies(),
        new BackRowFilter()
    ),
    new ScoringStrategy<ArcherBaseScoring>(
        (new HpRatioScorer(), 0.5f),
        (new DebuffCountScorer(), 0.3f),
        (new ThreatScorer(), 0.2f)
    )
);
```

### Souls-like Lock-On Target Selection

```csharp
var lockOnArbiter = new Arbiter<IEnemy>(
    new RuleStrategy<LockOnRules>(
        new ExcludeOutOfSight(),
        new ExcludeOutOfRange()
    ),
    new ScoringStrategy<LockOnScoring>(
        (new CameraDirectionScorer(), 0.4f),
        (new DistanceScorer(), 0.3f),
        (new HpRatioScorer(), 0.3f)
    )
);
```

### 3D Action Context Action Selection

```csharp
var contextArbiter = new Arbiter<IContextAction>(
    new RuleStrategy<ContextRules>(
        new ExcludeUnavailableInCurrentState(),
        new ExcludeOnCooldown()
    ),
    new ScoringStrategy<ContextScoring>(
        (new DistanceScorer(), 0.4f),
        (new AngleScorer(), 0.3f),
        (new PriorityScorer(), 0.3f)
    )
);
```

### Asset Unload Target Selection

```csharp
var assetArbiter = new Arbiter<LoadedAsset>(
    new RuleStrategy<AssetUnloadRules>(
        new ExcludeInUseByCurrentScene(),
        new ExcludePreloadedAssets()
    ),
    new ScoringStrategy<AssetUnloadScoring>(
        (new LastAccessTimeScorer(), 0.4f),
        (new MemorySizeScorer(), 0.3f),
        (new ReferenceCountScorer(), 0.3f)
    )
);
```

### Overlap Example — Multiple Status Effects Applied Simultaneously

```csharp
// Default: RuleStrategy<ArcherBaseRules>(ExcludeHiddenEnemies, BackRowFilter)

// 1. True Vision buff
var trueVision = arbiter.Suppress<ArcherBaseRules, ExcludeHiddenEnemies>(weight: 10);
// → (ExcludeHiddenEnemies skipped) → BackRowFilter

// 2. Replace ExcludeHiddenEnemies with AreaDetection
var areaDetection = arbiter.Override<ArcherBaseRules, ExcludeHiddenEnemies>(
    new AreaDetectionFilter(), weight: 10);
// → Suppress is active so slot is skipped

// 3. Insert Taunt
var taunt = arbiter.InsertFirstAll(new ForceTauntTarget());
// → Taunt → (ExcludeHiddenEnemies skipped) → BackRowFilter

// 4. Confuse → replace strategy entirely
var confuse = arbiter.OverrideStrategy<ArcherBaseRules>(
    new RuleStrategy<ConfuseRules>(new RandomFilter()), weight: 10);
// → RandomFilter

// 5-8. Dispose in order — effects unwind in reverse
confuse.Dispose();       // → Taunt → (ExcludeHiddenEnemies skipped) → BackRowFilter
taunt.Dispose();         // → (ExcludeHiddenEnemies skipped) → BackRowFilter
trueVision.Dispose();    // → No Suppress → AreaDetectionFilter → BackRowFilter
areaDetection.Dispose(); // → original ExcludeHiddenEnemies → BackRowFilter
```

## Advanced Usage

### Grouping Strategies with Interfaces

You can group strategies using C# interface hierarchies.

```csharp
public interface IBaseRules { }

public struct ArcherBaseRules : IBaseRules { }
public struct TankBaseRules : IBaseRules { }
public struct MageBaseRules : IBaseRules { }
```

```csharp
// Insert Taunt into all base rule strategies
var taunt = arbiter.InsertFirst<IBaseRules>(new ForceTauntTarget());

// Insert only into archer base rules
var buff = arbiter.InsertFirst<ArcherBaseRules>(new ExcludeHiddenEnemies());
```

Since the interface acts as a strategy group, you can control scope with a single type without needing separate `All` APIs.

### Tag Management via Multiple Inheritance

```csharp
public interface IBaseRules { }
public interface IBackRowRules { }
public interface IPhysicalRules { }

public struct ArcherBaseRules : IBaseRules, IBackRowRules, IPhysicalRules { }
public struct MageBaseRules : IBaseRules, IBackRowRules { }
public struct TankBaseRules : IBaseRules, IPhysicalRules { }
```

```csharp
arbiter.InsertFirst<IBaseRules>(new ForceTauntTarget());           // All base rules
arbiter.Suppress<IBackRowRules, ExcludeHiddenEnemies>(weight: 10); // Back row only
arbiter.InsertLast<IPhysicalScoring>(new DefenseScorer(0.3f));     // Physical only
```

No separate tagging system needed — the C# type system fills that role. IDE auto-completion, compile-time verification, and refactoring safety come for free.

### Same Buff, Different Effects Per Class

Interfaces as tags allow the same buff to branch automatically by class. No if-statements needed.

```csharp
// "Frenzy" buff activates
arbiter.InsertFirst<IPhysicalRules>(new FrontRowFilter());           // Physical: attack front row
arbiter.Suppress<IBackRowRules, ExcludeHiddenEnemies>(weight: 10);   // Back row: ignore stealth

// Archer has IPhysicalRules and IBackRowRules → both applied
// Tank has only IPhysicalRules → only one applied
// Mage has only IBackRowRules → only one applied
```

### Immunity via Creation Weight

Setting a high weight on a creation-time strategy lets you express units immune to certain overrides without any separate system.

```csharp
// Regular monster: default creation weight (0)
var minionArbiter = new Arbiter<IEnemy>(
    new RuleStrategy<MinionRules>(new FrontRowFilter())
);

// Boss: creation weight 100
var bossArbiter = new Arbiter<IEnemy>(
    new RuleStrategy<BossRules>(new FrontRowFilter(), weight: 100)
);

// Charm (w:50) activates
var charm = arbiter.OverrideStrategy<IBaseRules>(
    new RuleStrategy<CharmRules>(new AllyFilter()), weight: 50);

// Minion: creation strategy (w:0) < Charm (w:50) → Charm applied
// Boss: creation strategy (w:100) > Charm (w:50) → Charm ignored (immune)
```

### Partial Immunity via Rule Weight

Immunity can also be expressed at the rule level rather than the strategy level. This is useful when you want to protect only specific rules.

```csharp
var bossArbiter = new Arbiter<IEnemy>(
    new RuleStrategy<BossRules>(
        new FrontRowFilter(),                        // Weight 0 → Override/Suppress freely
        (new ForceAvoidInvincible(), weight: 100)     // Weight 100 → this rule is protected
    )
);

// Suppress (w:50) → creation rule (w:100) > Suppress (w:50) → ignored
var suppress = arbiter.Suppress<BossRules, ForceAvoidInvincible>(weight: 50);

// Suppress (w:100) → equal weight, Suppress wins ties → applied
var tieSuppress = arbiter.Suppress<BossRules, ForceAvoidInvincible>(weight: 100);

// Suppress (w:200) → Suppress (w:200) > creation rule (w:100) → applied
var strongSuppress = arbiter.Suppress<BossRules, ForceAvoidInvincible>(weight: 200);
```

Strategy weight protects the entire strategy from replacement, while rule weight protects individual rules from Override/Suppress. Since these two layers are independent, you can naturally express requirements like "the strategy can be replaced, but this specific rule must never be turned off."

## DI Integration (VContainer)

Since the arbiter's first type parameter is the strategy type, it can be distinguished via generics without string keys.

```csharp
builder.Register<Arbiter<ArcherBaseRules, IEnemy>>(Lifetime.Singleton)
    .WithParameter(new IStrategy<IEnemy>[]
    {
        new RuleStrategy<ArcherBaseRules>(new ExcludeHiddenEnemies(), new BackRowFilter()),
        new ScoringStrategy<ArcherBaseScoring>(
            (new HpRatioScorer(), 0.5f),
            (new ThreatScorer(), 0.5f))
    });
```

```csharp
public class ArcherAI
{
    readonly Arbiter<ArcherBaseRules, IEnemy> _arbiter;

    public ArcherAI(Arbiter<ArcherBaseRules, IEnemy> arbiter)
    {
        _arbiter = arbiter;
    }
}
```

## License

MIT
