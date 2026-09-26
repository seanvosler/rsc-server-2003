# AGENTS.md

Guidance for agentic coding workers contributing to Admin 2026.

This file describes the project context, architectural constraints, implementation priorities, and working rules an autonomous coding agent should understand before changing code.

## Mission

Build a secure, low-overhead administration and observability layer for the legacy 2003Scape RuneScape Classic server without destabilizing or unnecessarily rewriting the emulator.

The game server remains authoritative.

The dashboard:

- reads explicit serialized views of game state
- subscribes to explicit admin/telemetry events
- sends explicit validated commands
- never mutates arbitrary server objects directly

Read `admin-2026/readme.md` before implementing features.

## Repository context

This is a legacy Node.js/CommonJS RuneScape Classic emulator.

Important facts:

- default branch: `master`
- package: `@2003scape/rsc-server`
- current package version in this repo: `1.1.0`
- game tick interval: 640 ms
- no established automated test suite is present in the repository
- current code and dependencies are primarily from 2021
- the server can run as:
  - normal TCP server
  - WebSocket server
  - browser WebWorker server
- persistence and account-related services are delegated to `rsc-data-server`
- game definitions and locations come substantially from `@2003scape/rsc-data`

Do not assume contemporary Node, browser, build-tool, or package behavior without verifying it.

## First files to inspect

Before modifying the runtime, inspect at minimum:

- `package.json`
- `src/server.js`
- `src/model/world.js`
- `src/model/player.js`
- `src/model/character.js`
- `src/model/entity.js`
- `src/model/entity-list.js`
- `src/model/local-entities.js`
- `src/plugins/index.js`
- relevant files under `src/packet-handlers/`
- relevant plugin files under `src/plugins/`

If work touches persistence, inspect:

- `src/data-client.js`
- `src/browser-data-client.js`
- any related protocol code
- the corresponding `rsc-data-server` repository if available

If work touches browser mode, inspect:

- `src/browser-index.js`
- `src/browser-socket.js`
- browser-specific imports and `process.browser` branches

## Architecture boundaries

### Preserve the game runtime

Do not make the dashboard required for the game server to start or function.

Admin initialization should fail soft whenever practical.

A dashboard outage must not:

- stop the world tick
- disconnect normal clients
- block player saves
- prevent game startup unless the operator explicitly configured admin as mandatory

### Do not expose model objects directly

Do not serialize `Player`, `World`, `NPC`, `Inventory`, or other domain objects wholesale.

Reasons:

- circular references
- accidental secrets
- unstable contracts
- large payloads
- unexpected getters/side effects
- privilege leakage
- coupling UI behavior to legacy internals

Create DTO mapper functions instead.

Good:

```js
function toPlayerSummary(player) {
    return {
        id: player.id,
        username: player.username,
        x: player.x,
        y: player.y,
        combatLevel: player.combatLevel
    };
}
```

Bad:

```js
res.json(player);
```

### Mutations are commands

Administrative writes must use explicit command handlers.

Do not implement generic object patching such as:

```text
PATCH /players/:id
```

with arbitrary property bodies.

Prefer:

```text
POST /commands/player/teleport
POST /commands/player/message
POST /commands/player/kick
```

Each mutation should:

1. validate payload
2. resolve the target safely
3. authorize the operator
4. call game-domain behavior
5. capture success/failure
6. audit the attempt
7. emit an admin event if appropriate

### Events describe facts

Prefer past-tense conceptual semantics.

Examples:

- player logged in
- player logged out
- NPC died
- admin command executed
- plugin invocation failed

Do not use the event bus as a hidden RPC system.

## Integration strategy

Prefer the smallest viable integration into the legacy server.

Likely server-side shape:

```text
src/
  admin/
    index.js
    event-bus.js
    dto/
    commands/
    auth/
    transport/
```

The dashboard application and shared contracts may live under:

```text
admin-2026/
  web/
  packages/
    contracts/
```

Do not force this structure if implementation evidence suggests a cleaner one.

## TypeScript policy

The dashboard should generally use TypeScript.

The legacy game server is CommonJS JavaScript.

Do not begin by converting the emulator to TypeScript.

For the admin bridge, choose one of these deliberately:

1. CommonJS JavaScript with typed schemas/contracts externally
2. TypeScript compiled to code compatible with the existing runtime

Prefer the option with the smallest build/runtime disruption.

A full TypeScript migration of the legacy emulator is out of scope unless explicitly requested.

## API contract conventions

Keep contracts stable and boring.

Recommended common envelope fields where useful:

```ts
type AdminEvent<T> = {
  id?: string;
  worldId: number;
  type: string;
  timestamp: string;
  data: T;
};
```

Use:

- stable IDs
- ISO timestamps at transport/storage boundaries
- explicit `worldId`
- explicit nullability
- bounded payloads

