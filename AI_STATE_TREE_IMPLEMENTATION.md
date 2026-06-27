# AI State Tree Implementation Notes

This document is the low-level companion to `AI_STATE_TREE_PLAN.md`.

It captures the implementation-oriented structure and the discussion context that led to the current design:

- how target acquisition works
- how pursuit membership is established
- how goals and actions differ
- how `AIPursuit` drives formation-based tactical role changes
- how `AICopManager` drives support-based tactical role changes
- how to translate that into a practical per-vehicle runtime state model

## Scope

This file is intentionally implementation-facing.

It is not just a conceptual tree. It is a map of:

- runtime layers
- ownership boundaries
- transition causes
- per-vehicle state responsibilities
- what should be explicit state vs context data

## Core Architecture

The AI in the current codebase is not one flat state machine.

It is layered:

1. per-vehicle local behavior
2. group-level pursuit control
3. support/spawn/roadblock orchestration
4. goal selection
5. action selection

The main pieces are:

- `AIVehicleCopCar`
- `AIVehiclePursuit`
- `AIPursuit`
- `AICopManager`
- `AIGoal`
- `AIAction`

## What Each Major Class Owns

### `AIVehicleCopCar`

Per-cop local behavior shell.

Responsibilities:

- watch for targets
- acquire target candidates
- check pursuit eligibility
- run local update loop
- run current goal each frame

Discussion context:

- This is where a cop first sees a target.
- Initial target acquisition does not itself mean full pursuit has started.

### `AIVehiclePursuit`

Per-vehicle pursuit-capable extension.

Responsibilities:

- start pursuit / patrol / flee / roadblock modes
- store pursuit-local state
- expose `PursuitRequest()`
- keep visibility timing (`TimeSinceTargetSeen`)
- hold tactical assignment data such as:
  - `InPositionGoal`
  - `SupportGoal`
  - `PursuitOffset`
  - `InPositionOffset`

Discussion context:

- `StartPursuit()` is the real entry into pursuit mode.
- `DoInPositionGoal()` is the direct high-level goal switch for formation roles.

### `AIPursuit`

Shared group pursuit object for one target.

Responsibilities:

- hold the shared target
- maintain the list of pursuing cops
- assign formation roles
- assign target-relative offsets
- pick tactical formations
- start tactical goal transitions on individual cops

Discussion context:

- This is the tactical commander.
- It does not directly drive steering every frame.
- It decides which cop should become a rammer, pit car, pull-over unit, etc.

### `AICopManager`

Global orchestration layer.

Responsibilities:

- attach vehicles to pursuits
- create/find pursuit activities
- spawn new cops
- assign support vehicles
- assign support goals like head-on ram and roadblock

Discussion context:

- This is not the local per-car brain.
- It manages pursuit membership and special support roles.

### `AIGoal`

Per-vehicle high-level behavior mode.

Responsibilities:

- represent the vehicle's current high-level tactical state
- contain a set of candidate actions
- pick the current action

Discussion context:

- Goals are the right level to use as main child states inside `Combat`.

### `AIAction`

Per-goal immediate execution mode.

Responsibilities:

- decide if it can run now
- provide score/priority for selection
- update steering/speed/input behavior

Discussion context:

- Actions should not be the same thing as tasks.
- They are behavior modes chosen inside a goal.

## Target Acquisition Flow

This section records the earlier logic discussion so the state implementation remains grounded in code behavior.

### Initial sighting

In `AIVehicleCopCar`, the cop:

1. scans potential targets
2. builds a temporary `AITarget`
3. checks `CanSeeTarget(...)`
4. checks pursuit-related conditions
5. stores the target if allowed

Implementation consequence:

- There is a meaningful distinction between:
  - target detected
  - pursuit requested
  - active pursuit

This distinction motivated the `Hostile` layer.

### Pursuit request

After a target is stored but before active pursuit begins, the cop may expose a `PursuitRequest()`.

Implementation consequence:

- a vehicle can be in a pre-combat escalation state
- this is one reason `Hostile` is useful in the high-level tree

### Pursuit attach

`AICopManager` notices the request and attaches the vehicle to an `AIPursuit`.

