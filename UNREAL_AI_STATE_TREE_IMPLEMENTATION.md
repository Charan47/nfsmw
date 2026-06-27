# Unreal AI State Tree Implementation

Implementation details of AI logic in the Unreal Engine state tree.

## Purpose

This file is the history book for the Unreal Engine AI state tree implementation we are building.

It should record:

- what the original codebase currently does
- how we are translating that into the Unreal state tree
- what assumptions we are making
- what we intentionally simplify at first
- how later objectives and tasks expand from the initial implementation

## Naming Note

For this implementation:

- original codebase term: `Goal`
- Unreal implementation term used here: `Objective`

These are conceptually the same layer.

## Initial Combat Structure

The first test implementation will focus on `Combat`.

Planned structure:

- `Combat` as the parent state
- a task under `Combat` that decides:
  - formation
  - offsets
  - objective transition
- `Objective` states as children of `Combat`

The first objective to implement is:

- `Pursuit`

## Current Codebase: What Pursuit Does

In the current codebase, `Pursuit` is the default active chase objective after a cop has fully entered pursuit mode.

### Entry into pursuit

At the per-vehicle level, pursuit begins when `StartPursuit()` is called on the cop vehicle.

That function does the following at a high level:

- enables cop lights
- acquires the target
- updates targeting data
- marks the vehicle as in pursuit
- switches the vehicle to the pursuit goal

Practical meaning:

- pursuit is the first real combat objective
- it is not just "target seen"
- it is not just "pursuit requested"
- it is the committed chase state after attachment to a shared pursuit

### Role of Pursuit in the current code

`Pursuit` is the baseline combat objective.

While in pursuit, the vehicle:

- actively chases the target
- keeps target visibility timing updated
- refreshes whether the target position is directly drivable
- allows higher-level pursuit systems to assign tactical roles later

It is the state a cop stays in until one of the following happens:

- a tactical objective is assigned, such as:
  - `Ram`
  - `Pit`
  - `PullOver`
  - `HeadOnRam`
  - `StaticRoadBlock`
- a recovery condition overrides it, such as:
  - stuck
  - airborne
  - too damaged
- pursuit ends or transitions into flee/disengage behavior

### Internal behavior while in Pursuit

In the current codebase, the pursuit goal contains a small action-selection layer.

The important pursuit actions are:

- `PursuitOffRoad`
- `Race`
- `TrafficSearch`
- `GetUnstuck`
- `Airborne`
- `TooDamaged`

From the current discussion and code reading:

- `Race` is the more navigation/road-following chase behavior
- `PursuitOffRoad` is the more direct chase behavior when the target is sufficiently reachable and drivable

So the current `Pursuit` objective is not one hardcoded behavior.
It is a combat objective that can express multiple immediate chase styles.

### Tactical relationship to Pursuit

`Pursuit` also acts as the staging objective for later tactical escalation.

That means:

- the vehicle can remain in pursuit while the shared pursuit controller evaluates formations
- the vehicle can receive formation data such as:
  - pursuit offset
  - in-position offset
  - in-position objective
- once committed, the vehicle may leave `Pursuit` and transition into a more specific objective

This is important for the Unreal implementation:

- `Pursuit` should be treated as the baseline combat objective
- formation logic should not replace it immediately
- formation logic should be able to assign future tactical objectives while the vehicle is still in `Pursuit`

## Unreal Translation: First Objective

The first objective we will implement is `Pursuit`.

### Unreal meaning of Pursuit objective

In the Unreal state tree, `Pursuit` should mean:

- the AI is in active combat/chase mode
- it has a valid target
- it is attached to an active pursuit context
- it is using chase behavior as its current objective
- it is eligible for later tactical reassignment

### What the Combat-level task should do before or alongside Pursuit

The `Combat` parent task should eventually be responsible for:

- maintaining pursuit context
- deciding formation
- computing/storing offsets
- deciding whether to remain in `Pursuit`
- deciding whether to transition into another objective

For the first pass, we are simplifying that.

### First-pass implementation intent

For the initial test:

- enter `Combat`
- transition into `Pursuit` objective
- allow a combat-level task to prepare future formation/offset logic
- do not require the full tactical transition system yet

### Simplified first-pass understanding

For now, `Pursuit` in Unreal should be treated as:

- the default combat objective
- the place where standard chase behavior lives
- the base state from which more specific tactical objectives will later branch

## Notes for Later Expansion

When more objectives are added, `Pursuit` should remain the baseline branch point for:

- `Ram`
- `Pit`
- `PullOver`
- `HeadOnRam`
- `StaticRoadBlock`

This file should be extended as each of those objectives is implemented.

## Next Implementation Step: Combat Parent Task

Earlier in the discussion this was described as the `Combat` parent task or `Combat` controller task.