Avoid:

- serializing class instances
- implicit field meanings
- undocumented numeric enums
- deeply nested copies of world state

## Player identity

Do not assume username is the best stable identifier.

Prefer internal player/account ID where available for commands and historical records.

Usernames can be included for display.

When targeting a live player, handle the case where the player logs out between query and command execution.

## Event bus

Start simple.

Node's `EventEmitter` is acceptable.

The event bus should not become gameplay authority.

Keep event publication cheap.

If an event requires expensive enrichment, publish a minimal event and enrich it outside the tick-sensitive path.

Do not synchronously perform database writes, remote requests, or expensive formatting inside event emission called from hot gameplay code.

## Tick-loop safety

The 640 ms tick is a hard architectural concern.

Before adding code to any hot path, consider:

- frequency
- object count
- serialization cost
- allocation rate
- log volume
- downstream blocking
- fan-out to multiple dashboard clients

Avoid:

- serializing all players every tick
- serializing all entities every tick
- synchronous telemetry storage
- unbounded event queues
- verbose per-tick logging in production
- slow async work awaited by the tick

Prefer:

- incremental counters
- periodic snapshots
- bounded queues
- sampled metrics
- asynchronous drainers
- per-client subscription filters

## Plugin system

The current plugin mechanism is function-name based.

Recognized hooks are defined in `src/model/world.js`.

Do not break existing plugin exports.

Any metadata extension must be optional and backward compatible.

Desired future capability:

```js
module.exports = {
    meta: {
        id: 'quests.free.example',
        name: 'Example Quest',
        category: 'quest'
    },
    onTalkToNPC
};
```

But agents must first verify how plugin modules are flattened and loaded before changing export shape.

The current generated `src/plugins/index.js` expects existing module structures. Changing plugin export conventions without tracing this loader can silently disable content.

## Build-index awareness

The repository contains `build-index.js` and a generated plugin index.

Before modifying plugin loading:

- inspect `build-index.js`
- determine whether `src/plugins/index.js` is generated
- preserve browser bundling constraints
- do not manually create a convention that the build process will overwrite

## Browser mode

Browser/WebWorker operation is an unusual but intentional feature.

Changes in shared files such as `src/server.js` and `src/model/world.js` can affect browser builds.

Admin features do not necessarily need to operate in browser mode.

If admin functionality is server-only:

- guard Node-only imports carefully
- avoid unconditional imports of HTTP, filesystem, crypto, database, or other Node-only admin dependencies from browser-bundled modules
- ensure `npm run build-browser` remains viable

Prefer isolating Node-only admin bootstrap behind a runtime branch.

## Security rules

The admin surface is privileged.

Never implement:

- arbitrary remote JS evaluation
- arbitrary module loading from user input
- arbitrary SQL execution
- unrestricted filesystem browsing
- unrestricted shell execution
- direct exposure of credentials
- generic mutation of world/player object properties

Admin input must be treated as untrusted.

Validate:

- player IDs
- coordinates
- item IDs
- quantities
- text lengths
- enum values
- pagination sizes
- filter syntax

Commands with potentially destructive effects should have explicit capability checks.

## Authentication and authorization

Do not invent a production identity provider prematurely.

For early local development, a deliberately scoped auth mechanism is acceptable.

However, architecture should distinguish:

- authentication: who is the operator?
- authorization: may this operator perform this command?

Suggested roles:

- viewer
- moderator
- game-master
- admin

Prefer capability checks internally, for example:

```text
players.read
players.message
players.teleport
players.kick
players.mute
world.broadcast
world.save
plugins.inspect
plugins.manage
```

Role-to-capability mapping can then evolve without rewriting command handlers.

## Audit rules

Every state-changing admin command should generate an audit record.

Minimum useful fields:

- timestamp
- world ID
- operator ID
- command type
- target
- sanitized input summary
- success/failure
- error category if failed
- correlation/request ID where available

Do not write secrets into audit logs.

Do not store passwords, session tokens, auth headers, or raw credential material.

## Privacy

Instrumentation must be intentional.

Public chat may reasonably be considered operational/moderation data depending on project policy.

Private messages should not automatically become visible merely because the server can technically observe them.

Before instrumenting private communications:

- require an explicit product decision
- document retention
- document authorized roles
- minimize exposure

## Error handling

Admin failures should normally fail closed for the admin action and fail soft for gameplay.

Examples:

- malformed admin command -> reject command
- dashboard disconnected -> game continues
- telemetry DB unavailable -> drop/buffer telemetry according to policy, game continues
- auth backend unavailable -> deny privileged actions
- event subscriber throws -> isolate subscriber from game logic

Never let an admin WebSocket client exception bubble into the world tick.

## Logging

Prefer structured log fields for new admin systems.

Avoid logging:

- passwords
- data-server password
- auth tokens
- entire Player objects
- inventory/bank snapshots unless intentionally debugging with safe access controls

Log enough context to correlate admin actions without leaking unnecessary player data.

## Testing strategy

The repository currently lacks a mature test harness.

For Admin 2026, introduce tests incrementally around new boundaries rather than attempting to test the entire emulator immediately.

Priority order:

1. pure DTO mapping tests
2. command validation tests
3. authorization tests
4. command handler tests with small fakes
5. API contract tests
6. event bus behavior
7. integration smoke test against a running server

When fixing legacy bugs discovered during admin work, add the smallest regression test feasible.

Do not create giant mocking frameworks around the whole world model.

## Known issue worth separating

A likely bug exists in `src/server.js`:

```js
this.incomingMessages.set(socket, []);
```

but the socket close handler appears to call:

```js
this.incomingMessages.delete(this);
```

Investigate and fix separately from unrelated admin work, ideally with regression coverage.

Do not opportunistically bundle broad cleanup into the first admin PR.

## Dependency policy

The base repository is old.

Before introducing a dependency:

- verify current maintenance status
- check Node compatibility
- check CommonJS/ESM implications
- check browser bundling impact
- prefer small dependencies
- avoid adding infrastructure libraries for problems built-in Node APIs solve adequately

Do not upgrade the entire dependency tree just to add the admin dashboard.

Dependency modernization should be isolated and justified.

## API versioning

The first private implementation does not need elaborate semantic API versioning.

Still, organize routes/contracts so versioning can be introduced without a rewrite.

For example:

```text
/admin/api/v1/status
/admin/api/v1/players
```

or keep a version constant in shared contracts.

Do not prematurely support multiple API versions.

## Suggested first vertical slice

Agents should prefer completing one end-to-end feature before building broad scaffolding.

Recommended sequence:

### Step 1: status DTO

Expose read-only values such as:

- world ID
- members flag
- player count
- NPC count
- object counts
- ticks
- uptime
- process memory where server-side
- current/rolling tick timing if available

Do not add expensive scans just to generate status.

### Step 2: players list

Return small summaries for online players.

Support bounded pagination/filtering if the API shape warrants it.

### Step 3: player detail

Return a safe snapshot with selected:

- stats
- position
- fatigue
- inventory
- quest progress

Do not expose socket or server references.

### Step 4: event stream

Publish login/logout first.

Wire WebSocket delivery with bounded per-client buffering.

### Step 5: one command

Implement either:

- player message
- teleport

Message is lower-risk; teleport proves model mutation more thoroughly.

Whichever is selected, include validation, authorization, audit and result handling.

### Step 6: dashboard

Build:

- world status
- online player list
- player inspector
- live login/logout feed
- UI for the implemented command

Then reassess architecture before expanding.

## Git/change discipline

Keep changes reviewable.

Prefer:

- one architectural concern per commit
- small vertical slices
- explicit commit messages
- no drive-by formatting across legacy files
- no unrelated dependency churn

When modifying legacy files, preserve surrounding style unless there is a strong reason not to.

Do not reformat an entire large file for a two-line change.

## Agent workflow

Before coding:

1. read this file
2. read `admin-2026/readme.md`
3. inspect relevant runtime files
4. identify browser-mode impact
5. identify tick-loop impact
6. identify security/authorization impact
7. identify a minimal test strategy

During implementation:

1. keep admin code isolated
2. preserve legacy runtime behavior
3. validate all external input
4. avoid unbounded queues
5. avoid blocking hot paths
6. add tests around new pure boundaries
7. update project documentation if contracts change

Before declaring work complete:

1. run available lint/build commands
2. run any new tests
3. run browser build if shared runtime code changed
4. verify normal server startup
5. verify server still works with admin disabled
6. verify unauthorized mutation is rejected
7. verify dashboard/admin failure does not crash gameplay
8. summarize known limitations

## Definition of an acceptable agent contribution

A contribution is good when it:

- is narrow enough to review
- preserves game authority
- is backward compatible unless explicitly stated otherwise
- has clear boundaries
- avoids unnecessary infrastructure
- respects tick-loop performance
- validates privileged input
- produces auditable mutations
- does not expose secrets
- includes reasonable verification

## Things agents should challenge

If a task proposes any of these, stop and reassess the design before implementing:

- serializing the entire world every tick
- giving the browser a raw Player object
- arbitrary admin-side code execution
- direct browser access to the game database
- synchronous analytics writes from combat/tick paths
- mandatory Redis for one world
- mandatory Kubernetes
- rewriting the emulator before shipping one admin feature
- changing all plugin exports at once
- coupling dashboard availability to game availability
- putting privileged admin commands on the public game socket protocol without a strong authentication model

## Primary project principle

Make the old server easier to understand and operate without making it harder to run.
