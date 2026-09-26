# Admin 2026 Tasklist

> This file is the canonical lightweight progress tracker for Admin 2026.
> Update it whenever work changes project state.

Keep this document short and operational. It should answer four questions:

- What is done?
- What is being worked on now?
- What is next?
- What is intentionally deferred?

## Working conventions

Use only these four states:

- **Doing** — actively in progress now. Keep this to one or two items whenever possible.
- **To Do** — agreed work that is ready or expected to happen.
- **Done** — completed work that materially changed project state.
- **Back Burner** — worthwhile ideas intentionally deferred.

When work starts, move the task into **Doing**.

When work finishes, move it into **Done** in the same PR/commit series whenever practical.

If implementation changes scope, update this file rather than letting it drift.

Keep detailed design discussion, ADR-style decisions, and long-form notes elsewhere. This file is a progress ledger, not a design document.

---

## Doing

### Phase 0 — Architecture and baseline

- [ ] Verify the server boots on the chosen current Node.js version.
- [ ] Record baseline startup steps and required companion services.
- [ ] Record baseline tick behavior, memory usage, and obvious startup/runtime warnings.
- [ ] Define initial DTO, command, and event naming/convention rules.
- [ ] Establish the initial local-development authentication approach.
- [ ] Define the minimum verification strategy for new admin code.

## To Do

### Phase 1 — Observation

- [ ] Add isolated admin bootstrap wiring.
- [ ] Add `WorldStatus` DTO.
- [ ] Add `PlayerSummary` DTO.
- [ ] Add `PlayerDetails` DTO.
- [ ] Add read-only status API.
- [ ] Add online players API.
- [ ] Add player details API.
- [ ] Add internal admin/telemetry event bus.
- [ ] Emit player login events.
- [ ] Emit player logout events.
- [ ] Add WebSocket event stream.
- [ ] Add first dashboard application shell.
- [ ] Add world/server status view.
- [ ] Add live online-player list.
- [ ] Add player inspector.
- [ ] Add basic live event feed.

### Phase 2 — Controlled operations

- [ ] Define roles and capability mapping.
- [ ] Add admin mutation audit records.
- [ ] Add player message command.
- [ ] Add player teleport command.
- [ ] Add player kick command.
- [ ] Add mute/unmute command.
- [ ] Add world broadcast command.
- [ ] Add save-all command.
- [ ] Add audit-history view.

### Phase 3 — Plugin observability

- [ ] Define backward-compatible optional plugin metadata.
- [ ] Instrument plugin invocation counts.
- [ ] Instrument plugin execution timing.
- [ ] Capture plugin errors safely.
- [ ] Add plugin/content explorer.
- [ ] Add quest/content diagnostics.

### Phase 4 — World tooling

- [ ] Add entity inspector.
- [ ] Add shop inspector.
- [ ] Add spawn inspection.
- [ ] Add live world map.
- [ ] Add pathfinding/debug views.

### Phase 5 — Historical analytics

- [ ] Introduce historical admin/event storage.
- [ ] Add moderation history.
- [ ] Add economy-flow analytics.
- [ ] Add quest/completion analytics.
- [ ] Add operational trend views.

### Phase 6 — Multi-world control plane

- [ ] Add world registry.
- [ ] Add shared admin authentication.
- [ ] Add centralized dashboard across worlds.
- [ ] Add cross-world event transport if needed.
- [ ] Add cross-world operations and analytics.

## Done

### Project setup

- [x] Create `admin-2026/readme.md` project manifest.
- [x] Create `admin-2026/AGENTS.md` agent guidance.
- [x] Create canonical lightweight task tracker.

## Back Burner

- [ ] Redis pub/sub for multi-world operation.
- [ ] Prometheus integration.
- [ ] Grafana dashboards.
- [ ] OpenTelemetry tracing.
- [ ] PostgreSQL historical analytics store.
- [ ] Plugin hot reload.
- [ ] Deep packet inspection UI.
- [ ] Full replay tooling.
- [ ] Broad legacy dependency modernization.
- [ ] Full emulator TypeScript migration.