This is the task that lives on the `Combat` state itself and keeps the high-level combat context updated while child objectives run beneath it.

For the Unreal implementation, this task should be treated as an innate/always-running combat task.

Important clarification:

- this task is not a solo authority
- it communicates with `RagePursuitSubsystem`
- it should be treated as the per-vehicle combat-side coordinator that consumes and contributes pursuit context

### Purpose of the Combat parent task

It should not directly replace objective logic.

Its job is to maintain and prepare combat-wide data so the current objective can run and so later tactical transitions become possible.

It should also serve as the bridge between:

- per-vehicle state tree execution
- shared pursuit/tactical data owned or mediated by `RagePursuitSubsystem`

### Responsibilities for the first pass

For the initial implementation, the `Combat` parent task should do the following:

- validate that combat still has a valid target
- maintain combat context
- maintain pursuit-related shared data references
- communicate with `RagePursuitSubsystem`
- compute or refresh tactical assignment placeholders
- compute or refresh offsets placeholders
- decide whether the current objective should remain `Pursuit`
- expose data needed by the child objective state

### What it should not do yet

For the first pass, it should not fully implement:

- tactical reassignment into `Ram`
- tactical reassignment into `Pit`
- pull-over logic
- support-goal switching
- full formation selection

Instead, it should act as the stable parent controller for the first usable combat path.

### Unreal interpretation

Recommended first-pass interpretation:

- `Combat` owns the always-running per-vehicle combat coordination task
- that task communicates with `RagePursuitSubsystem`
- child objective handles immediate chase behavior

This keeps the structure close to the original code split:

- `AIPursuit` / higher-level context update
- then per-vehicle objective/action behavior

## First Objective Path: Pursuit -> Race

The first concrete objective path to implement should be:

```text
Combat
└─ Pursuit
   └─ Race
```

This is the safest initial implementation path because `Race` is the clearest road-following chase behavior in the current code discussion.

### Why start with Race

From the current code understanding:

- `Race` is a stable navigation-driven chase mode
- it is easier to model first than direct tactical transitions
- it fits naturally under `Pursuit`
- it gives us a working baseline before `PursuitOffRoad`, `Ram`, or `Pit`

### Current-code meaning of Race

`Race` under pursuit should be interpreted as:

- road-following chase behavior
- navigation-driven pursuit
- lane/path-aware pursuit movement
- baseline movement toward the target when direct tactical pursuit is not the current special case

## Proposed Unreal State Tree Shape For This Step

```text
Combat
├─ Task: CombatContextTask
└─ Objective: Pursuit
   └─ State: Race
```

## CombatContextTask

Suggested first-pass name:

- `CombatContextTask`

This is the name that best matches what we discussed earlier as the combat parent task/controller task.

If implementation naming should reflect the subsystem link more explicitly, an alternative name would be:

- `CombatControllerTask`

That name better communicates that this task is coordinating with `RagePursuitSubsystem` rather than acting alone.

### First-pass data this task should own or refresh

- current target handle/reference
- whether target is valid
- whether combat should continue
- pursuit context reference
- subsystem-provided pursuit state reference
- placeholder formation type
- placeholder pursuit offset
- placeholder in-position offset
- placeholder future objective assignment

This gives us enough structure to grow into:

- formation logic
- objective switching
- support-role logic

without rebuilding the state tree shape later.

### First-pass output expectation

For now the task should effectively say:

- combat is active
- current objective remains `Pursuit`
- no special tactical objective assigned yet

And that conclusion should be made with subsystem context available, not from local-only state.

## Pursuit Objective

The `Pursuit` objective should be the first child objective under `Combat`.

### First-pass responsibility

For this step, `Pursuit` should:

- represent active baseline chase mode
- consume the combat context produced by `CombatContextTask`
- choose the simple immediate chase state we are implementing first

For now that child state is:

- `Race`

## Race State

The `Race` state is the first immediate behavior state inside `Pursuit`.

### Intended meaning

In Unreal, `Race` should mean:

- chase the target using navigation/path-following logic
- behave like the current codebase's road-following pursuit mode
- avoid introducing tactical complexity yet

### First-pass behavior goal

The first-pass `Race` state should be responsible for:

- moving toward the target along nav/path logic
- acting as the default pursuit movement state
- serving as the stable baseline for later transitions to:
  - `PursuitOffRoad`
  - `Ram`
  - `Pit`

### First-pass simplification

At this stage, `Race` should not try to fully reproduce:

- direct tactical interception
- role-based pursuit offset maneuvers
- support behavior
- pit or ram commitment

It should simply be:

- the default navigation-driven chase state under `Pursuit`

## First-Pass Transition Logic

For the first working version, use the simplest possible transition model:

```text
Enter Combat
  -> run CombatContextTask
  -> enter Pursuit objective
  -> enter Race state
```

Remain there while:

- target is valid
- combat context is valid
- no tactical override objective is assigned

Later expansions can add transitions out of `Pursuit/Race` into:

- `Pursuit/PursuitOffRoad`
- `Ram`
- `Pit`
- `PullOver`
- `HeadOnRam`

## Implementation Intent Summary

The current implementation step should be treated as:

- building the stable `Combat` parent structure
- building the first objective under it: `Pursuit`
- building the first immediate chase state under pursuit: `Race`

This gives us a clean, extensible starting spine:

```text
Combat
├─ CombatContextTask / CombatControllerTask
└─ Pursuit
   └─ Race
```

Where the parent task:

- coordinates local combat state
- communicates with `RagePursuitSubsystem`
- prepares data for later tactical objective switching

Everything else can branch from this later without changing the top-level design.

## Detailed Current-Code Understanding: Race

This section records the current understanding of what `Race` is doing in the original codebase.

Important caveat:

- the visible `AIActionRace::Update(float dT)` body is still stubbed in the current decomp snapshot
- the breakdown below is based on visible setup logic, helper methods, member intent, and supporting functions
- so this is a detailed responsibility map, not a claimed exact instruction-for-instruction runtime reconstruction

### Summary interpretation

`Race` in the current codebase is best understood as the baseline road-based chase driver.

It is not just:

- picking a racing line
- catching up in a simplistic way
- or following one lane blindly

It appears to bundle:

- road/nav validation
- lane/path setup
- nav look-ahead
- speed planning from road shape
- target-relative speed shaping
- pursuit/flee/support modifiers
- dead-end/off-path recovery
- stable road-following chase continuation

### 1. Establish a valid road-based drive path

`Race` first validates that road-based driving is even possible.

Current-code meaning:

- build a temporary road/nav object from current position and heading
- verify that direction-based road navigation is valid from the current location

Behavioral meaning:

- `Race` should only run when the vehicle can sensibly attach itself to road-based pursuit driving
- if no valid road-based path can be formed, `Race` should not be the active immediate chase state

Implementation meaning for Unreal:

- `Race` begins from "road chase is viable here"

### 2. Choose an appropriate driving lane/path mode

`Race` explicitly configures the driving nav to:

- direction-based road navigation
- racing lane mode
- valid lane selection

Behavioral meaning:

- the action is intentionally road-following
- it chooses a lane/path mode suited for aggressive pursuit driving
- it is not simply moving directly to the target in free space

Implementation meaning for Unreal:

- `Race` should own road/path mode configuration
- lane/path policy is part of the action's identity

### 3. Look ahead along navigation

`Race` advances navigation ahead of the vehicle rather than steering to a purely local point.

Current-code meaning:

- maintain a projected navigation point in front of the car
- update that point using look-ahead distance
- update occluded navigation position as part of nav maintenance

Behavioral meaning:

- smoother road following
- less reactive wobble
- more stable pursuit driving at speed

Implementation meaning for Unreal:

- `Race` should not chase a purely immediate point
- projected nav/look-ahead behavior is a core responsibility

### 4. Compute a speed envelope from road shape and vehicle capability

`Race` computes how fast the AI should be driving from:

- road curvature
- grip
- top speed
- vehicle capability scaling
- AI skill/performance scaling

Behavioral meaning:

- it does not simply drive flat-out
- it computes a contextual road-speed envelope
- tighter roads reduce acceptable speed
- better vehicle capability increases acceptable speed

Implementation meaning for Unreal:

- `Race` should own a road-aware speed model
- "how fast should I drive on this road?" is one of its primary jobs

### 5. Modify that speed envelope relative to the target

In pursuit mode, `Race` also uses target-relative information:

- target position
- target velocity
- relation between cop and target along the chase direction
- pursuit geometry

Behavioral meaning:

- the car does not simply obey the road-speed envelope
- it biases its speed according to the chase situation
- it tries to manage pursuit pressure in relation to the target

Implementation meaning for Unreal:

- `Race` should include target-relative chase shaping on top of pure road-speed logic

### 6. Use pursuit/flee/support modifiers

`Race` changes behavior depending on wider tactical context.

Visible contextual modes include:

- pursuit mode
- flee mode
- support-goal-influenced mode

Behavioral meaning:

- the same broad road-driving behavior is reused across multiple combat contexts
- chase driving is not identical across pursuit, flee, and support preparation scenarios

Implementation meaning for Unreal:

- `Race` should not be treated as a single context-free nav behavior
- mode modifiers should be part of the implementation contract

### 7. Recover if the nav path becomes poor

`Race` explicitly checks whether the current navigation path is degrading.

Current-code meaning:

- inspect out-of-bounds state
- attempt to construct a better nav solution from current position
- reset and rebuild nav if the current path is poor and a better one exists

Behavioral meaning:

- pursuit driving is actively maintained
- the AI should not stubbornly follow a bad road path forever
- nav recovery is part of stable chase behavior

Implementation meaning for Unreal:

- `Race` should include path-quality repair or off-path recovery support

### 8. Continue driving toward the target in a stable road-following chase style

This is the combined behavioral result of the previous responsibilities.

Behavioral meaning:

- stay anchored to a sensible road path
- keep navigation ahead of the car
- choose speed from both road constraints and chase needs
- recover path quality when needed
- continue pressuring the target without immediately escalating into tactical attack roles

Implementation meaning for Unreal:

- `Race` should be treated as the stable baseline road-chase action under `Pursuit`

### Practical one-line definition

For the Unreal implementation, the safest current-code-faithful definition is:

- `Race` = baseline intelligent road-following chase behavior

### Important distinction from `PursuitOffRoad`

This understanding also reinforces the distinction:

- `Race`
  - road-following chase
  - lane/path-aware pursuit
  - stable baseline pursuit locomotion

- `PursuitOffRoad`
  - more direct target-oriented chase
  - less constrained by standard lane-following
  - still not completely unconstrained

### Implementation note

When translating `Race` into Unreal, it is acceptable to decompose this bundled behavior into reusable tasks.

However, that decomposition is a new implementation structure, not a literal copy of the original `AIActionRace` source layout.

## Race Implementation Checklist

This checklist is based on the current notes already written in this file.

It is intended to answer:

- what is already part of the planned structure
- what still needs to exist for `Race` to become meaningful

### Already written / already defined in this document

- [x] `Combat` is the parent state for active chase/tactical behavior
- [x] `Combat` has an always-running parent/controller task concept
- [x] the parent combat task is not solo and should communicate with `RagePursuitSubsystem`
- [x] `Pursuit` is the first baseline combat objective
- [x] `Race` is the first immediate state under `Pursuit`
- [x] `Combat -> Pursuit -> Race` is the initial state-tree spine
- [x] `Race` is understood as baseline intelligent road-following chase behavior
- [x] `Race` is distinguished from `PursuitOffRoad`
- [x] `Race` is understood to be decomposable into reusable implementation tasks later

### Needed for first usable Race implementation

- [ ] `CombatControllerTask` or equivalent parent combat coordinator task exists in code
- [ ] `CombatControllerTask` can request transition into `Pursuit`
- [ ] `Pursuit` objective exists in code
- [ ] `Pursuit` has its own task that can request transition into `Race`
- [ ] `Race` state exists in code
- [ ] `RaceTask` or equivalent immediate chase task exists in code
- [ ] a valid target reference is available to `Race`
- [ ] basic combat/pursuit context is available to `Race`

### Needed for Race to reflect the current-code meaning

- [ ] ability to establish a valid road-based drive path
- [ ] ability to choose an appropriate driving lane/path mode
- [ ] ability to compute or maintain look-ahead along navigation
- [ ] ability to compute a speed envelope from road shape and vehicle capability
- [ ] ability to modify that speed envelope relative to the target
- [ ] ability to apply pursuit/flee/support-style mode modifiers
- [ ] ability to recover if the navigation path becomes poor
- [ ] ability to continue stable road-following chase behavior over time

### Dependencies Race likely needs around it

- [ ] road network representation
- [ ] way to anchor the vehicle to the road network
- [ ] lane/path selection support
- [ ] navigation look-ahead support
- [ ] simple drive target generation from nav
- [ ] simple speed planning
- [ ] simple dead-end or invalid-path handling
- [ ] simple collision/drivability checks

### Can be simplified in the first pass

- [ ] full formation logic can be skipped initially
- [ ] tactical reassignment out of `Pursuit` can be skipped initially
- [ ] full `PursuitOffRoad` implementation can be skipped initially
- [ ] full support-goal handling can be skipped initially
- [ ] full pathfinding implementation can be deferred
- [ ] full collision-aware routing can be deferred
- [ ] advanced off-path recovery can be deferred

### Recommended first-pass goal

The first practical target for `Race` should be:

- [ ] enter `Combat`
- [ ] parent combat task requests `Pursuit`
- [ ] pursuit task requests `Race`
- [ ] `Race` receives a valid target and pursuit context
- [ ] `Race` can follow a road-based navigation target
- [ ] `Race` can move the AI in a stable baseline chase loop

### Recommended second-pass additions

After the first pass works, expand `Race` with:

- [ ] better lane/path decisions
- [ ] target-relative speed shaping
- [ ] off-path recovery
- [ ] dead-end handling
- [ ] pursuit-mode modifiers
- [ ] support/flee modifiers
