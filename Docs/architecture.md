# Technical Architecture

The Smart Recycling Robotic Cell separates admission, perception, planning, control, completion and observational telemetry. Direct and Recovery requests share one arm sorting pipeline. The MiniBot is a separate autonomous subsystem coordinated through recovery reservation and handoff.

## Workflow and data flow

Solid arrows describe work progression. Dotted arrows describe observations or lifecycle feedback.

```mermaid
flowchart TB
    user["User / waste selection"] --> admission["FIFO request admission<br/>mode chosen once + bounded capacity"]
    admission --> spawn["Spawn candidate validation<br/>region / separation / recovery route"]
    spawn --> observe["Physical settling + fresh matching observation<br/>SIMULATED localization"]
    observe --> evaluate{"Workspace / pickup validation"}
    evaluate -->|Direct| dispatch["Direct arm dispatch"]
    evaluate -->|Recovery| coordinator["Recovery coordinator<br/>workspace reservation"]
    coordinator --> bot["MiniBot<br/>floor-graph BFS / staging / push"]
    bot --> verify["Settle + arm-handleability check<br/>clear pickup area"]
    verify --> handoff["Handoff SAME physical item"]
    dispatch --> arm["Shared articulated arm pipeline<br/>wrist analysis / roll alignment / grasp / lift"]
    handoff --> arm
    arm --> transfer["Staged transfer / release / entry verification"]
    transfer --> cleanup["Correct-bin bottom cleanup<br/>+ authoritative cycle success"]
    bot --> home["Safe return home"]
    home -.-> reservation["Release reservation<br/>after arm acquisition + bot home"]
    arm -.-> reservation
    reservation -.-> admission

    subgraph telemetry["Observational telemetry"]
        sorted["Sorted Waste Telemetry<br/>deduplicated category + assisted totals"]
        cell["Cell Health Telemetry<br/>session / cycles / subsystem states"]
        health["Arm Health Telemetry<br/>SIMULATED load / temperature / maintenance"]
        ui["UI presenters / engineering dashboard"]
        sorted -.-> cell
        sorted --> ui
        cell --> ui
        health --> ui
    end
    cleanup -.-> sorted
    coordinator -.->|handoff identity| sorted
    coordinator -.->|attempt / mobile state| cell
    arm -.->|lifecycle / arm state| cell
    arm -.->|joint telemetry| health
    observe -.->|localization cues| ui
```

The branch validates the requested mode and current object geometry; it does not reroll a queued request. Recovery route serviceability is checked before instantiation. Both dispatch paths wait for settling and a fresh observation of the intended item.

## Responsibility boundaries

| Layer | Responsibility |
| --- | --- |
| Admission | Store accepted requests in order, enforce floor/pending limits and reserve a clear recovery workspace. |
| Perception | Produce simulated robot-relative localization, projected bounds and grasp/orientation cues from scene metadata and geometry. |
| Planning | Validate reach and endpoint clearance, construct pickup poses, select roll equivalences and plan MiniBot graph/staging routes. |
| Arm control | Execute joint-target ramps, effort-based grasping, lift, staged transfer and physical release. |
| Recovery control | Navigate, push the assigned dynamic item, verify arm suitability, clear the pickup area and return home. |
| Completion | Verify the configured destination, allow the physical fall to the bin bottom and remove the sorted item. |
| Telemetry / UI | Observe existing outcomes and states, deduplicate counts and format dashboard/camera cues. |

## Shared pipeline and failure lifecycle

Direct dispatch requests the shared sorting workflow. Recovery hands the same physical item directly to that workflow after reach verification and safe retreat; it does not spawn a replacement or duplicate the arm pipeline.

On a non-cancelled pre-handoff recovery failure, the coordinator re-evaluates the same item. An arm-handleable item can use the existing retreat/handoff/return sequence. Otherwise, safe retreat and return occur without another push; terminal unrecoverable cleanup follows confirmed home arrival and reservation release. That cleanup is not recorded as a successful sort.

Sorted telemetry joins correct-bin bottom-cleanup evidence with sorter success. Cell health reuses successful totals and observes terminal lifecycle outcomes. Arm health reads joint telemetry into a simple informational model; it never changes actuation.

## Platform boundary

Editor/macOS and WebGL share the core workflow. Web presentation reduces rendering cost. WebGL also uses an assigned-target, overlap-confirmed pusher-contact compatibility layer; native recovery uses PhysX alone. Neither platform is a real robot controller.

The architecture has no implemented external cloud, ROS, trained vision or hardware connection. Azure and digital-twin connectivity remain future work.

[Return to project showcase](../README.md)
