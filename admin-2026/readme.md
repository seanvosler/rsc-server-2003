# Admin 2026

A modern server administration, observability, moderation, and live-world tooling layer for the 2003Scape RuneScape Classic emulator.

This directory is intentionally additive. The first goal is to define a safe architecture and implementation plan before changing the legacy emulator runtime.

## Project intent

Admin 2026 should make a running RSC world inspectable and operable without turning the game server into a web application.

The game server remains authoritative. The dashboard observes state, subscribes to events, and sends explicit validated admin commands.

The desired end state is a live RSC operations console that can answer questions such as:

- Who is online and what is each player doing?
- What is happening in the world right now?
- Which plugins are loaded and which hooks are firing?
- Are ticks healthy?
- Are saves, sockets, pathfinding, and the data server healthy?
- What items are being created, destroyed, traded, dropped, or sold?
- Which quests and content plugins are producing errors?
- What moderation and administrative actions have occurred?
- Can an authorized operator safely teleport, message, mute, kick, inspect, or otherwise assist a player?

## Architectural principle

Do not expose mutable game objects directly to the dashboard.

Use a narrow admin boundary:

```text
Browser dashboard
      |
      | HTTP requests
      | WebSocket events
      v
Admin bridge / control API
      |
      | explicit DTOs, queries, commands, events
      v
Existing rsc-server runtime
      |
      +-- World
      +-- Player / NPC / Entity models
      +-- packet handlers
      +-- plugins
      +-- shops / skills / quests
      +-- rsc-data
      +-- rsc-data-server
```

The existing emulator remains runnable without the dashboard.

## Existing server architecture to integrate with

The current server is a CommonJS Node.js application.

Important integration points include:

- `src/server.js`
  - owns transport setup
  - handles TCP and WebSocket clients
  - queues inbound and outbound messages
  - owns `world` and `dataClient`
- `src/model/world.js`
  - authoritative world simulation
  - 640 ms tick interval
  - owns players, NPCs, objects, wall objects, ground items, shops, plugins, pathfinding, landscape, saves
- `src/model/player.js`
  - authoritative live player state
  - stats, fatigue, quests, inventory, bank, prayer, trade, social state, position, combat-related state
- `src/model/*`
  - entity, inventory, bank, trade, shop, NPC, character, local entity and world abstractions
- `src/packet-handlers/*`
  - user/game commands received from clients
- `src/plugins/*`
  - game content and hook-driven behavior
- `@2003scape/rsc-data`
  - definitions, locations, quest metadata and world data
- `rsc-data-server`
  - persistence and cross-world data services

Admin 2026 should work with these systems instead of duplicating them.

## Proposed stack

### Admin bridge

Preferred direction:

- Node.js
- TypeScript where practical
- in-process with the game server initially
- REST-style HTTP API for queries and commands
- WebSocket stream for live events
- lightweight runtime validation for external input
- explicit role-based authorization
- append-only audit records for mutations

The bridge may begin as JavaScript if introducing TypeScript into the legacy build would create unnecessary friction. The API contracts should still be written as if they are typed.

Avoid introducing a large framework into the game process unless it has a clear benefit.

### Dashboard

Recommended:

- Next.js
- React
- TypeScript
- TanStack Query
- Tailwind CSS
- shadcn/ui or similarly lightweight component primitives

The browser must never have direct access to internal server objects or unrestricted game-server execution.

### Historical/admin data

For initial development, persistent historical telemetry is optional.

When needed:

- PostgreSQL
- Drizzle ORM or Prisma
- event/audit tables separate from gameplay persistence

Do not overload `rsc-data-server` with analytics workloads unless there is a compelling compatibility reason.

### Optional later infrastructure

Only add these when scale requires them:

- Redis for multi-world pub/sub and shared ephemeral state
- Prometheus for metrics
- Grafana for deep operational dashboards
- OpenTelemetry for traces and structured telemetry

Do not introduce Kafka, Kubernetes, or a microservice topology for a single-world implementation.

## Core contracts

The admin surface should distinguish four concepts.

### Queries

Read-only requests for current authoritative state.

Examples:

- `world.status`
- `players.list`
- `players.get`
- `entities.search`
- `plugins.list`
- `plugins.get`
- `shops.list`

### DTOs

Serialized, stable representations of game state.

Examples:

- `WorldStatus`
- `PlayerSummary`
- `PlayerDetails`
- `EntitySummary`
- `PluginStatus`
- `QuestProgress`
- `ServerMetrics`

DTOs should be deliberately smaller and safer than the underlying game objects.

