# Admin 2026 — Related RSC Project Research

_Last updated: 2026-09-26_

This is a focused landscape check of public GitHub projects related to RuneScape Classic / RSC that may overlap with, complement, or inform Admin 2026.

The goal is not to catalog every RSC repository. The goal is to identify reusable ideas, code, data, operational patterns, and adjacent projects that could materially improve this server or the planned admin dashboard.

## Executive summary

Several existing projects are highly relevant:

1. **Open-RSC/Core-Framework** is the strongest source of operational/admin feature ideas. It has a mature staff hierarchy, large admin/moderator command surface, server-management commands, moderation controls, player inspection concepts, and ongoing development as of September 2026.
2. **Open-RSC/Website-Portal** already implements many web-admin concepts we were independently considering: player lookup, bank/inventory views, chat logs, private-message logs, staff logs, trade logs, login history, item/NPC databases, world-map views, and admin tasks.
3. **2003scape/rsc-world-map** is directly reusable or adaptable for Admin 2026's future live map. It is framework-free JavaScript, accepts object/point overlays, supports planes and search, and uses the same coordinate/data ecosystem as this repo.
4. **2003scape/rsc-data-server** already exposes useful multi-world and online-player primitives that should be treated as an existing data source instead of reimplemented.
5. **RSCPlus/rscplus** contains two especially useful concepts: replay/session recording and a structured server-extension mechanism. These are relevant to debugging, support tooling, and future client-aware admin features.
6. **2003scape/rsc-landscape** and **rsc-path-finder** can support world inspection, static/dynamic map tooling, collision overlays, path visualization, and debugging.
7. **Open-RSC/rsc-c** and **damiantw/rsc-docker** are useful for deployment/client ecosystem lessons, but are less directly relevant to Admin 2026 itself.
8. **RSCGo** is interesting as an alternate server implementation, but appears effectively dormant since 2018 and has less immediate value than OpenRSC for admin/dashboard work.

The most valuable immediate takeaway is that Admin 2026 should borrow **feature taxonomy and operational lessons** from OpenRSC, while reusing **native 2003Scape libraries** wherever possible.

---

## High-value projects

## 1. Open-RSC/Core-Framework

Repository:

- https://github.com/Open-RSC/Core-Framework

Status observed:

- active
- latest observed commit: 2026-09-16
- default branch: `develop`
- license: AGPLv3

Why it matters:

OpenRSC appears to be the most feature-complete actively developed open-source RSC server ecosystem in this landscape.

Its README describes:

- multiple server configurations
- configurable custom features
- auction house
- clans
- parties
- pets
- holiday events
- custom quests
- custom sprites
- world configuration options

More importantly for Admin 2026, the repository contains an extensive staff and command system.

### Existing staff roles

Observed role concepts include:

- Owner
- Admin
- Super Moderator
- Moderator
- Developer
- Event
- regular Player

This is useful evidence that our planned role/capability model is solving a real operational problem.

We should not necessarily copy these exact roles. Admin 2026's capability-oriented model is still preferable, but these roles are useful examples when designing default role bundles.

### Existing admin/moderator command surface

OpenRSC documents a large number of commands, including examples such as:

- save all players
- graceful restart/update
- immediate shutdown
- spawn/remove world items
- spawn NPCs
- modify player stats
- heal/recharge players
- inspect IP-related state
- freeze experience
- skull/unskull
- mutate inventory/bank contents
- world/landscape reload
- scheduled holiday drops/events
- moderation alerts
- bans/mutes/IP bans
- player possession/observation tooling

This is one of the best sources we found for deciding which actions belong in a server-admin dashboard.

### What to borrow

Use OpenRSC primarily as a **feature checklist and operational reference**.

Potential Admin 2026 features inspired by it:

- graceful restart with countdown
- server save-all
- moderation actions
- staff role/capability bundles
- world event controls
- entity spawn/remove tools
- player stat/inventory/bank tooling
- player observation tools
- world reload/debug controls
- staff command audit history
- IP/account relationship investigation

### What not to copy blindly

