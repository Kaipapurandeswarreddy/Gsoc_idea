# InVesalius PubSub Architecture Analysis & Modernization Proposal — GSoC 2026
---

My name is K. Purandeswar Reddy, and I'm a 3rd year B.Tech Data Science student at Santhiram Engineering College. I've been exploring InVesalius over the past few days to prepare for my GSoC 2026 application, and I've come across something I think would make a strong project idea. I wanted to share what I've found and get your thoughts before putting together a formal proposal.


## What I Found

While testing InVesalius from source, I ran into crashes when switching between projects and filed issues [#1122](https://github.com/invesalius/invesalius3/issues/1122)and [#1124](https://github.com/invesalius/invesalius3/issues/1124). At first glance they look like simple bugs (assigning `np.nan` to an integer NumPy array in `tracker.py`), but when I dug deeper into the codebase to understand why, I realized they're actually symptoms of a bigger structural problem in how InVesalius components talk to each other through PyPubSub.

I did a full analysis of the event flow, and here's what I found:


## How the Event System Currently Works

InVesalius uses PyPubSub (wrapped in `invesalius/pubsub/pub.py`) as its internal message bus. About 50+ files across the codebase use it. I mapped out the architecture into four layers:

![WhatsApp Image 2026-03-02 at 8 25 31 PM](https://github.com/user-attachments/assets/04919566-c939-4ba8-b182-e412060ac25e)

In short:
- The **GUI layer** (frame.py, task_navigator.py, task_slice.py, etc.) captures user actions and publishes messages like `"Show open project dialog"`
- The **Controller layer** (control.py) subscribes to 40+ topics and orchestrates project lifecycle — open, close, save, import
- The **Data layer** (slice_.py, surface.py, volume.py, measures.py, viewer_*.py) handles rendering and state
- The **Navigation layer** (tracker.py, navigation.py, robot.py, image.py, markers.py) handles neuronavigation — this is where the crash occurs at `ResetTrackerFiducials`

All messages are dispatched **synchronously** — when `sendMessage` is called, every subscriber runs in sequence before the caller continues.


## The Close Project Chain — Where It Breaks

When a user opens a second project while one is already loaded, `Controller.CloseProject()` fires. Here's what happens step by step:

```
Controller.CloseProject()
 │
 ├─ sendMessage("Enable style")            → resets interaction mode
 ├─ sendMessage("Stop navigation")         → stops the navigator
 ├─ sendMessage("Hide content panel")      → hides the UI panels
 │
 └─ sendMessage("Close project data")      → THIS IS THE BIG ONE
      │
      ├─ viewer_slice.OnCloseProject()     → resets slice viewer
      ├─ viewer_volume.OnCloseProject()    → resets volume viewer
      ├─ slice_.OnCloseProject()           → clears slice data
      ├─ surface.OnCloseProject()          → removes surfaces
      ├─ volume.OnCloseProject()           → removes volumes
      ├─ measures.OnCloseProject()         → clears measurements
      ├─ data_notebook.OnCloseProject() ×4 → clears data panels
      ├─ task_slice.OnCloseProject() ×2    → resets slice tasks
      ├─ task_surface.OnCloseProject()     → resets surface task
      ├─ task_tractography.OnCloseProject()→ resets tractography
      ├─ task_mepmapping.OnCloseProject()  → resets MEP mapping
      ├─ default_viewers.DisablePreset()   → disables presets
      ├─ slice_menu._close()              → resets slice menu
      ├─ mep_visualizer.OnCloseProject()   → resets MEP viz
      ├─ surface_geometry.OnCloseProject() → resets geometry
      │
      └─ NavigationPanel.OnCloseProject()  ← CRASH HAPPENS HERE
           │
           ├─ tracker.ResetTrackerFiducials()  ← ValueError!
           │     assigns np.nan to integer array
           │
           ├─ sendMessage("Disconnect tracker")
           ├─ sendMessage("Delete all markers")
           ├─ sendMessage("Remove tracts")
           └─ ... 4 more cascading messages
```

24+ handlers are subscribed to `"Close project data"`. They all run synchronously, one after another. There is **no error isolation** — when `ResetTrackerFiducials` throws the `ValueError`, PyPubSub stops right there. Any subscriber that hasn't run yet simply doesn't. The app is now half-closed: the session says "closed" but viewers, surfaces, and slices still hold old data. That's why issue #1124 reports that everything breaks afterwards — manual edition, 3D surfaces, fMRI — all of them crash because they're operating on stale objects from a project that was never fully cleaned up.


## The Three Core Problems

I think this goes beyond just fixing the NaN bug. The event system has three structural issues:

**1. No error isolation.** PyPubSub dispatches synchronously. If subscriber number 15 out of 24 throws an exception, subscribers 16 through 24 are simply skipped. There's no try/except around individual handlers. One bad subscriber can corrupt the entire application state.

**2. String-based topics with no registry.** There are roughly 200+ unique topic strings scattered across 50+ files (things like `"Close project data"`, `"Update threshold limits"`, `"Reset tracker fiducials"`). There's no central place where these are defined. A typo in a topic string — for example, `"Close Project data"` vs `"Close project data"` — silently fails with no warning. You can't use IDE tooling to refactor or trace them.

**3. Deep cascading chains.** Subscribers frequently publish new messages inside their own handlers. `NavigationPanel.OnCloseProject` publishes 6 more messages from within a `"Close project data"` handler, each of which triggers its own chain of subscribers. A single `CloseProject()` call ends up invoking 50+ handlers in a deeply nested synchronous call stack. This makes the event flow extremely hard to follow and debug.


## What I'm Proposing

A phased project to modernize the event system, fixing the immediate problems first and then gradually upgrading to a type-safe architecture:


### Phase 1 — Harden the Existing System (Weeks 1–3)

This is the quick win. No new libraries, no API changes — just making the current system robust.

**Error isolation in `sendMessage`:** Wrap each subscriber call in a try/except inside the existing `pub.py` wrapper. If a handler crashes, log the error and continue to the next handler. This single change would have prevented issue #1124.

**Centralized topic registry:** Create a `topics.py` file with all ~200 topic strings as constants. Replace the string literals across the codebase. Now IDE autocomplete works, and a typo becomes a `NameError` instead of a silent failure.

**Fix the known bugs:** Fix the NaN-to-integer issue in `tracker.py` and audit `LoadState` to enforce `dtype=np.float64`.

Deliverable: All existing tests pass, issues #1122 and #1124 are resolved, and the event system is resilient to individual subscriber failures.


### Phase 2 — Build the Typed Event Bus (Weeks 4–6)

I noticed the codebase already uses `@dataclasses.dataclass` in several places — for example, the `Marker` class in `data/markers/marker.py` is a well-structured dataclass with typed fields and serialization methods. The idea here is to extend that same pattern to the event system, making event topics first-class typed objects instead of raw strings:

```python
# Instead of this:
Publisher.sendMessage("Enable state project", state=True)

# You write this:
@dataclass(frozen=True)
class EnableStateProject(Event):
    state: bool

bus.publish(EnableStateProject(state=True))
```

This follows the same approach the project already uses for markers, but applied to events. The IDE knows `state` is a `bool`, autocomplete works, and passing a wrong type is caught immediately.

The existing `add_sendMessage_hook` in `pub.py` is perfect for building a bridge adapter — old PyPubSub code and new EventBus code can coexist during migration without breaking anything.

Deliverable: Working EventBus class with error isolation, bridge adapter, and event dataclasses for the project lifecycle group (~15 events).


### Phase 3 — Migrate Components (Weeks 7–11)

Migrate the codebase file by file, starting with the most impactful files:

1. `control.py` (40+ subscriptions — the hub)
2. Data layer (`slice_.py`, `surface.py`, `volume.py`, `viewer_slice.py`, `viewer_volume.py`)
3. Navigation layer (`tracker.py`, `navigation.py`, `robot.py`, `image.py`)
4. GUI panels (`task_navigator.py`, `task_slice.py`, `task_surface.py`, etc.)

Each migration is backward-compatible through the bridge adapter. Existing tests should keep passing at every step.

Deliverable: All 50+ files migrated to the typed event bus. PyPubSub dependency removed.


### Phase 4 — Documentation and Testing (Week 12)

Write developer documentation explaining the new event system. Add tests that verify the error isolation behavior and the event flow for critical paths like project closing.

Deliverable: Documentation, tests, and a clean PR ready for merge.


## Why This Matters

It is the kind of infrastructure improvement that makes every other feature more reliable. The navigation team won't have to worry about one subscriber crashing the entire close flow. Developers adding new features get IDE support and type safety. Debugging event-driven bugs becomes tractable instead of hunting through 50 files for string matches.

I noticed the ideas page also had "Fix GUI inconsistencies" and "Convert GUI from WXPython to Qt" as considered items — I think this proposal addresses the same underlying motivation (improving the application's internal architecture) in a much more practical and less risky way than a full Qt migration.


## About Me

I am a 3rd-year B.Tech (CSE–Data Science) student with a strong interest in machine learning and software systems. I have worked with medical imaging workflows using MONAI pipelines and 3D Slicer, where I explored segmentation and dataset handling. Recently, I started running InVesalius from source and learning open-source development by reproducing bugs and reading the codebase — which is how I ended up discovering the pubsub architecture issues described above.

I'd love to hear your thoughts on whether this would be a good fit for GSoC 2026. I'm happy to share the detailed analysis I've done so far, or adjust the scope if you think it should be smaller or larger.

**Note on AI usage:** The bug discovery, codebase investigation, and the overall analysis are entirely my own work — I filed issues #1122 and #1124 after encountering the crashes while testing from source, and traced the pubsub flow by reading the code. I did use AI as a tool to help navigate the large codebase faster and to illustrate the architecture diagrams and flow charts. The conclusions and proposed solution are based on my own understanding of the system.

Thanks for your time,  
K. Purandeswar Reddy  
purandeswarreddykaipa@gmail.com  
GitHub: https://github.com/Kaipapurandeswarreddy
