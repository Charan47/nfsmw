# AI State Tree Plan

This document captures a practical per-vehicle AI state-tree structure based on the current `AIVehiclePursuit`, `AIPursuit`, `AICopManager`, `AIGoal`, and `AIAction` logic in `src/Speed/Indep/Src/AI`.

The intent is to model the existing code clearly without forcing every piece of tactical data into a runtime state node.

## Design Principles

- Use states for stable behavior modes.
- Use goals as the main child states inside combat.
- Use actions as the currently selected behavior inside a goal.
- Keep formation, support assignment, visibility, and drivability as context/transition data rather than top-level states.
- Treat `AIPursuit` as group-tactics control and the per-vehicle tree as local execution.

## Recommended Top-Level State Tree

```text
CopAI
├─ Calm
│  └─ Patrol
├─ Hostile
│  ├─ TargetDetected
│  ├─ PursuitRequested
│  └─ SearchOrTracking
└─ Combat
   ├─ PursuitGoal
   ├─ RamGoal
   ├─ PitGoal
   ├─ HeadOnRamGoal
   ├─ PullOverGoal
   ├─ StaticRoadBlockGoal
   └─ FleeGoal
```

## Top-Level Meaning

### Calm

Use for normal non-engaged police behavior.

- No valid target.
- No active pursuit membership.
- Patrol/default roaming behavior.

Closest code concept:

- `AIGoalPatrol`

### Hostile

Use for the transition band between patrol and committed combat.

- Target seen or acquired.
- Pursuit may be requested but not fully established.
- Vehicle is suspicious/engaging but not yet in a committed tactical goal.

This is mainly a presentation/design layer. The current code does not expose one single clean `Hostile` goal, but the behavior exists across:

- target acquisition
- pursuit request
- early pursuit establishment

### Combat

Use for active pursuit/tactical engagement.

- Vehicle is attached to a pursuit or committed to an active tactical role.
- Goal switching becomes the main high-level state mechanism.

This is the most important parent state for per-vehicle behavior.

## Combat Child States = Goals

Inside `Combat`, use goals as the real child states because that is how the code actually switches high-level behavior via `SetGoal(...)`.

### PursuitGoal

Default active chase mode.

Typical actions:

- `PursuitOffRoad`
- `Race`
- `TrafficSearch`
- `GetUnstuck`
- `Airborne`
- `TooDamaged`

### RamGoal

Used when pursuit formation logic promotes a cop into a ramming role.

Typical actions:

- `Ram`
- `PursuitOffRoad`
- `Race`
- recovery actions

### PitGoal

Used when pursuit formation logic promotes a cop into a pit role.

Typical actions:

- `Ram`
- `PursuitOffRoad`
- `Race`
- recovery actions

### HeadOnRamGoal

Usually support-driven rather than default formation-driven.

Typical actions:

- `HeadOnRam`
- `Race`
- recovery actions

### PullOverGoal

Special positional/tactical containment state.

This currently appears more like a formation-commanded role than a fully fleshed-out action-rich goal in the available decomp snapshot.

### StaticRoadBlockGoal

Roadblock/support role.

### FleeGoal

Exit/retreat behavior after pursuit detaches or when specific vehicles should disengage.

## Action Layer

Actions should be modeled as the currently active behavior inside a goal, not as siblings of goals.

Recommended action layer:

```text
Goal
├─ PrimaryAction
├─ FallbackAction
├─ RecoveryAction
└─ EmergencyAction
```

Examples:

### PursuitGoal actions

- `Race`
- `PursuitOffRoad`
- `TrafficSearch`
- `GetUnstuck`
- `Airborne`
- `TooDamaged`

### RamGoal actions

- `Ram`
- `PursuitOffRoad`
- `Race`
- `GetUnstuck`
- `Airborne`
- `TooDamaged`

### PitGoal actions

- `Ram`
- `PursuitOffRoad`
- `Race`
- `GetUnstuck`
- `Airborne`
- `TooDamaged`

### HeadOnRamGoal actions

- `HeadOnRam`
- `Race`
- recovery actions

## Context Layers That Should Not Be Top-Level States

These should be stored as data/context/transition conditions, not made into full state nodes.

### Tactical Context

```text
TacticalContext
├─ InFormation
├─ PursuitOffset
├─ InPositionOffset
├─ InPositionGoal
└─ SupportGoal
```

Purpose:

- holds tactical assignments coming from `AIPursuit` or `AICopManager`
- explains why a vehicle may leave `PursuitGoal` and enter `RamGoal`, `PitGoal`, or `HeadOnRamGoal`

### Perception Context

```text
PerceptionContext
├─ TargetValid
├─ CanSeeTarget
├─ TimeSinceTargetSeen
├─ DrivableToTargetPos
└─ DrivableToDriveNav
```

Purpose:

- drives action selection inside goals
- explains transitions such as visible chase vs search-like behavior

### Pursuit Context

```text
PursuitContext
├─ PursuitRequestPending
├─ AttachedToPursuit
├─ SupportAssigned
└─ RoadBlockAssigned
```

Purpose:

- captures whether this vehicle is merely interested in a pursuit, fully attached to one, or assigned to a specialized support role

## Formation and Support Should Be Transition Drivers

Formation should not be represented as the main child state of `Combat`.

Instead:

- `AIPursuit` assigns formation role data to the vehicle
- that assignment sets `InPositionGoal`
- the vehicle later transitions from `PursuitGoal` to a tactical goal such as `PitGoal` or `RamGoal`

Similarly:

- `AICopManager` support logic may set a `SupportGoal`
- that can transition the vehicle into `HeadOnRamGoal` or `StaticRoadBlockGoal`

Recommended interpretation:

- formation/support are transition causes
- goals are the actual state nodes

## Transition Model

### Common transition path

```text
Calm
  -> Hostile
  -> Combat/PursuitGoal
  -> Combat/TacticalGoal
  -> Combat/FleeGoal or Calm
```

### Tactical escalation

```text
Combat/PursuitGoal
  -> RamGoal        when formation assigns `AIGoalRam`
  -> PitGoal        when formation assigns `AIGoalPit`
  -> PullOverGoal   when formation assigns `AIGoalPullOver`
  -> HeadOnRamGoal  when support assigns `AIGoalHeadOnRam`
  -> StaticRoadBlockGoal when support assigns roadblock role
```

### Recovery overrides inside a goal

Inside a given goal, actions may switch to:

- `GetUnstuck`
- `Airborne`
- `TooDamaged`

without changing the parent goal.

## Why This Structure Is Preferred

This structure stays close to the code while avoiding unnecessary complexity.

Benefits:

- easy to reason about per vehicle
- matches `SetGoal(...)` high-level switching
- preserves action selection beneath goals
- keeps tactical metadata separate from stable states
- avoids turning every offset/flag into an explicit node

## Minimal Practical Runtime Shape

If this were implemented simply, the runtime per-vehicle model could be:

```text
TopState = Calm | Hostile | Combat
CombatGoal = Pursuit | Ram | Pit | HeadOnRam | PullOver | StaticRoadBlock | Flee
CurrentAction = goal-specific selected action
Context = perception + tactical assignment + pursuit membership
```

That is the recommended compact version.
