# Pursuit Formation Notes

This note summarizes the current findings about how police pursuit formations are chosen in the GameCube decomp worktree, starting from the zero-agent case.

## Short Answer

- A new `AIPursuit` starts with `mActiveFormation = STAGGER_FOLLOW`.
- That initial formation object is created immediately with `InitFormation(0)`.
- The real runtime formation switching is handled later by `AIPursuit::UpdateFormation`, which is present in the original symbol data but not yet decompiled in this worktree.
- Formation selection is driven by the active `pursuitlevels` data set, which is effectively the chase tuning table for the current pursuit/heat context.
- Once a formation is chosen, runtime viability is checked using available cops, per-formation limits, per-slot minimum target counts, and target-relative geometry.

## Zero-Agent Start

When a new pursuit activity is constructed, it does not start with "no formation". It starts with `STAGGER_FOLLOW`:

- [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L382)
- [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L383)

Relevant lines:

- `mActiveFormation = STAGGER_FOLLOW;`
- `InitFormation(0);`

`InitFormation` only instantiates the formation object for the already-selected `mActiveFormation`:

- [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L703)

## How a Pursuit Starts from Patrol

A patrol cop first decides whether a target is worth pursuing in:

- [`src/Speed/Indep/Src/AI/Common/AIVehicleCopCar.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIVehicleCopCar.cpp#L98)

Important gating:

- If the target is not already in pursuit, there is no active 911 call, there was no recent traffic hit, heat is `<= 3`, and the target speed is below `65 MPH`, the cop refuses to start pursuit.
- See [`src/Speed/Indep/Src/AI/Common/AIVehicleCopCar.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIVehicleCopCar.cpp#L126)

If that check passes, the cop latches the target:

- [`src/Speed/Indep/Src/AI/Common/AIVehicleCopCar.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIVehicleCopCar.cpp#L130)

Then `AICopManager` either:

- attaches the cop to an existing pursuit for that target, or
- creates a new `AIPursuit` activity and adds the cop to it.

Relevant points:

- existing pursuit lookup / creation: [`src/Speed/Indep/Src/AI/Activities/AICopManager.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Activities/AICopManager.cpp#L371)
- pursuing cop request pickup: [`src/Speed/Indep/Src/AI/Activities/AICopManager.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Activities/AICopManager.cpp#L1316)
- add to matching existing pursuit: [`src/Speed/Indep/Src/AI/Activities/AICopManager.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Activities/AICopManager.cpp#L1322)
- create a fresh `AIPursuit`: [`src/Speed/Indep/Src/AI/Activities/AICopManager.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Activities/AICopManager.cpp#L1400)

## What `pursuitlevels` Is

`pursuitlevels` is the generated AttribSys class that holds most of the tuning data for a chase:

- [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h)

`AIPursuit` reads it through:

- [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L422)

The target/perpetrator is effectively the owner of the currently active pursuit tuning.

### Data Inside `pursuitlevels`

Formation types:

- [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h#L17)

Formation records:

- [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h#L28)
- [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h#L194)

Each `CopFormationRecord` contains:

- `Formation`
- `Duration`
- `Frequency`

Other important pursuit tuning entries:

- `MaxCopsCollapsing`: [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h#L130)
- `CollapseInnerRadius`: [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h#L154)
- `CollapseAggression`: [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h#L178)
- `StaggerFormationTime`: [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h#L230)
- `CollapseOuterRadius`: [`src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Generated/AttribSys/Classes/pursuitlevels.h#L294)

The same data set also controls spawn pacing, engage/evade thresholds, heli timing, roadblock probability, and cop-type mix.

## How `pursuitlevels` Is Selected

The active `pursuitlevels` instance is not hardcoded inside `AIPursuit`. It is loaded based on escalation tables, heat/race context, and stored on the AI vehicle side:

- [`src/Speed/Indep/Src/AI/Common/AIVehicle.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIVehicle.cpp#L1172)

In that area, `AIVehicle` creates `pursuitlevels` and `pursuitsupport` from `pursuitescalation` tables depending on race mode or heat mode.

So, in practical terms:

- heat / pursuit context chooses the active tuning table
- that table defines the formation candidates and timing

## Are Formations Decided by Heat Level?

Broadly yes, but not by heat level alone.

The decision model is:

1. Current pursuit/heat context selects a `pursuitlevels` data set.
2. That data set defines which formations exist in the pool through `CopFormations[]`.
3. `AIPursuit::UpdateFormation` evaluates the live chase and chooses or maintains a formation.
4. That chosen formation is then checked against real runtime viability.

So heat decides the rule set. Runtime logic decides whether a formation from that rule set is actually usable now.

## Runtime Viability Checks

The worktree already contains the helper logic used after formation choice.

### Formation-Wide Limits

Each formation class sets its own limits, for example:

- `SetMaxCops(...)`
- `SetMinFinisherCops(...)`
- `SetHasFinisher(...)`

Examples:

- `BoxInFormation`: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L46)
- `RollingBlockFormation`: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L109)
- `FollowFormation`: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L164)
- `PitFormation`: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L223)

### Per-Slot Minimum Cop Counts

Each offset slot inside a formation is added with a `minTargets` requirement:

- [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L42)

Examples:

- Box-in has positions requiring 1, 2, or 4 cops before that slot becomes eligible.
- Follow/stagger/follow-like layouts also encode slot priority through `minTargets`.

### Assignment and Geometry

Once a formation is active, these helpers distribute cops into usable slots:

- assign a slot to a cop: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L738)
- special chopper handling: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L750)
- expand/trim formation offsets to fit current candidates: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L770)
- assign closest cops to formation slots: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L821)
- collapse / ring setup logic: [`src/Speed/Indep/Src/AI/Common/AIPursuit.cpp`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIPursuit.cpp#L968)

These functions show that viability is not just "do we have enough cops". It also depends on:

- whether the cops are usable candidates
- where they are around the target
- whether they are support units or special cases
- whether the current target motion and chase geometry fit the tactic

## What Is Missing in the Current Decompiled Source

The exact runtime selection logic that changes `mActiveFormation` is not yet present in the current `AIPursuit.cpp` source body.

However, the original binary metadata confirms these functions exist:

- `UpdateFormation__9AIPursuitf`
- `OnTask__9AIPursuitP10HSIMTASK__f`

Their original addresses from [`config/GOWE69/symbols.txt`](X:/P4Workspaces/NFSMWSourceCode/nfsmw/config/GOWE69/symbols.txt) are:

- `UpdateFormation__9AIPursuitf = .text:0x80032850`
- `OnTask__9AIPursuitP10HSIMTASK__f = .text:0x80034300`

The available debug metadata for `UpdateFormation` shows locals such as:

- `formationCandidateLimit`
- `countInFormation`
- `countInPosition`
- `pursuitLevelAttrib`
- collapse-related values

This strongly supports the current interpretation:

- `pursuitlevels` provides the candidate formations and tuning
- `UpdateFormation` performs the live viability filtering and assignment

## Current Best Model

At the moment, the most accurate model is:

1. A new pursuit begins in `STAGGER_FOLLOW`.
2. The active `pursuitlevels` table, chosen from heat/pursuit context, defines what tactical formations are available.
3. Runtime `UpdateFormation` chooses or maintains a formation from that table.
4. The chosen formation is only used if enough valid cops are available and the geometry works.
5. Slot-level `minTargets` and formation-wide limits determine how much of that formation can actually be filled.
6. Assigned cops receive target-relative offsets and tactical goals such as `AIGoalRam`, `AIGoalPit`, or `AIGoalPullOver`.

## Practical Answer to the Earlier Question

Yes:

- formations are broadly driven by the active heat/pursuit-level tuning table
- then runtime logic checks whether the formation is viable
- and part of that viability check is minimum cop count, both at the full-formation level and at the per-slot level

But:

- viability is not only cop count
- it also includes target-relative positioning and chase state