OpenRSC's in-game command interface should not become our primary admin architecture.

Admin 2026 should still:

- expose explicit server-side commands
- require authorization per capability
- audit mutations
- keep the web API separate from the game packet protocol
- avoid arbitrary command-string execution from the browser

A useful pattern is:

```text
OpenRSC command concept
          ↓
Admin 2026 explicit command type
          ↓
validated server-side handler
          ↓
audit event
```

For example:

```text
::saveall
→ world.saveAll
```

or:

```text
::item
→ player.giveItem
```

### Additional security lesson

OpenRSC configuration includes optional restrictions around admin login IP ranges.

That reinforces our existing preference to make Admin 2026:

- private-network/localhost bound by default
- authenticated
- capability-gated
- optionally restricted by network origin

---

## 2. Open-RSC/Website-Portal

Repository:

- https://github.com/Open-RSC/Website-Portal

Status observed:

- active
- latest observed commit: 2026-08-21
- default branch: `develop`
- license: GPLv3

This project is especially relevant because it already implements many of the information views we were planning.

Observed web views/features include:

- player list
- player lookup/search
- inventory views
- bank views
- item statistics
- NPC/monster database
- item database
- quest pages
- world map
- chat logs
- private-message logs
- staff logs
- trade logs
- login history
- rename logs
- generic logs
- auction logs
- admin task page
- invite-management tooling
- webserver information

### Why this matters

This confirms that Admin 2026's proposed information architecture maps well to actual RSC operational needs.

It also gives us a mature reference for:

- which player fields operators care about
- useful moderation log filters
- economy/admin queries
- navigation structure
- staff-only data visibility
- cross-linking items and players

### Strong candidates to study before building equivalent screens

Before implementing the following Admin 2026 areas, inspect the corresponding OpenRSC implementation:

#### Player inspector

Study:

- `playerlist.blade.php`
- player detail/search controllers
- inventory views
- bank views

Questions worth extracting:

- what fields are most useful?
- what searches do admins actually need?
- how are large inventories/banks displayed?
- how do they cross-reference item IDs?

#### Moderation/logs

Study:

- `chat_logs.blade.php`
- `pm_logs.blade.php`
- `staff_logs.blade.php`
- `loginlist.blade.php`
- `rename_logs.blade.php`

These can inform our event taxonomy and future analytics schema.

#### Economy

Study:

- trade logs
- auction logs
- item statistics

These likely contain useful ideas for historical event storage.

### Privacy note

The OpenRSC portal includes private-message logging views.

That does **not** mean Admin 2026 should automatically implement private-message visibility.

Our existing privacy rule still stands: private communication visibility should be an explicit policy decision with documented retention and access rules.

### Architecture lesson

The OpenRSC portal is database-driven and separate from the game process.

Admin 2026 is aiming for a more live, event-oriented control surface.

We can combine the strengths:

- live state from the game server
- durable history from an analytics/audit database later

---

## 3. 2003scape/rsc-world-map

Repository:

- https://github.com/2003scape/rsc-world-map

Status observed:

- historically maintained around 2021
- license: AGPLv3

This is probably the clearest direct reuse opportunity.

Features include:

- interactive RSC world map
- framework-free JavaScript
- multiple zoom levels
- plane switching
- search/autocomplete
- labels
- points of interest
- object overlays
- mobile/touch support
- embeddable into any block element

Critically, its object coordinates come directly from `@2003scape/rsc-data`, which is the same ecosystem used by this server.

### Admin 2026 opportunity

Rather than building a world map renderer from scratch, we should evaluate embedding or adapting this package for Phase 4.

Potential live overlays:

- online players
- NPCs
- ground items
- game objects
- wall objects
- recent deaths
- moderation targets
- selected player's movement
- pathfinding routes
- spawn locations

A plausible implementation:

```text
rsc-world-map
   +
static rsc-data overlays
   +
Admin 2026 WebSocket live entity positions
   =
live operations map
```

### Recommendation

Move the Phase 4 "live world map" task from a greenfield assumption to:

> evaluate `@2003scape/rsc-world-map` as the base renderer before writing new map code.

---

## 4. 2003scape/rsc-data-server

Repository:

- https://github.com/2003scape/rsc-data-server

Status observed:

- original 2003Scape-era implementation
- latest observed commit: 2021-01-17

The data server already provides several operations relevant to Admin 2026.

Documented features include:

- multiple-world support
- player persistence
- friend communication across worlds
- SQLite backend
- player counts
- world list
- online status
- login handling
- cross-world player world lookup
- hiscore ranking

Relevant handlers include:

- `worldConnect`
- `worldDisconnect`
- `worldGetList`
- `playerCount`
- `playerOnlineCount`
- `playerGetWorlds`

### Admin 2026 implication

Do not calculate or duplicate information that the data server already owns.

For example, future multi-world dashboard status should probably combine:

```text
rsc-server live world state
+
rsc-data-server world registry
```

The data server could become a natural source for:

- registered world list
- total registered players
- cross-world online state
- per-world population

### Potential future work

We may eventually want a small set of admin-specific read handlers on the data server, but that should be considered carefully.

For Phase 0–2, prefer using the existing game-server connection where possible rather than expanding the persistence protocol immediately.

---

## 5. RSCPlus/rscplus

Repository:

- https://github.com/RSCPlus/rscplus

Status observed:

- latest observed commit: 2025-07-13
- license: GPLv3

RSC+ is a heavily enhanced RuneScape Classic client.

Features include:

- replay/session recording
- replay playback
- debug modes
- NPC/player/item information overlays
- position overlay
- configurable UI enhancements
- server extensions
- world subscriptions

Two concepts are particularly interesting for Admin 2026.

### Replay architecture

RSC+ can record and replay sessions using real server/client data.

Potential Admin 2026 applications:

- support/debug session capture
- regression reproduction
- packet/problem analysis
- moderation evidence
- bug reproduction
- content verification

We should **not** make packet replay part of early phases.

But the future "Full replay tooling" back-burner item has credible prior art.

A lighter first step could be:

- structured server events
- optional recent-event ring buffer
- downloadable diagnostic bundle

before any full packet replay system.

### Server Extension framework

RSC+ has a formal mechanism for server-specific client features.

It includes:

- extension identifiers
- world IDs
- server-controlled world metadata
- validation of server endpoints
- server-specific client behavior

This suggests a longer-term opportunity:

Admin 2026 could expose some metadata intended for compatible enhanced clients.

Examples:

- world status
- maintenance notices
- server feature flags
- world labels/types

This is not core admin functionality, but it could become a clean integration point if this server is ever paired with RSC+.

### Security lesson

RSC+'s extension/world-subscription system performs endpoint/domain validation to prevent malicious server metadata from redirecting credentials.

The broader lesson is useful: configuration downloaded from a server must still be treated as potentially dangerous and validated against trusted origins.

---

## 6. 2003scape/rsc-landscape

Repository:

- https://github.com/2003scape/rsc-landscape

License:

- AGPLv3

This package is already a dependency of our server.

It can:

- deserialize landscape archives
- inspect tile properties
- retrieve tiles by game coordinates
- generate static world maps
- dump world sectors to JSON
- render sectors/maps to canvas

### Admin 2026 possibilities

Useful for:

- tile inspector
- terrain/collision debugging
- dungeon/plane visualization
- map rendering
- identifying blocked/indoor/bridge terrain
- future world editor/debug tooling

Because the running server already holds a `Landscape` instance, Admin 2026 may be able to expose selected tile information without loading another copy.

Avoid serializing whole landscape sectors through the API unless explicitly requested.

---

## 7. 2003scape/rsc-path-finder

Repository:

- https://github.com/2003scape/rsc-path-finder

License:

- AGPLv3

This package is also already used by the server.

It supports:

- path queries
- object/wall collision
- line-of-sight
- map/collision rendering

### Admin 2026 possibilities

Potential developer/debug tools:

- show route from A to B
- inspect why movement fails
- visualize blocked tiles
- show line-of-sight
- inspect door/wall state
- compare player path versus expected path

