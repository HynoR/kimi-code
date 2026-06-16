# Kimi Web server feasibility and design

> Internal design note, not user-facing documentation. Do not wire this file into VitePress navigation.

## Summary

`kimi-web` should be a separate product entry from `kimi`. The existing `kimi` binary keeps the current CLI/TUI runtime. The new `kimi-web` binary starts a WebSocket server that owns conversation state and streams state projections to a thin visual panel.

The recommended implementation is a new workspace package at `apps/kimi-web`, with its own `package.json`, `src/main.ts`, build scripts, tests, and eventual native packaging. It must not register a subcommand in `apps/kimi-code`, and it must not import `apps/kimi-code/src/*` or TUI modules.

The server should use `@moonshot-ai/kimi-code-sdk` as the runtime boundary, so config, auth, model resolution, session creation, session resume, prompt execution, approval handling, and question handling stay aligned with the current product behavior. In the first implementation, `kimi-web` only consumes existing config/auth state; it does not implement model/provider configuration, login, or OAuth setup flows.

## Goals

- Provide a separate binary, `kimi-web`, for server mode.
- Keep `kimi` and its TUI code path unchanged.
- Let `kimi-web` read the same Kimi Code config and auth state as `kimi`, without becoming a config-management surface.
- Let the server, not the panel, own all authoritative conversation state.
- Stream assistant text, thinking, tool calls, tool progress, tool results, status, approvals, and questions through one WebSocket connection.
- Support approval and question interactions by letting the panel send intent responses, while the server resolves SDK handlers.
- Minimize future merge conflicts by keeping web-specific implementation under `apps/kimi-web`.

## Non-goals

- Do not add `kimi web` to `apps/kimi-code/src/main.ts` or `apps/kimi-code/src/cli/commands.ts`.
- Do not move or refactor TUI controllers for web support.
- Do not make the visual panel a full client that reconstructs state from raw SDK events.
- Do not import `@moonshot-ai/agent-core` from `apps/kimi-web` for runtime behavior unless a later SDK gap is explicitly identified and fixed in the SDK.
- Do not expose raw SDK events as the public panel protocol.
- Do not implement model provider setup, `/connect`-style configuration, config editing, login, OAuth device flow, or token management in `kimi-web` initially.

## Scope cut: agent work only

The first `kimi-web` product scope is the agent work mainline:

- session create/resume/open
- prompt submit/steer/abort
- assistant and thinking streaming
- tool-call streaming and results
- approval and question interaction for active agent work
- status, usage, goal, subagent, and background-task visualization
- reconnect, replay, and snapshot recovery

Configuration and account flows stay outside the first server scope. If the model is not configured, credentials are missing, or OAuth login is required, `kimi-web` should surface a typed server state/error that tells the panel the agent cannot start. The actual remediation remains the existing `kimi` / `kimi login` / provider setup path.

## Product and package boundary

The two product entries should stay physically separate:

```text
apps/kimi-code/
  bin: kimi
  owner: CLI, TUI, print mode, ACP mode

apps/kimi-web/
  bin: kimi-web
  owner: WebSocket server runtime
```

`apps/kimi-web` should depend on workspace packages, not on `apps/kimi-code`:

- Allowed runtime dependencies: `@moonshot-ai/kimi-code-sdk`, `@moonshot-ai/protocol`, `@moonshot-ai/kimi-telemetry` if telemetry is needed, and a small HTTP/WebSocket server library.
- Forbidden runtime dependencies: `apps/kimi-code/src/*`, TUI components/controllers, `@moonshot-ai/agent-core` direct imports for runtime behavior.

This keeps `apps/kimi-code/src/main.ts`, `apps/kimi-code/src/cli/commands.ts`, `apps/kimi-code/src/cli/run-shell.ts`, `apps/kimi-code/src/tui/kimi-tui.ts`, and `apps/kimi-code/src/tui/controllers/*` out of the change set.

## Workspace and packaging impact

Adding `apps/kimi-web/package.json` is enough for pnpm workspace discovery because `pnpm-workspace.yaml` already includes `apps/*`.

`flake.nix` still has hardcoded workspace lists. Initial integration must add:

- `./apps/kimi-web` to `workspacePaths`
- `@moonshot-ai/kimi-web` to `workspaceNames`

If `kimi-web` gets its own Nix/native packaging, add a separate derivation and output for `kimi-web`. Do not reuse or modify the existing `kimi-code` derivation or native scripts as the primary path, because current native packaging is hardcoded around the `kimi` binary and `apps/kimi-code`.

Suggested package scripts:

```json
{
  "name": "@moonshot-ai/kimi-web",
  "private": true,
  "bin": {
    "kimi-web": "dist/main.mjs"
  },
  "scripts": {
    "dev": "tsx src/main.ts",
    "build": "tsdown",
    "typecheck": "tsc -p tsconfig.json --noEmit",
    "test": "vitest run"
  }
}
```

Suggested commands:

```sh
pnpm --filter @moonshot-ai/kimi-web run dev -- --port 3100
pnpm --filter @moonshot-ai/kimi-web run build
pnpm --filter @moonshot-ai/kimi-web run typecheck
pnpm --filter @moonshot-ai/kimi-web run test
```

Root `pnpm -r run build` will include `apps/kimi-web` once it defines `build`, so the first scaffold must keep the build stable. Root `typecheck` and root Vitest do not currently include arbitrary app packages automatically, so `kimi-web` should start with explicit package-level verification and only later be added to root CI gates.

## Config, auth, and session parity

`kimi-web` should create its own harness:

```ts
const harness = createKimiHarness({
  identity: {
    userAgentProduct: 'kimi-web',
    version,
  },
  uiMode: 'web',
  telemetry,
  onOAuthRefresh,
});
```

Startup should follow the same behavior as the current app runtimes:

1. Install the global proxy dispatcher before constructing network clients.
2. Create a `KimiHarness` through `createKimiHarness`.
3. Call `harness.ensureConfigFile()`.
4. Call `harness.getConfig()` and `harness.getConfigDiagnostics()`.
5. Surface config diagnostics through server status events or startup logs.
6. Use `harness.auth.status()` only for read-only auth readiness checks in the first implementation.

Do not parse `config.toml` in `apps/kimi-web`. The SDK already owns default config paths, `KIMI_CODE_HOME`, config salvage, environment model overlays, provider resolution, OAuth token refresh, and session lifecycle.

Do not add server endpoints or WebSocket controls for editing `config.toml`, adding provider models, running `/connect`, launching OAuth login, or writing tokens. Those are separate product surfaces. Keeping them out of `kimi-web` is part of the isolation strategy.

Session operations should use SDK APIs:

- `harness.createSession(...)`
- `harness.resumeSession(...)`
- `harness.listSessions(...)`
- `harness.closeSession(...)`
- `session.prompt(...)`
- `session.steer(...)`
- `session.cancel()`
- `session.onEvent(...)`
- `session.setApprovalHandler(...)`
- `session.setQuestionHandler(...)`
- `session.getStatus()`
- `session.getResumeState()`

For cwd-sensitive resume behavior, prefer the print-mode pattern: use `listSessions({ sessionId, workDir })` before `resumeSession` when the request targets a specific working directory. Do not copy ACP's looser resume behavior unless that difference is explicitly intended.

## Server-owned state model

The panel is a visualization surface. It must not own canonical state or infer state from raw agent events.

`kimi-web` should maintain, per session:

- `sessionId`, `workDir`, title, status, model, thinking level, permission mode, plan mode
- current prompt and prompt queue
- current turn id and step
- transcript messages and live assistant/thinking drafts
- active tool calls keyed by `turnId:toolCallId`
- tool progress and tool results
- goal snapshot
- background task snapshot
- subagent snapshot
- pending approval requests
- pending question requests
- usage and context status
- monotonic WebSocket sequence number
- bounded replay buffer

The reducer should process SDK events first, update the authoritative state, append a state event to the replay buffer, then fan out to connected panels. WebSocket push failure must not roll back server state.

Important event ordering rule: `tool.call.delta` can arrive before `tool.call.started`. The server must lazily create a pending tool call from the first delta, then upgrade it when `tool.call.started` arrives. Use `turnId:toolCallId` as the public tool key to avoid raw tool id collisions across turns.