Implementation consequence:

- `Combat` should begin when pursuit membership is active, not just when the target was first seen

## What `StartPursuit()` Actually Means

When pursuit truly starts, the vehicle:

- turns on police lights
- acquires the shared target
- runs `UpdateTargeting()`
- marks `InPursuit = true`
- switches to `AIGoalPursuit`

Implementation consequence:

- the real entry state under `Combat` should be `PursuitGoal`
- not `RamGoal`, `PitGoal`, etc.

## Meaning of `UpdateTargeting()`

`UpdateTargeting()` computes whether the target position is directly drivable from the current vehicle position without world collision blocking the way.

Practical meaning:

- it updates `DrivableToTargetPos`

Discussion context:

- this does not pick tactics
- it does not directly steer the vehicle
- it is used by later actions to decide if direct pursuit is feasible

Implementation consequence:

- `DrivableToTargetPos` should be context/blackboard data
- not a state node

## Visibility Context

The earlier discussion established that visibility matters in two places:

1. initial acquisition
2. later pursuit maintenance

`AIVehiclePursuit` updates:

- `TimeSinceTargetSeen`
- periodic `CanSeeTarget(...)` tests during pursuit

Implementation consequence:

- visibility should live in perception context
- not as a separate top-level behavior state

## Why `Combat` Should Contain Goals

The code switches high-level tactical behavior with `SetGoal(...)`.

That means:

- `Pursuit`
- `Ram`
- `Pit`
- `HeadOnRam`
- `PullOver`
- `StaticRoadBlock`
- `Flee`

are the real stable behavior modes.

Discussion context:

- We considered whether formation should be the main child structure.
- The conclusion was no: formation is a transition driver and assignment system.
- Goals are the correct child states.

Implementation consequence:

- `Combat` child states should be goals

## Why Actions Should Be Nested Under Goals

The code shows that each goal owns a set of actions and chooses the current one.

Examples:

- `AIGoalPursuit` chooses from `PursuitOffRoad`, `Race`, `TrafficSearch`, recovery actions
- `AIGoalRam` chooses from `Ram`, `PursuitOffRoad`, `Race`, recovery actions
- `AIGoalPit` chooses from `Ram`, `PursuitOffRoad`, `Race`, recovery actions

Discussion context:

- We discussed whether actions should be tasks.
- Conclusion:
  - goals = stable tactical modes
  - actions = immediate behavior modes inside goals
  - tasks = low-level update work

Implementation consequence:

- Actions should be children or submodes inside goals
- Tasks should not be modeled as siblings of actions

## `AIActionRace` vs `AIActionPursuitOffRoad`

This was a major part of the earlier reasoning and should be preserved here because it influences state naming and transitions.

### `AIActionRace`

Best interpretation from available code:

- road-following chase
- uses navigation heavily
- lane-based pursuit behavior
- updates along the road network toward the target

Design interpretation:

- normal chase using road/navigation logic

### `AIActionPursuitOffRoad`

Best interpretation from available code:

- more direct pursuit than plain road-following
- still constrained by drivability, road affinity, separation, and wall avoidance
- not simply "drive on dirt"

Design interpretation:

- direct chase mode
- less lane-bound than `Race`
- still world- and nav-aware

Implementation consequence:

- inside `PursuitGoal`, a common action competition is:
  - `PursuitOffRoad`
  - `Race`

## Formation-Driven Tactical Goal Switching

This was identified as the main tactical escalation mechanism.

### Important clarification

The transition is not:

- `AIActionPursuitOffRoad -> AIActionRam`

The real transition is:

- `AIGoalPursuit -> AIGoalRam` or `AIGoalPit` or `AIGoalPullOver`

through assignment and later `DoInPositionGoal()`.

### How it works

`AIPursuit`:

1. picks or maintains a formation
2. filters candidate cops
3. assigns pursuit offset / in-position offset / in-position goal
4. later commits the role on the selected cop

Stored tactical data on the cop:

- `PursuitOffset`
- `InPositionOffset`
- `InPositionGoal`
- `InFormation`

Commit step:

- `DoInPositionGoal()`

Effect:

- old goal is destroyed
- new tactical goal is created

Implementation consequence:

- formation is best modeled as tactical context + transition trigger
- not as the primary child state under `Combat`

## Pit Formation Context

This topic came up specifically in the earlier discussion.

### Key conclusion

Pit does not emerge from the off-road action itself.

Instead:

1. the vehicle may currently be in `PursuitGoal`
2. and currently running `PursuitOffRoad`
3. `AIPursuit` later assigns `InPositionGoal = AIGoalPit`
4. `DoInPositionGoal()` switches the vehicle to `AIGoalPit`

So pit is a goal-layer escalation, not an action-layer branch.

### Practical implementation note

When a tactical role is assigned, it should be represented as:

- pending tactical assignment in context
- later committed goal switch

not as an instantaneous replacement of the current action without a goal change

## Support-Driven Tactical Goal Switching

This is the second major escalation mechanism.

`AICopManager` can assign:

- `SupportGoal = AIGoalHeadOnRam`
- `SupportGoal = AIGoalStaticRoadBlock`

Then when the support vehicle attaches to the pursuit, `StartSupportGoal()` activates that goal.

Implementation consequence:

- support assignment should be modeled as:
  - context on the vehicle
  - transition trigger into a specific combat goal

## Why `PullOver` Looks Different

This came up in the later part of the discussion.

Reason:

- `PullOver` currently appears more like a direct formation-commanded positional state than a richly populated goal in the available decomp snapshot

Interpretation:

- it is still a valid goal node in the tree
- but it may need lighter internal structure until more implementation detail is recovered

Implementation consequence:

- keep `PullOverGoal` as a combat child state
- allow it to have a simpler internal action model than `PursuitGoal` or `RamGoal`

## Recommended Runtime Model

Use a compact runtime representation:

```text
TopState
  Calm | Hostile | Combat

CombatGoal
  Pursuit | Ram | Pit | HeadOnRam | PullOver | StaticRoadBlock | Flee

CurrentAction
  goal-specific selected action

Context
  Perception + TacticalAssignment + PursuitMembership
```

This preserves the code's behavior while keeping the system sane to maintain.

## Recommended Per-Vehicle State Structure

```text
CopAI
├─ TopState
│  ├─ Calm
│  ├─ Hostile
│  └─ Combat
├─ CombatGoal
│  ├─ PursuitGoal
│  ├─ RamGoal
│  ├─ PitGoal
│  ├─ HeadOnRamGoal
│  ├─ PullOverGoal
│  ├─ StaticRoadBlockGoal
│  └─ FleeGoal
├─ CurrentAction
│  ├─ Race
│  ├─ PursuitOffRoad
│  ├─ Ram
│  ├─ HeadOnRam
│  ├─ TrafficSearch
│  ├─ GetUnstuck
│  ├─ Airborne
│  └─ TooDamaged
└─ Context
   ├─ Perception
   ├─ TacticalAssignment
   └─ PursuitMembership
```

## Suggested Context Data

### Perception

- `TargetValid`
- `CanSeeTarget`
- `TimeSinceTargetSeen`
- `DrivableToTargetPos`
- `DrivableToDriveNav`
- `TargetDistance`

### Tactical Assignment

- `InFormation`
- `PursuitOffset`
- `InPositionOffset`
- `InPositionGoal`
- `SupportGoal`

### Pursuit Membership

- `PursuitRequested`
- `AttachedToPursuit`
- `SupportAssigned`
- `RoadBlockAssigned`

## Guidance On Implementation Weight

This should not be implemented as a giant deeply nested formal state machine where every variable is also a state.

Keep it light:

- make only stable behavior modes explicit states
- keep tactical/perception data as context
- use context to drive transitions

Discussion context:

- We explicitly concluded that the model becomes too heavy only if every offset, visibility flag, or tactical detail becomes its own state node

## Final Practical Interpretation

The best implementation pattern is:

- `TopState` gives broad engagement phase
- `CombatGoal` gives current tactical mode
- `CurrentAction` gives immediate behavior
- context explains why the current goal/action was chosen and when it should change

That is the recommended low-level structure for continuing AI design work.