This strengthens the Phase 4 pathfinding/debug tooling idea.

Again, use the existing running `world.pathFinder` instance rather than creating duplicate state.

---

## 8. Open-RSC/rsc-c

Repository:

- https://github.com/Open-RSC/rsc-c

Status observed:

- active into late 2025
- portable C99 client
- works with 2003Scape/OpenRSC-compatible servers
- supports native and browser/WebAssembly builds

Interesting features include:

- responsive/resizable UI
- searchable bank
- ground-item text
- wiki lookup
- configurable status displays
- browser build

### Relevance

This is more relevant to future client work than to Admin 2026.

However, it is worth tracking if we ever want:

- a modern browser client
- an embedded spectator/debug client
- a standardized client for local development

For the admin dashboard itself, reuse is probably low.

---

## 9. damiantw/rsc-docker

Repository:

- https://github.com/damiantw/rsc-docker

This is a modern containerized self-hosting project around OpenRSC and a browser client.

It demonstrates:

- Docker Compose-based deployment
- bundled server/database/client
- WebSocket-accessible browser client
- scripted server setup
- admin-rank bootstrap

### Relevance

This is useful primarily for Phase 0 deployment research.

It suggests that a future local development environment for this repo could provide:

```text
docker compose up
```

for:

- rsc-data-server
- rsc-server
- admin dashboard
- browser client

That could substantially reduce onboarding friction.

Do not make containerization a prerequisite for early Admin 2026 work.

---

## 10. Zlacki/RSCGo

Repository:

- https://github.com/Zlacki/RSCGo

Status observed:

- Go implementation
- protocol 204
- latest observed commits: July 2018

### Relevance

RSCGo is useful as an alternate interpretation of the same RSC protocol and server model.

It may help when:

- validating packet behavior
- comparing architecture
- investigating ambiguous protocol behavior

It appears much less valuable than OpenRSC for operational/admin feature reuse because it has not seen comparable recent development.

Treat it primarily as a protocol/reference implementation.

---

## Other 2003Scape ecosystem projects worth remembering

### 2003scape/rsc-data

Repository:

- https://github.com/2003scape/rsc-data

This remains important because Admin 2026 can use it to turn raw IDs into human-readable content.

Examples:

- NPC names
- item names
- object definitions
- quest information
- region information
- spawn locations
- shops

This means many admin DTOs can include both:

```json
{
  "id": 11,
  "name": "Chicken"
}
```

without introducing a separate metadata database.

### 2003scape/rsc-models

Repository:

- https://github.com/2003scape/rsc-models

Potential future use:

- asset inspection
- model previews
- developer/content tooling

A 3D model viewer inside Admin 2026 would be fun but is clearly optional/back-burner.

### 2003scape/rsc-map-scrape

Repository:

- https://github.com/2003scape/rsc-map-scrape

Mostly useful as provenance/tooling for map POI data.

Probably not needed at runtime.

### 2003scape/rsc-www

Repository:

- https://github.com/2003scape/rsc-www

Important architectural precedent:

- Next.js frontend
- Express backend
- direct connection to `rsc-data-server`

Admin 2026 independently selected a similar frontend direction.

Before scaffolding the dashboard, inspect this project for:

- old Next.js integration assumptions
- data-server client reuse
- period-themed UI components that may be useful

Do not copy its old build setup wholesale without checking modern compatibility.

---

## Feature overlap matrix