## WebSocket protocol shape

Reuse `@moonshot-ai/protocol` for the existing WS envelope concepts:

- server hello
- client hello
- subscribe/unsubscribe
- `last_seq_by_session`
- ack envelope
- ping/pong
- abort
- error
- `resync_required`

Add a `kimi-web` business event layer in `@moonshot-ai/protocol` or locally in `apps/kimi-web` first, then promote once stable. The public stream should be state-oriented:

```text
server -> panel
  server.hello
  session.snapshot
  session.patch
  runtime.not_ready
  prompt.started
  prompt.completed
  prompt.failed
  prompt.aborted
  interaction.pending
  interaction.resolved
  resync_required
  error

panel -> server
  client.hello
  session.open
  session.resume
  prompt.submit
  prompt.steer
  prompt.abort
  approval.respond
  question.respond
  ping/pong
```

The first implementation can keep all traffic on one WebSocket. If HTTP endpoints are useful for debugging, keep them read-only and mirror the same server-owned state.

`runtime.not_ready` covers missing model configuration, missing credentials, invalid config diagnostics, or auth-required states. It should be informational and actionable for the panel, but it must not become a login/config mutation API.

## Prompt, queue, and cancel behavior

The SDK has one active turn per session. If the server wants queue semantics, the queue belongs to `kimi-web`.

Recommended behavior:

1. `prompt.submit` mints `prompt_id` and `user_message_id`.
2. If the session is idle, mark the prompt `running` and call `session.prompt(...)`.
3. If a turn is active, enqueue the prompt in server state.
4. On `turn.started`, bind `prompt_id` to `turnId`.
5. On `turn.ended`, mark the active prompt `completed`, `failed`, or `aborted`.
6. After completion, start the next queued prompt.

For cancel:

- If the prompt is queued, remove it from the queue and emit `prompt.aborted`.
- If the prompt is active, call `session.cancel()` and resolve any related pending approval/question as cancelled or dismissed.
- If the prompt already ended, return an idempotent non-aborted response with the latest known sequence.

## Approval and question handling

Register handlers immediately after session creation or resume:

```ts
session.setApprovalHandler((request) => approvalBridge.handle(request));
session.setQuestionHandler((request) => questionBridge.handle(request));
```

The bridge should:

1. Mint `approval_id` or `question_id`.
2. Attach `session_id`, current `turn_id`, related `tool_call_id`, `created_at`, and `expires_at`.
3. Store a pending resolver in server state.
4. Emit `interaction.pending`.
5. Await `approval.respond` or `question.respond` from the panel.
6. Resolve the SDK handler.

Failure policy:

- Approval transport failure, timeout, disconnect without another connected panel, prompt abort, or session close resolves as `{ decision: 'rejected' }` or `{ decision: 'cancelled' }` according to the interaction.
- Question failure, timeout, prompt abort, or session close resolves as dismissed/null.
- Unknown approval decisions must never approve by default.

Unlike ACP, `kimi-web` should support the full question schema: multiple questions, multi-select, and free-form "other" answers. ACP's single-question fallback is a protocol limitation and should not be copied.

## Reconnect and replay

Every state event sent to the panel should have a per-session `seq`. The panel reconnects with `last_seq_by_session`.

Reconnect behavior:

- If all missing events are still in the ring buffer, replay `seq > last_seq`.
- If the buffer cannot satisfy the request, emit `resync_required`.
- After `resync_required`, send a full `session.snapshot`.

SDK live deltas are not replayable after process restart. If `kimi-web` restarts during an active prompt, it should treat the old WebSocket state as lost, emit or expose `session_recreated`, and rebuild the closest explainable state from `session.getResumeState()` after resume. This recovers history and final persisted context, but not token-by-token deltas that were live-only.

## File layout proposal

