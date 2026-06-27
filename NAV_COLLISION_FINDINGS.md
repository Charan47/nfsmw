# Navigation And Collision Findings

This document summarizes local findings from the current project snapshot about:

- how navigation appears to be represented
- how collision appears to be queried and resolved
- whether there is evidence of waypoint-based navigation
- whether there is evidence of A* pathfinding

Important caveat:

- several world/navigation source files in this snapshot are empty or not yet decompiled in a useful way
- this write-up therefore separates:
  - what is directly supported by visible code
  - what is only suggested by declarations

## Short Answer

The current project appears to use:

- a **road-network-based navigation system**
- centered on **road segments, road nodes, lane profiles, splines, and `WRoadNav`**
- with **cookie-trail history and look-ahead navigation**
- and a **world collision system** centered on `WCollisionMgr`, `WCollider`, collision instances, barriers, and tri/object lists

There is **clear evidence of pathfinding support existing** in the API surface:

- `FindPath`
- `FindPathNow`
- `CancelPathFinding`
- `kTypePath`
- explicit stored path segments

But in the current snapshot there is **no visible implementation** proving whether that search is A*, some custom segment search, or something else.

So:

- navigation is clearly **not just raw waypoint hopping**
- it is clearly **road-network navigation**
- but **the exact pathfinding algorithm is not recoverable from the currently visible code alone**

## 1. How Navigation Appears To Be Represented

The main types are in:

- [WRoadNetwork.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WRoadNetwork.h)
- [WRoadElem.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WRoadElem.h)
- [CookieTrail.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Misc/CookieTrail.h)

### Core world-road data

The road system is built from:

- `WRoad`
- `WRoadNode`
- `WRoadProfile`
- `WRoadSegment`
- `WRoadIntersection`

From [WRoadElem.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WRoadElem.h):

- `WRoadNode` has:
  - world position
  - profile index
  - connected segment indices
- `WRoadSegment` has:
  - two node indices
  - road id
  - flags
  - handle lengths / control-handle data
- `WRoadProfile` has lane definitions
- `WRoadIntersection` tracks connected and entrance segments

This strongly suggests a **road graph with segment-based geometry**, not a simple flat waypoint list.

### Lane data exists explicitly

`WRoadProfile` stores `WRoadLane` data, and `WRoadNav` exposes lane modes such as:

- `kLaneRacing`
- `kLaneTraffic`
- `kLaneCop`
- `kLaneCopReckless`
- `kLaneAny`

This means navigation is lane-aware, not just road-centerline-aware.

### Splines are part of navigation state

`WRoadNav` stores:

- `USpline fRoadSpline`
- `USpline fLeftSpline`
- `USpline fRightSpline`

So even though the graph is segment-based, the navigation state is not moving from raw node to raw node only.  
It appears to construct spline-driven local navigation along the road geometry.

## 2. Is It Waypoint Navigation?

Not in the simple sense of:

- "array of waypoints, steer to next one, then next one"

What is visible looks more sophisticated than that.

### Why it does not look like simple waypointing

`WRoadNav` stores:

- current segment
- current node index
- lane information
- lane offsets
- path segments
- curvature
- splines
- occluded position
- cookie trail

This is much richer than a waypoint cursor.

### What it looks more like

It looks like:

- road-segment graph navigation
- local spline interpolation inside segments
- lane-aware position and offset management
- look-ahead advancement along road geometry

So the best description is:

- **segment-and-lane road navigation with spline/local-cookie support**

not:

- basic waypoint following

## 3. What `WRoadNav` Appears To Do

`WRoadNav` appears to be the main runtime navigation object used by AI.

Visible API in [WRoadNetwork.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WRoadNetwork.h):

- `InitAtPoint(...)`
- `InitAtSegment(...)`
- `InitFromOtherNav(...)`
- `IncNavPosition(...)`
- `UpdateOccludedPosition(...)`
- `SnapToSelectableLane(...)`
- `ChangeDragLanes(...)`
- `FindPath(...)`
- `FindPathNow(...)`
- `CancelPathFinding()`
- `IsSegmentInPath(...)`
- `IsSegmentInCookieTrail(...)`

And state such as:

- `fSegmentInd`
- `fLaneInd`
- `fLaneOffset`
- `fCurvature`
- `fPosition`
- `fLeftPosition`
- `fRightPosition`
- `fForwardVector`
- `fOccludedPosition`
- `nPathGoalSegment`
- `pPathSegments`