### Commands

All mutations should use explicit command types.

Example:

```json
{
  "type": "player.teleport",
  "playerId": 42,
  "x": 120,
  "y": 640,
  "reason": "admin intervention"
}
```

Potential commands:

- `player.teleport`
- `player.message`
- `player.kick`
- `player.mute`
- `player.giveItem`
- `world.broadcast`
- `world.saveAll`
- `plugin.reload` if safe reload semantics are implemented later

Every command should:

1. validate input
2. check operator authorization
3. execute through authoritative game APIs
4. produce a result
5. write an audit event
6. emit a live admin event where appropriate

Do not let the dashboard PATCH arbitrary player/world properties.

### Events

Events describe things that already happened.

Examples:

- `player.login`
- `player.logout`
- `player.chat`
- `player.death`
- `npc.death`
- `trade.completed`
- `item.created`
- `item.destroyed`
- `plugin.invoked`
- `plugin.error`
- `world.tick`
- `world.save.started`
- `world.save.completed`
- `admin.command.executed`

Start with a small event vocabulary and expand it deliberately.

## Internal event bus

Introduce an internal admin/telemetry event bus rather than coupling dashboard code to arbitrary model methods.

A simple Node `EventEmitter` is sufficient initially.

Conceptually:

```js
adminEvents.emit('player.login', {
    id: player.id,
    username: player.username,
    x: player.x,
    y: player.y
});
```

The WebSocket transport subscribes to the bus.

The same event stream can later feed:

- audit storage
- metrics
- debugging
- moderation rules
- replay tooling
- multi-world aggregation

Do not emit sensitive fields such as passwords, authentication material, private server credentials, or raw session secrets.

## Dashboard information architecture

### Overview

Show:

- world ID
- members/F2P mode
- uptime
- server version
- connected player count
- entity counts
- current and rolling tick duration
- memory usage
- data-server connection state
- last successful global save
- socket/client counts
- recent warnings/errors

### Players

Searchable online-player table with:

- username
- rank
- combat level
- coordinates
- current activity where inferable
- fatigue
- health
- session duration

Player inspector should eventually expose:

- skills
- inventory
- bank
- equipment bonuses
- quest stages
- quest points
- fatigue
- prayers
- position
- nearby entities
- movement queue
- trade state
- social lists where appropriate
- moderation state
- recent server-side events

Sensitive information should be role-gated.

### World inspector

Inspect:

- NPCs
- game objects
- wall objects
- ground items
- shops
- spawn definitions
- coordinates
- entity IDs
- respawn state
- nearby players

A future map view can visualize live positions and entities.

### Plugins and content

Expose:

- loaded plugin modules
- registered hook types
- invocation counts
- execution timing
- last invocation
- thrown errors
- referenced entity/item/object IDs when metadata exists

Long-term plugin metadata may take a form such as:

```js
module.exports = {
    meta: {
        id: 'quests.free.dragon-slayer',
        name: 'Dragon Slayer',
        category: 'quest',
        version: 1,
        admin: {
            inspectable: true,
            reloadable: false
        }
    },

    onTalkToNPC,
    onNPCDeath
};
```

Do not require an immediate rewrite of all existing plugins. Metadata must be incremental and optional.

### Moderation

Potential features:

- live public chat stream
- player search
- mute/unmute
- kick
- broadcast
- player notes
- operator action history
- suspicious activity/event views

Any private-message visibility must be an explicit product and privacy decision, not an incidental consequence of instrumentation.

### Economy

Historical telemetry can eventually answer:

- item creation/destruction rates
- item abundance
- shop purchases/sales
- trade volume
- GP creation/destruction
- high-value transfers
- unusual duplication patterns

Economy analytics should consume emitted events rather than changing gameplay code solely for reporting.

### Developer tools

Potential tools:

- event stream
- structured logs
- tick profiler
- slow plugin report
- packet debugging
- pathfinding inspection
- entity neighborhood inspection
- command console using predefined commands

Never add arbitrary remote JavaScript execution.

## Security model

Treat the admin API as privileged infrastructure.

Initial requirements:

- bind privately by default
- authenticated sessions
- CSRF-safe mutation model where applicable
- explicit roles
- command-level authorization
- server-side input validation
- audit every mutation
- no secrets in browser payloads
- no raw arbitrary code execution
- no direct database credentials exposed to the dashboard

Suggested roles:

- `viewer`
- `moderator`
- `game-master`
- `admin`

Authorization should be capability-based where possible rather than assuming every authenticated user can perform every action.