```text
apps/kimi-web/
  AGENTS.md
  README.md
  package.json
  tsconfig.json
  tsdown.config.ts
  vitest.config.ts
  src/
    main.ts
    cli.ts
    server.ts
    harness.ts
    state/
      reducer.ts
      store.ts
      replay-buffer.ts
      transcript.ts
    protocol/
      inbound.ts
      outbound.ts
      schemas.ts
    session/
      manager.ts
      prompt-queue.ts
      approval-bridge.ts
      question-bridge.ts
    transport/
      websocket.ts
      http.ts
    telemetry.ts
  test/
    reducer.test.ts
    prompt-queue.test.ts
    approval-bridge.test.ts
    websocket-protocol.test.ts
```

`apps/kimi-web/AGENTS.md` should be added with local rules:

- Do not import from `apps/kimi-code/src/*`.
- Do not modify `apps/kimi-code` to add web behavior.
- Use `@moonshot-ai/kimi-code-sdk` as the core runtime boundary.
- Keep panel protocol state-oriented, not raw SDK event-oriented.
- Run package-level `typecheck`, `test`, and `build` after changes.

## Implementation phases

### Phase 0: scaffold and no-op server

- Add `apps/kimi-web` package.
- Add `kimi-web` bin.
- Add package-local build, typecheck, and tests.
- Add `flake.nix` workspace path/name.
- Do not add native packaging yet.
- Do not touch `apps/kimi-code`.

### Phase 1: config/auth/session parity

- Create harness with `uiMode: 'web'`.
- Install proxy dispatcher.
- Run `ensureConfigFile`, `getConfig`, and diagnostics.
- Expose server startup state and read-only auth readiness.
- Emit `runtime.not_ready` for missing model/config/auth prerequisites.
- Do not implement login, OAuth, provider setup, or config editing.
- Implement create/resume/list session through SDK.

### Phase 2: state reducer and streaming

- Subscribe through `session.onEvent`.
- Implement transcript and tool-call reducer.
- Implement `session.snapshot` and `session.patch`.
- Add per-session sequence and replay buffer.
- Cover assistant, thinking, tool, turn, status, goal, background, and subagent events.

### Phase 3: prompt queue and interactions

- Implement `prompt.submit`, queue, active prompt tracking, and cancel.
- Register approval and question handlers.
- Implement `approval.respond` and `question.respond`.
- Add timeout and disconnect policies.

### Phase 4: protocol hardening

- Move stable protocol schemas to `@moonshot-ai/protocol` if they start local.
- Add resync tests.
- Add malformed message tests.
- Add reconnect tests.

### Phase 5: packaging and release

- Add separate native packaging only after JS server behavior is stable.
- Add independent Nix output `.#kimi-web`.
- Decide whether `@moonshot-ai/kimi-web` is private/internal or published.
- If private/internal, update changeset ignore policy to avoid release noise.

## Verification plan

Package-local checks:

```sh
pnpm --filter @moonshot-ai/kimi-web run typecheck
pnpm --filter @moonshot-ai/kimi-web run test
pnpm --filter @moonshot-ai/kimi-web run build
```

Repository checks after initial workspace integration:

```sh
pnpm -r run build
pnpm run lint
node scripts/check-nix-workspace.mjs
```

If Nix output is added:

```sh
nix build .#kimi-web
```

Do not claim `kimi-web` is covered by root `typecheck` or root Vitest until root scripts and `vitest.config.ts` explicitly include it.

## Open risks

- `@moonshot-ai/protocol` has generic WS envelopes but not yet a complete `kimi-web` business event union. Start local or add protocol schemas deliberately.
- SDK events use camelCase while protocol resource shapes use snake_case. The server needs explicit mappers.
- Protocol background task types do not exactly match SDK background task types. The server should define a stable web projection instead of leaking either shape directly.
- Server restart during an active prompt cannot recover live-only deltas. Document this as resync behavior unless persistent server event logs are added.
- Prompt queue semantics are server-owned and need tests. SDK `prompt` itself does not provide a durable queue.
- Changes to `packages/node-sdk` should only happen after a concrete SDK gap is proven. The default implementation should stay in `apps/kimi-web`.

## Decision

The direction is feasible and aligned with the isolation requirement. Implement `kimi-web` as a separate `apps/kimi-web` product with its own binary and build pipeline. Share behavior through SDK and protocol packages only. Keep current `kimi`/TUI code untouched except for repository-level workspace maintenance that is unavoidable when adding a new app.