### Practical interpretation

`WRoadNav` seems to be responsible for:

- anchoring an AI vehicle to a road segment/lane
- advancing a navigation point forward
- maintaining lane-relative geometry
- optionally maintaining a higher-level path
- exposing local geometric information needed for steering and speed planning

## 4. What Is A `NavCookie`?

From [CookieTrail.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/Misc/CookieTrail.h), a `NavCookie` stores:

- left boundary
- right boundary
- forward direction
- segment length
- curvature
- offsets
- center position
- segment number and node side

This is very important.

It suggests the navigation system caches local road slices or local road snapshots that can be used for:

- steering
- lane-boundary checks
- curvature sampling
- cookie-trail validation
- road continuity / path continuity logic

So navigation is not just:

- "what segment am I on?"

It is also:

- "what is the local drivable corridor and direction around me?"

## 5. Is There Explicit Pathfinding Support?

Yes, at the API level.

Visible evidence in `WRoadNav`:

- `kTypePath`
- `SetPathType(...)`
- `FindPath(...)`
- `FindPathNow(...)`
- `FindingPath()`
- `GetPathDistanceRemaining()`
- `IsSegmentInPath(...)`
- `CancelPathFinding()`
- `nPathGoalSegment`
- `pPathSegments`
- `nPathSegments`

This clearly indicates that pathfinding/path-storage functionality exists.

## 6. Is It A*?

This is the important answer:

- **I cannot verify A*** from the currently visible code

### Why not

I checked:

- [WPathFinder.cpp](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/Common/WPathFinder.cpp)
- [WRoadNetwork.cpp](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/Common/WRoadNetwork.cpp)

Both are empty in this snapshot.

I also searched for obvious signs of A*:

- `AStar`
- `A*`
- open list / closed list terminology
- heuristic naming

and found no visible implementation.

### What can be said safely

There is pathfinding support.

There is not enough visible implementation to conclude:

- A*
- Dijkstra
- custom segment BFS / best-first search
- cached route-only navigation

So the correct statement is:

- **pathfinding appears to exist, but the exact algorithm is not provable from the current visible source**

## 7. How AI Uses Navigation

AI code directly uses `WRoadNav` in multiple places.

Examples found in:

- [AIActionRace.cpp](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Actions/AIActionRace.cpp)
- [AIActionTraffic.cpp](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Actions/AIActionTraffic.cpp)
- [AIVehicle.cpp](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Common/AIVehicle.cpp)
- [AICopManager.cpp](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/AI/Activities/AICopManager.cpp)

Common usage patterns:

- `InitAtPoint(...)`
- `IncNavPosition(...)`
- checking `HitDeadEnd()`
- checking `IsSegmentInCookieTrail(...)`
- checking `IsSegmentInPath(...)`
- calling `CancelPathFinding()`

### Practical conclusion

The AI mostly appears to use nav like this:

- attach current vehicle state to a road/segment/lane
- advance a future nav point
- steer/speed relative to that nav point
- optionally validate or repair path continuity

This fits the earlier observation that `Race` is a road-following chase driver.

## 8. How Collision Appears To Be Represented

Main types:

- [WCollision.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WCollision.h)
- [WCollisionTri.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WCollisionTri.h)
- [WCollider.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WCollider.h)
- [WCollisionMgr.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WCollisionMgr.h)

### Collision content types

Visible representations include:

- collision instances
- collision barriers
- collision triangles
- collision objects / OBB-like objects
- surface references

This is not a single monolithic collider.
It appears to be a world collision system that can query multiple collision primitive collections.

## 9. What `WCollider` Appears To Do

`WCollider` seems to be the per-body cached world query structure.

Visible members in [WCollider.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WCollider.h):

- requested position/radius
- last refreshed position
- cached instance list
- cached barrier list
- cached tri list
- cached object list

And methods:

- `Refresh(...)`
- `Clear()`
- `IsEmpty()`

### Practical interpretation

`WCollider` appears to:

- gather nearby relevant world collision data around an object
- cache local collision-region contents
- provide those cached lists for later collision tests and world-position updates

This looks like a local broadphase cache around a moving body.

## 10. What `WCollisionMgr` Appears To Do

`WCollisionMgr` is the main collision query and resolution interface.

