# HeyCLI Daemon — Hermes Integration Decision Record

**Baseline:** `asayeed95/MentalClaw` @ `8c8003f80b528377d4387b96faa2c00283168d68` (upstream `NousResearch/hermes-agent` `main`, committed 2026-09-16T21:34:59Z).
**Pre-sync fork state preserved at branch** `pre-sync-baseline-2026-04-21` (`ce98e1e`, 2026-04-21).
**Inspection date:** 2026-09-16. **Method:** file-level inspection of the checked-out tree. Nothing below was executed or load-tested.

> Status: **proposed** — pending non-author review. This record does not amend `01-NORTH-STAR.md` or `04-decisions.md`; it is an input to that decision.

---

## Context

The HeyCLI daemon requirement is a persistent, voice-first agent with two independent loops (conversation + work), durable commitments, and attachment to already-running coding-agent sessions. The question was whether to keep building that runtime in-house or adopt an existing open-source harness.

**License:** `LICENSE` in this tree is the MIT License, Copyright (c) 2025 Nous Research. Third-party/bundled dependency terms have **not** been audited; do this before shipping.

### Verified present in the tree (file paths, this SHA)

| Requirement | Evidence in tree | Read |
|---|---|---|
| Full-duplex barge-in with pre-roll capture | `apps/desktop/src/lib/voice-barge-in.ts`; Python twin referenced as `tools/voice_mode.full_duplex_listen` | Detection alone loses first syllables, so a recorder runs continuously as pre-roll; noise floor calibrated from quiet samples only and held through playback; windowed-majority trigger |
| Live voice conversation loop | `apps/desktop/src/app/chat/composer/hooks/use-voice-live-conversation.ts` | Distinguishes a stop *word* from a stop *request* via a 1.5s utterance-settle window; `onInterrupt` is a separate seam from submission |
| Wake-word detection | `tools/wake_word.py`, `tools/wake_word_engines.py`, `tools/wakewords/` | Engines present: Porcupine, openWakeWord, Sherpa KWS |
| TTS provider abstraction | `agent/tts_provider.py`, `agent/tts_registry.py`, `apps/desktop/src/lib/tts-lease.ts` | Pluggable TTS + playback leasing |
| Background work independent of the conversation | `tools/async_delegation.py`, `agent/background_review.py`, `agent/review_idle_queue.py` | `delegate_task(background=true)` returns a handle immediately; completion pushes onto a shared `completion_queue` drained while idle, so results surface as a **new turn, never mid-turn**, with de-dup and crash-recovery wiring |
| Queue survives interruption | `apps/desktop/src/app/session/hooks/use-background-queue-drain.ts`, `use-background-queue-preservation.ts` | Queue drain and preservation are separate concerns with tests |
| Durable state | 40+ `hermes_state_*.py` modules (`_wal`, `_repair`, `_rewind`, `_lockguard`, `_fts`, `_sessions`, `_timeline`) | SQLite with WAL, repair, rewind, FTS search |
| Turn crash recovery | `agent/turn_recovery.py` | — |
| Remote client transport | `gateway/relay/ws_transport.py`, `transport.py`, `auth.py`, `descriptor.py` | WebSocket; gateway **dials out** to the connector; HMAC-SHA256 bearer upgrade token with multi-secret rotation. Marked **EXPERIMENTAL — may change without a deprecation cycle** until ≥2 Class-1 platforms validate it |
| HTTP API surface | `gateway/platforms/api_server.py`, `api_server_runs.py`, `api_server_run_idempotency.py` | Run dispatch with idempotency |
| Provider breadth / BYO keys | `providers/`, `agent/credential_pool.py`, `agent/model_metadata.py`, adapters for Bedrock, Codex Responses, etc. | Custom OpenAI-compatible endpoints documented upstream |
| Extension without forking internals | `plugins/plugin_loader.py`, `plugins/*/dashboard/manifest.json` | Manifest declares `entry`, `css`, `api` (`plugin_api.py`) |
| Multiplexing prior art | `hermes_cli/gateway_multiplex_mode.py` | Refuses to fold profiles when it would double-bind a fleet; logs the blocker, never fatal |

**Scale:** 13,608 tracked files, 276 MB — 6,474 Python, 2,311 TS, 967 TSX.

### Verified absent — this is HeyCLI's remaining job