## Performance rules

The dashboard must not destabilize the 640 ms world loop.

Therefore:

- avoid expensive serialization on every tick
- do not walk every entity graph for every dashboard connection
- aggregate metrics incrementally
- rate-limit expensive queries
- sample high-frequency telemetry
- batch event delivery where useful
- keep historical writes asynchronous from gameplay logic
- never block the world tick waiting for dashboard clients

Operational telemetry should degrade gracefully if the dashboard or analytics store is unavailable.

## Multi-world path

Build the first version for one world but make identifiers explicit.

Include fields such as:

- `worldId`
- `playerId`
- stable event IDs where historical storage exists
- event timestamps

A future control plane may aggregate several game-server processes, but a single server should not require Redis or an external broker.

## Initial API sketch

Possible first endpoints:

```text
GET  /admin/api/status
GET  /admin/api/players
GET  /admin/api/players/:id

POST /admin/api/commands/player/message
POST /admin/api/commands/player/teleport

GET  /admin/api/events   -> WebSocket upgrade or separate /admin/ws
```

Exact routing is not important yet. Stable contracts are.

## First vertical slice

The recommended first implementation milestone is intentionally small.

### Server side

Implement:

- admin bridge bootstrap
- authenticated/private development access
- `WorldStatus` DTO
- `PlayerSummary` DTO
- `PlayerDetails` DTO
- `GET /status`
- `GET /players`
- `GET /players/:id`
- internal event bus
- WebSocket connection
- `player.login`
- `player.logout`
- one safe command such as `player.message` or `player.teleport`
- mutation audit logging

### Frontend

Implement:

- application shell
- server status card
- live online-player list
- player details page/drawer
- basic event feed
- UI for the single implemented command

### Acceptance criteria

The first slice is successful when:

1. the existing game server still starts normally
2. gameplay works without the dashboard running
3. the dashboard can observe live players
4. login/logout updates arrive without manual refresh
5. an authorized operator can perform one audited command
6. dashboard failure does not stop or materially delay the world loop

## Suggested repository shape

This directory may eventually evolve into something like:

```text
admin-2026/
├── readme.md
├── AGENTS.md
├── packages/
│   └── contracts/
│       └── ...
├── server/
│   └── ...
└── web/
    └── ...
```

This is a direction, not a requirement. Avoid scaffolding empty packages until implementation work begins.

The legacy server-side integration may ultimately live under `src/admin/` if that provides cleaner runtime wiring. Shared contracts may live under `admin-2026/packages/contracts/`.

## Phased roadmap

### Phase 0 — architecture and baseline

- document integration points
- verify current server boots on the chosen Node version
- record baseline tick/memory behavior
- define DTO, command and event conventions
- establish auth strategy for local development

### Phase 1 — observation

- status API
- players API
- live event stream
- dashboard shell
- player inspector
- basic metrics

### Phase 2 — controlled operations

- role system
- message
- teleport
- kick
- mute
- broadcast
- save-all
- audit history

### Phase 3 — plugin observability

- plugin metadata convention
- invocation instrumentation
- timing/error reporting
- plugin/content explorer
- quest/content diagnostics

### Phase 4 — world tooling

- entity inspector
- shop inspector
- spawn inspection
- live map
- pathfinding/debug views

### Phase 5 — historical analytics

- PostgreSQL event store
- moderation history
- economy flows
- quest completion analytics
- operational trends

### Phase 6 — multi-world control plane

Only if actually needed:

- world registry
- shared authentication
- centralized dashboard
- Redis/event transport
- cross-world operations and analytics

## Known legacy concerns

Do not assume the inherited code is bug-free.

One example already identified in `src/server.js` is the socket close path deleting `this` from `incomingMessages` even though the map is keyed by socket. That likely deserves a separate fix and regression coverage.

Expect additional issues around:

- dependency age
- Node version compatibility
- network error handling
- authentication assumptions
- plugin exceptions
- lack of automated tests
- browser-specific behavior
- persistence failure behavior

Do not combine unrelated legacy cleanup with an admin feature unless the admin work depends on it.

## Non-goals for the first release

- rewriting the emulator in TypeScript
- replacing `rsc-data-server`
- replacing the packet protocol
- rewriting all plugins
- supporting arbitrary code execution
- building a distributed microservice platform
- storing every tick or every packet indefinitely
- making admin UI state authoritative

## Definition of done for Admin 2026

The project should eventually provide a secure, low-overhead, live view into an RSC world with auditable operator controls while preserving the emulator as the single source of gameplay truth.