Visible methods in [WCollisionMgr.h](/X:/P4Workspaces/NFSMWSourceCode/nfsmw/src/Speed/Indep/Src/World/WCollisionMgr.h):

- `GetWorldHeightAtPoint(...)`
- `GetWorldHeightAtPointRigorous(...)`
- `CheckHitWorld(...)`
- `GetInstanceList(...)`
- `GetObjectList(...)`
- `GetBarrierList(...)`
- `GetTriList(...)`
- `Collide(...)`
- `GetClosestIntersectingBarrier(...)`
- `GetWorldNormal(...)`

### Practical interpretation

`WCollisionMgr` appears to handle:

- ray/segment-style world intersection tests
- height queries
- normal queries
- gathering nearby collision instances/objects/barriers
- geometry-vs-world collision resolution

This is the main world collision query service.

## 11. How Collision Is Detected In AI

At the AI layer, one common pattern is:

- use a segment from current position to destination
- call `WorldCollision(...)` or `WCollisionMgr::CheckHitWorld(...)`
- convert the result into a simple "drivable or blocked" decision

Examples:

- `AIVehicle::UpdateTargeting()` sets `mDrivableToTargetPos` from `WorldCollision(...)`
- `AIVehicle` also uses world collision for drive-nav validity
- `AIActionPursuitOffRoad::UpdateAvoidWalls()` uses `CheckHitWorld(...)` to look ahead and generate wall-avoidance steering

### Practical conclusion

The AI mostly treats collision detection as:

- segment/raycast-style world obstruction checks
- plus broader world-position/rigid-body collision handling through physics

## 12. How Collision Is Used In Physics

Physics-side code shows world collision is not just AI-only probing.

Examples from `RigidBody`:

- `DoWorldCollisions(...)`
- `ResolveWorldCollision(...)`
- `OnWCollide(...)`

and use of:

- `WCollider`
- `WCollisionMgr`

So the same collision ecosystem supports both:

- AI/pathing visibility/drivability queries
- actual rigid-body/world collision handling

## 13. Is There A Broadphase / Spatial Partition?

Probably yes, but the current useful implementation evidence is limited.

Why this seems likely:

- `WCollisionMgr` gathers instance/object/barrier lists by region/query
- `NodeIndexList` is part of `WCollisionMgr`
- `WGrid` / `WGridNode` types exist

However:

- `WGrid.cpp` is empty in this snapshot
- so the exact spatial partition implementation is not visible here

Safe conclusion:

- the system appears to use some form of region/node/grid-based world query support
- but the exact implementation details are not currently visible enough to describe confidently

## 14. Best Current High-Level Model

Based on the visible code, the safest current model is:

### Navigation

- world road graph built from nodes, segments, intersections, and lane profiles
- per-vehicle navigation handled by `WRoadNav`
- local road geometry represented with splines and `NavCookie`s
- pathfinding support exists but algorithm not yet visible

### Collision

- world collision handled by `WCollisionMgr`
- local cached collision context held in `WCollider`
- collision content includes instances, barriers, triangles, and objects
- AI often uses segment-style collision tests to decide whether direct driving is possible

## 15. Direct Answers To The Original Questions

### "How is nav implemented?"

It appears to be implemented as a **road-network navigation system** using:

- road nodes
- road segments
- lane profiles
- local splines
- per-agent `WRoadNav`
- cookie-trail/local corridor data

### "Like waypoints?"

Not in the simple waypoint sense.

It is better described as:

- **segment/lane/spline-based road navigation**

### "How is collision detected?"

Through the world collision system centered on:

- `WCollisionMgr`
- `WCollider`
- collision instances / barriers / tris / objects

AI often uses:

- segment/ray-style checks such as `CheckHitWorld(...)`

### "How is nav done, A*?"

Pathfinding support clearly exists, but the current visible source does **not** prove A*.

So the correct answer is:

- **unknown from this snapshot**
- only that pathfinding/path segment support is present in the API

## 16. Confidence Summary

### High confidence

- nav is road-network based
- nav is lane-aware
- nav uses splines and cookie-trail-like local corridor data
- collision uses `WCollisionMgr`
- AI uses world collision checks to decide drivable direct pursuit

### Medium confidence

- there is some broader graph/path system beyond local segment following
- there is some form of region/grid-based broadphase support

### Low confidence / unknown

- exact pathfinding algorithm
- exact `WRoadNav::FindPath` implementation
- exact `WGrid` implementation details in this snapshot