- **No iOS/mobile client.** `apps/` contains only `bootstrap-installer`, `desktop`, `shared`. No Swift/Xcode/React Native/Expo files. The voice loop above is a **desktop Electron/web** implementation; iOS audio-session and background-execution constraints are unaddressed here.
- **No live coding-session adoption.** Nothing in this tree attaches to an already-running Claude Code / Codex / Grok PTY session. `delegate_task` spawns *its own* subagents. This is the North Star acceptance sentence and it is **not** solved by adopting Hermes.
- **No single-driver fencing** for an externally-owned session.
- **No project/session alias resolution** ("Beacon" → North Sun's active worker).

### Prior art distinction

tmux/zellij multiplex terminals but have no agent loop. Anthropic Remote Control and Codex cloud spawn vendor-hosted sessions. Hermes is the closest prior art to the *agent runtime* half and ships barge-in, wake words, background delegation and durable state we would otherwise rebuild. **It is not prior art for adopting a session the user already started on their own machine** — that remains HeyCLI's differentiator, alongside the phone UX and the two-loop guarantee.

---

## Options

| Option | What it changes | Cost | Locks in | How it fails |
|---|---|---|---|---|
| **A. Keep building in-house** | Nothing | Rebuild barge-in, wake words, TTS leasing, background queue, WAL state, crash recovery | Nothing | Another quarter on substrate; the acceptance sentence stays unproven |
| **B. Extract components into HeyCLI now** | Copy voice/queue/state modules out | High: each module assumes Hermes' session, config and permission context | Copied code with no upstream path | Modules silently depend on `hermes_state_*` and `process_registry`; we inherit maintenance without the reliability |
| **C. Run Hermes as the daemon's agent engine behind a HeyCLI adapter** ⭐ | HeyCLI owns phone client, session authority, task ledger; Hermes owns model/tool loop | Integration + upstream sync + security review | A replaceable boundary, if kept thin | Relay is EXPERIMENTAL and may break; Electron voice loop does not transfer to iOS as-is |

**Recommendation: C.** B is premature — option B's cost is only knowable after C proves the boundary. The 30,739-commit gap you just closed is the argument against carrying a fork of internals.

---

## Delta

### Proposed ADR entry (for `04-decisions.md`)

```
## 2026-09-16 — Hermes as the daemon's agent engine, behind a HeyCLI adapter

Status: proposed, pending non-author review

Context: HeyCLI's daemon requirement is a persistent voice-first agent with
independent speak/work loops. Inspection of NousResearch/hermes-agent @ 8c8003f
(MIT, 2026-09-16) found full-duplex barge-in with pre-roll, three wake-word
engines, background delegation that surfaces results as a new turn via a shared
completion queue, WAL-backed SQLite state with repair/rewind, and turn crash
recovery. It contains no iOS client and no adoption of an externally-started
coding session.

Decision: Adopt Hermes as the replaceable agent engine for the daemon, behind a
HeyCLI-owned adapter. HeyCLI retains sole ownership of: the phone client and its
audio session, attention gating, project/session identity and aliases, the
durable task ledger, single-driver fencing, and adoption of already-running
agent sessions. Hermes-side orchestration is exposed only as narrow tools that
call HeyCLI's session authority — never as unrestricted shell access.

Alternatives considered: (a) continue in-house — rejected, rebuilds solved
substrate; (b) extract Hermes components into HeyCLI now — rejected as
premature, cost unknowable until the adapter boundary is proven.

Consequences: We inherit an EXPERIMENTAL relay wire contract (gateway/relay/*,
documented as changeable without deprecation) and a desktop-only voice loop that
does not transfer to iOS unchanged. Upstream sync becomes a recurring cost.
Authoritative session IDs and the task ledger MUST live outside Hermes' private
state so the engine stays replaceable.

Rollback: Delete branch heycli-daemon-integration. No change to deployed HeyCLI,
its session authority, or its task ledger. Not reversible: nothing — no
production path is touched by this record.

Ratification: proposed, pending non-author review.
```

### `docs/memory/architecture.md` amendment

Add under the daemon section — **before:** the daemon's model runtime is an in-house Anthropic client with `tools: []`. **After:**

> The daemon's reasoning engine is a vendored agent runtime (Hermes, MIT) reached through a HeyCLI-owned adapter. The engine may select among tools; authority is enforced outside it. Session identity, driver fencing, and the task ledger are HeyCLI-owned and stored outside the engine's private state, so the engine remains replaceable.

### Protocol delta

**None** for this record. No message variant changes until the adapter's first slice is specified. When it lands, both twins (`server/src/ws/protocol.ts`, `app/src/protocol/messages.ts`) change in one commit, additive only, gated on `client_hello` capabilities.

### Oracle

The one test that would prove the boundary: **while a delegated background task is in flight, an unrelated spoken question is answered without cancelling the task, and the task's completion arrives as a new turn.** Invariant asserted: *interruption ≠ cancellation*, across the adapter seam. `tools/async_delegation.py`'s "never mid-turn" contract is the upstream half; HeyCLI's ledger is the other half.

---

## Next slice (smallest provable step)

1. Run this checkout unmodified; configure one provider via API key. Confirm the voice loop and `delegate_task(background=true)` behave as the source claims.
2. Add one HeyCLI tool through the plugin manifest path — `heycli_list_sessions` — that calls HeyCLI's session authority read-only.
3. Only then attempt `heycli_send_to_session`, gated on HeyCLI-side permission and fencing.

Do not extract a single module until step 3 passes on a real machine.

## Unverified

- Any latency, reliability, or all-day-listening claim. Not measured.
- Whether the relay's experimental wire contract is stable enough for a shipped phone client.
- Third-party dependency licenses bundled in this tree.
- Whether iOS background audio policy permits the intended always-available listening posture.