| Admin 2026 idea | Existing prior art | Likely action |
|---|---|---|
| Player list/search | OpenRSC Website Portal | Study UX/query model |
| Player detail | OpenRSC Website Portal | Study fields; build live DTO version |
| Inventory/bank viewer | OpenRSC Website Portal | Study UX; use our live Player model |
| Staff roles | OpenRSC Core | Use as reference; keep capability model |
| Moderation commands | OpenRSC Core | Convert useful concepts into explicit commands |
| Command audit logs | OpenRSC staff logs | Study schema/filters |
| Chat logs | OpenRSC Website Portal | Study event/history shape |
| Trade history | OpenRSC Website Portal | Study schema/event fields |
| Login history | OpenRSC Website Portal | Study schema/event fields |
| Live world map | 2003scape rsc-world-map | Strong reuse candidate |
| Static world map | rsc-landscape | Reuse |
| Collision/path debugger | rsc-path-finder | Reuse running instance |
| Multi-world list | rsc-data-server | Reuse existing handler/data |
| Player population | rsc-data-server | Reuse existing handler/data |
| Replay/debug sessions | RSC+ | Future inspiration |
| Server-specific client features | RSC+ extensions | Future integration |
| Docker dev stack | rsc-docker | Consider after baseline |
| Item/NPC metadata | rsc-data | Direct reuse |
| 3D model preview | rsc-models | Back burner |

---

## Recommended near-term changes to our plan

This research does not require changing the overall Admin 2026 architecture.

It does suggest several refinements.

### 1. Add an explicit "study OpenRSC command taxonomy" step before Phase 2

Before defining the full Admin 2026 command set:

- review `Open-RSC/Core-Framework/Commands.md`
- categorize commands into:
  - moderation
  - player support
  - developer/debug
  - world operations
  - content/event operations
- only implement commands with clear operational value

### 2. Treat rsc-world-map as the preferred Phase 4 starting point

Do not begin a custom map renderer until this library has been prototyped.

### 3. Use OpenRSC Portal as a historical analytics reference

When designing the event/audit database, inspect its log views first.

Fields and filters that have survived real-world use are valuable signals.

### 4. Make rsc-data lookups first-class in DTOs

Human-readable names should be resolved server-side when inexpensive.

Avoid forcing the dashboard to maintain a parallel item/NPC metadata database.

### 5. Inventory existing rsc-data-server capabilities during Phase 0

Before designing status/multi-world APIs, identify which values already come from the data server.

### 6. Keep replay tooling deferred

RSC+ proves the idea is viable, but it is substantially more complex than normal telemetry.

Structured events should come first.

### 7. Consider a containerized developer environment later

After we establish the actual modern Node/runtime requirements, a Compose stack may be a good onboarding improvement.

---

## Licensing notes

Licenses observed:

- this repository / 2003Scape server: AGPLv3+
- Open-RSC/Core-Framework: AGPLv3
- 2003scape/rsc-world-map: AGPLv3
- 2003scape/rsc-landscape: AGPLv3
- 2003scape/rsc-path-finder: AGPLv3
- Open-RSC/Website-Portal: GPLv3
- RSCPlus/rscplus: GPLv3
- RSCGo: Unlicense/public-domain dedication

Because this project is already AGPL-licensed, code reuse from AGPL-compatible 2003Scape/OpenRSC server projects may be practical, but any direct code copying should still preserve required attribution/notices and be reviewed case-by-case.

For GPL-only client/web projects, prefer:

- ideas
- UX patterns
- data-shape inspiration

unless we deliberately verify license compatibility for the specific code being reused.

This document is not legal advice.

---

## Suggested follow-up research

Not required before starting Phase 0, but valuable later:

- inspect OpenRSC's command authorization implementation in detail
- inspect OpenRSC's staff log database schema
- inspect OpenRSC player detail/bank/inventory controllers
- inspect OpenRSC world-map implementation
- compare OpenRSC player/account schema with rsc-data-server
- inspect RSC+ replay format only if replay tooling becomes active work
- inspect the 2003Scape world-map package against a modern frontend build
- inspect whether `rsc-www` contains reusable data-server client abstractions

---

## Current recommendation

Proceed with the existing Admin 2026 architecture, but treat these projects as reference implementations instead of reinventing RSC-specific operational behavior.

The strongest practical combination appears to be:

```text
2003Scape server architecture
+
2003Scape data/map/pathfinding libraries
+
OpenRSC admin/moderation feature lessons
+
OpenRSC portal analytics/inspection lessons
+
RSC+ replay/extension ideas for later phases
```

That gives us a modern admin architecture while benefiting from years of RSC-specific operational experience already present in the open-source ecosystem.
