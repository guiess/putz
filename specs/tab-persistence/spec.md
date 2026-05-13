# Spec: Tab & Workspace Persistence Across Restarts

**Status:** Draft
**Owner:** PO Guardian
**Source:** User request — restore workspaces, restore tabs, restore CWDs, (P2) restore Copilot/Claude/agency sessions

---

## 1. User Scenarios & Testing

### Primary scenarios

**S1 — Workspace + tabs survive restart (essential)**
> Given Alice has 3 workspaces ("Default", "Project A", "Project B") with multiple tabs each, including terminal tabs in different cwds, an editor tab on `src/foo.ts`, and a git-graph tab — when she quits and relaunches Putz — then all 3 workspaces are present, the previously active workspace is shown, every region shows the same tab list it had, the previously active tab in each region is selected, and each terminal tab opens a fresh PTY at the directory it was in before close.

**S2 — Cwd is restored, not the live shell history (essential)**
> Given Bob's terminal tab was running `vim` in `~/projects/foo` — when he quits and relaunches — then a fresh shell starts in `~/projects/foo`. The vim process and its in-memory state are not restored; the shell is fresh.

**S3 — Editor / canvas / settings tabs are restored (essential)**
> Given Carol has a Monaco editor tab editing `src/auth.ts` and a Canvas tab — when she relaunches — then the editor reopens that file (loading from disk; unsaved changes are lost per existing editor save semantics) and the Canvas tab restores from its existing canvas-engine localStorage persistence.

**S4 — Splits and tab-bar position are restored (essential)**
> Given Dan has a workspace with a horizontal split between two regions, ratio 0.6, and tab bars on the bottom — when he relaunches — then the split structure, ratio, and tab-bar positions are identical.

**S5 — First launch after upgrade migrates cleanly (essential)**
> Given Eve upgrades from a Putz build that did NOT persist layouts — when she launches the new build — then the existing workspace metadata (names, colors) is preserved and each workspace opens with one fresh terminal tab (current behavior). No persisted layouts means no restoration; that is the expected starting state for the next session.

**S6 — Copilot CLI / Claude / agency session is re-launched (P2 / nice-to-have)**
> Given Frank had `copilot` running in a tab, and `agency claude` in another — when he relaunches — then Putz spawns the same command (with its CWD) in each tab. If the agent's own CLI exposes a `--resume <id>` flag, Putz uses it; otherwise the agent starts fresh (the user keeps their conversation history; only the agent process is re-spawned).

**S7 — User can opt out (essential)**
> Given Grace prefers a clean slate every time — when she toggles `Settings → Restore tabs on launch = off` — then on next restart no tabs are restored; each workspace opens with one fresh terminal.

**S8 — Restore failures degrade gracefully (essential)**
> Given Henry's persisted snapshot is corrupted (truncated JSON, schema mismatch, references a tab type removed in a prior version) — when he launches — then the corrupted workspace is replaced with a single fresh terminal tab, an in-app non-blocking notification reports "Could not restore tabs (corrupted state); started fresh" with a "Show details" link, and Putz must not crash on boot.

**S9 — Save cadence covers crashes (essential)**
> Given Iris's machine power-cycles unexpectedly — when she relaunches — then the snapshot from the most recent debounced save is restored (worst case: last save was up to 5 s before crash; tabs opened in that 5 s window may be missing — accepted trade-off, see Assumptions).

### Edge cases

- Cwd no longer exists at restore time → fall back to user home; surface a per-tab dim badge "cwd unavailable, opened in ~"
- Cwd exists but is not readable (permission denied) → same behavior as above
- Tab references a file that no longer exists (editor tab) → tab opens with empty buffer + path preserved + warning toast
- More than 50 tabs across all workspaces at save time → save anyway; no truncation; document `localStorage` quota implications
- A workspace was deleted in this session before restart → it stays deleted (workspace state is authoritative)
- A workspace had zero tabs at save time → it restores with zero tabs; the empty-region auto-creation in `App.tsx:316-361` only applies to the active workspace on first paint
- Concurrent multi-instance launch (user double-clicks the icon) → out of scope; relies on Tauri single-instance plugin if/when adopted

---

## 2. Requirements

### Functional

- **R1** — On app start, restore every workspace's `savedLayout` from the persisted store (currently force-cleared at `src/stores/workspaceStore.ts:80-86`).
- **R2** — A persisted `RegionTab` for `type: "terminal"` MUST carry a `cwd: string | null` field (last-known working directory). Spawn the new PTY with that cwd (`pty_spawn` already accepts `cwd`).
- **R3** — A persisted `RegionTab` MUST preserve `type`, `title`, `editorFilePath`, `editorScriptId`, `diffLeftPath`, `diffRightPath` for non-terminal tabs.
- **R4** — Tab IDs MAY be regenerated on restore; PTY `sessionId`s MUST be regenerated (the old PTYs are dead).
- **R5** — The `LayoutNode` tree (regions, splits, ratios, `tabPosition`) MUST round-trip exactly.
- **R6** — `activeWorkspaceId` and per-region `activeTabId` MUST be restored when the referenced tab/workspace still exists; otherwise fall back to the first item.
- **R7** — Persistence cadence: debounced save 1 s after any layout-changing action; flush on `before-unload` / Tauri `window-close-requested`.
- **R8** — A `Settings → Restore tabs on launch` boolean (default: `true`) gates restoration. When `false`, behavior matches today (workspace metadata only).
- **R9** — Schema version bump to v3; `migratePersistence.ts` MUST handle v1→v3 and v2→v3 by stripping unknown fields and stripping tab `type`s the running build does not recognize.
- **R10** — On migration failure or restoration failure for a single workspace, that workspace falls back to one fresh terminal tab. Other workspaces are unaffected.
- **R11 (P2)** — Persist a `command: { exec: string, args: string[] } | null` on terminal tabs when an explicit recipe was used to spawn (e.g., swarm spawn-tab). On restore, re-spawn with that command and cwd. If `command` is null, spawn a default shell.
- **R12 (P2)** — For Copilot CLI tabs (detected by `exec === "copilot"` or by swarm tab id presence), if Copilot CLI exposes a session-resume mechanism, pass the prior session id as `--resume <id>`. If not, spawn fresh. See Open Questions.
- **R13 (P2)** — Same as R12 for `claude` and `agency copilot` / `agency claude`.

### Non-functional (measurable)

- **N1 (Performance — startup)** — Restoring N tabs at launch MUST not delay the first visible paint by more than 50 ms above the baseline at N=10. PTY spawn is async after first paint.
- **N2 (Performance — save)** — Debounced save with N=50 tabs MUST complete in < 10 ms on the main thread (JSON.stringify of the snapshot).
- **N3 (Storage)** — Snapshot size for 10 tabs MUST be < 8 KB. Hard cap at 1 MB before refusing to persist (and surfacing a notice).
- **N4 (Reliability)** — Crash on corrupt persisted state: 0 occurrences. Verified by fuzz/property tests in `migratePersistence.test.ts`.
- **N5 (Privacy)** — Persisted snapshot MUST NOT contain command-line arguments that the user typed into the terminal, scrollback, env values, or PTY output. Only tab structure + cwd + (P2) the recipe-launched command.
- **N6 (Backward compat)** — Existing `putz-workspaces` localStorage payloads (schema v1 / v2) MUST migrate without user intervention.

---

## 3. Success Criteria

- **SC1** — A user with 3 workspaces × 3 terminal tabs each, distinct cwds, can quit + relaunch + see every tab back, every cwd correct. Verified by manual smoke test + Vitest scenario tests on the migrated stores.
- **SC2** — Toggling `Restore tabs on launch = off` immediately changes behavior on next launch. Verified by a Vitest test asserting the gate is checked.
- **SC3** — A corrupt snapshot (truncated, wrong schema) does not crash startup; user sees the fallback notification. Verified by a property test injecting random garbage.
- **SC4** — Snapshot includes no scrollback, no typed input, no env values. Verified by a Privacy Guardian-authored snapshot-content test.
- **SC5 (P2)** — A user who had `copilot` running can relaunch and the new tab spawns `copilot` again in the same cwd. Verified by manual test; Copilot resume only when CLI supports it (see Open Questions).

---

## 4. Assumptions

- **A1** — Workspaces remain in-app state in a single OS window (no multi-window). "Restore all workspaces" therefore means "make all workspaces' tab lists available again via the workspace switcher", not "open multiple OS windows".
- **A2** — The user's PTY processes are dead at restart; restoring "a session" means spawning a fresh shell at the prior cwd, not reattaching to a process. This matches the existing WorkspaceStore comment at `:82-83`.
- **A3** — Up to 5 s of work loss in a power-cycle is acceptable (debounced save). If unacceptable, switch to per-action sync save (perf cost — see Trade-offs).
- **A4** — `cwdRegistry` (in-memory) is the source of truth for current cwd; OSC 7 has driven it accurately enough that scope of work for cwd capture is just "read it at save time", not "build a new tracker".
- **A5** — Editor tabs reload from disk; unsaved buffer content is not persisted. This matches existing editor save semantics; expanding to autosave is out of scope.
- **A6** — Canvas tabs already persist their content via `src/lib/canvas/engine/persistence/localStorage.ts`; the new persistence only needs to remember that the Canvas tab existed and where, not its content.
- **A7 [NEEDS CLARIFICATION]** — Does Copilot CLI / Claude CLI / `agency` expose a stable `--resume` flag and a session id obtainable while running? If not, R11–R13 reduce to "re-spawn the executable with the same args and cwd; the agent reconstructs its own context from disk".
- **A8** — `localStorage` remains the persistence medium (already used for workspaces, settings, themes, bookmarks, canvas). Migration to Tauri Store / a JSON file is out of scope and tracked separately.

---

## 5. Decomposition

```
Epic: Tab & Workspace Persistence Across Restarts
  ├── T1: Persist & restore layout snapshots [essential, P0]
  │      Lift the savedLayout=null clamp; capture/restore LayoutNode + Region.tabs.
  │      Owns: workspaceStore.ts, App.tsx startup flow.
  │
  ├── T2: Capture & restore terminal CWD [essential, P0]
  │      Add `cwd` field to RegionTab snapshot; populate from cwdRegistry at save time;
  │      pass to pty_spawn at restore time; graceful fallback to home if cwd missing.
  │      Owns: RegionTab schema, layoutStore.addTerminalTab / restore path,
  │             cwdRegistry consumer.
  │
  ├── T3: Schema migration v3 + opt-out + resilience [essential, P0]
  │      Bump schema; v1→v3 / v2→v3 migrators; add Settings toggle;
  │      per-workspace fallback on corruption; user-visible failure notice.
  │      Owns: migratePersistence.ts, settingsStore.ts, Settings UI.
  │
  └── T4: Recipe / agent session restoration [P2 / nice-to-have]
         Persist explicit spawn `command` for tabs spawned via recipe (incl. copilot,
         claude, agency). On restore, re-spawn with command + cwd. Wire Copilot/
         Claude resume IFF the CLIs expose it.
         Owns: SpawnTabOptions persistence, swarm spawn-tab handler,
                src-tauri/src/swarm/spawn_recipe.rs.
```

T1, T2, T3 ship together (one PR per ticket; merge order T2 → T1 → T3 acceptable). T4 ships separately after T1–T3 are stable.

---

## 6. Guardian Consultation Results

> Inlined by PO Guardian based on codebase context (single-machine local-only terminal, no network surface for this feature). When this spec moves toward implementation, the Developer Guardian SHOULD invoke each Guardian for a fresh pass.

### Security Guardian (inline)

- **Input validation at restore** — Persisted cwd MUST be validated as an absolute path and resolved against canonical realpath BEFORE being passed to `pty_spawn` (path traversal hardening even though the user wrote it themselves; defense against tampered localStorage). The Rust side (`src-tauri/src/pty/manager.rs:91-230`) already validates cwd; ensure the same validation runs on restored values.
- **Schema validator** — Treat persisted state as untrusted input. Use a strict allowlist parser (not naive `JSON.parse` + cast). Reject unknown `RegionTab.type` values rather than coercing.
- **No code execution from snapshot** — A persisted `command` (T4) MUST be checked against the same recipe allowlist used by `src-tauri/src/swarm/spawn_recipe.rs::validate_for_spawn`. A snapshot must not be a vector for arbitrary command execution.

### Privacy Guardian (inline)

- **Snapshot content is Tier-1 (PII low risk) but cwd is borderline** — A user's cwd often contains a username (`/home/alice/...`) or project name. This is acceptable in localStorage on the same user account but MUST NOT be transmitted, logged to telemetry, or written to the swarm socket. Document this in `SECURITY.md` (see project audit).
- **No scrollback, no typed input, no env values in the snapshot** — Tested per N5 / SC4.
- **Opt-out is mandatory** (R8) — privacy-conscious users must be able to disable persistence wholesale.

### Platform Guardian (inline)

- **Storage medium** — `localStorage` is per-origin and per-user; no cross-user leakage on shared machines. Quota (~5–10 MB) is comfortably above N3 cap.
- **Concurrent instances** — Two Putz instances writing to the same localStorage cause last-write-wins. Tauri single-instance plugin adoption is tracked separately; document the limitation.

### Delivery Guardian (inline)

- **Feature flag / gradual rollout not required** — local feature, no network impact. Toggle in Settings (R8) is sufficient as user-side opt-out.
- **Telemetry** — No new telemetry; respect privacy posture. A local-only counter for restore failures (in-memory) is acceptable for diagnostics.
- **Rollback plan** — If a build ships with broken persistence, the next build's migrator MUST treat the broken-build's payload as corrupt and fall back per R10. Add a property test for this.

### Code Review Guardian (architectural impact, inline)

- **Affected components**:
  - `src/stores/workspaceStore.ts` — remove the clamp at `:80-86`, add capture-on-change + flush-on-unload
  - `src/stores/layoutStore.ts` — extend serialized RegionTab; restore path that calls `pty_spawn` per terminal tab
  - `src/utils/migratePersistence.ts` — schema v3
  - `src/types/index.ts` — add `cwd?: string` to `RegionTab` (in-memory + serialized; treat undefined as "use home")
  - `src/components/Terminal/cwdRegistry.ts` — add a `getAllCwds(): Map<sessionId, cwd>` snapshot getter
  - `src/stores/settingsStore.ts` — `restoreTabsOnLaunch: boolean`
  - `src/App.tsx:316-361` — gate the auto-create-first-tab on "no tabs were restored"
  - `src-tauri/src/ipc/terminal.rs` + `src-tauri/src/pty/manager.rs` — no schema change required (already accept `cwd` and `args`); validate restored cwds run through the same allowlist
- **Affected contracts**:
  - `localStorage` key `putz-workspaces` payload shape (schema v3)
  - `pty_spawn` IPC — no signature change
- **Architectural deltas**: shifts the WorkspaceStore from "metadata only" to "authoritative session-restoration source". The "PTY sessions die on restart" comment at `workspaceStore.ts:82-83` becomes obsolete and must be removed.
- **Backward compatibility**: 100% — old snapshots upgrade silently; users who had no persisted layout get the today behavior.
- **Risk surface**:
  - Introduced: corrupt persisted state crashing startup (mitigated by R10/SC3); replaying a stale cwd that no longer exists (mitigated by S8 edge case)
  - Reduced: feature gap with mainstream terminals; user friction on every restart

---

## 7. System Impact

### Affected components

| Component | Change |
|---|---|
| `src/stores/workspaceStore.ts` | Remove `savedLayout: null` clamp; add full layout restore + flush-on-close |
| `src/stores/layoutStore.ts` | Serialize/deserialize `RegionTab` with new `cwd` field; restore-time spawn |
| `src/types/index.ts` | Add `cwd?` (and P2: `command?`) to `RegionTab` |
| `src/utils/migratePersistence.ts` | Schema v3 migrator; corrupt-state fallback |
| `src/stores/settingsStore.ts` | `restoreTabsOnLaunch` flag (default `true`) |
| `src/components/Terminal/cwdRegistry.ts` | Add snapshot getter for save path |
| `src/App.tsx` | Gate auto-create-first-tab; surface restore-failure toast |
| `src-tauri/src/pty/manager.rs` | (validate) restored cwd goes through existing allowlist |

### Affected contracts

- `localStorage["putz-workspaces"]` — schema v3
- No IPC signature changes

### Architectural deltas

- WorkspaceStore becomes authoritative for cross-restart state
- A single source of truth for "what tabs exist" survives restart, eliminating the today-behavior comment at `workspaceStore.ts:82-83`

### Backward compatibility & migration

- v1 → v3 and v2 → v3 migrators in `migratePersistence.ts` must accept the existing `WorkspaceLayout` shape and add empty-cwd fields. Removed `RegionTab.type` values are stripped (existing pattern).
- Downgrade is not supported (an older build seeing v3 will fall back to a fresh layout via the existing migration error path).

### Risk surface

- **Introduced**: corrupt-state crash (mitigated R10/SC3); stale cwd open in wrong dir (mitigated by S8 fallback); minor startup latency (bounded N1)
- **Reduced**: user friction on every restart; workspace switcher today shows empty regions, which is misleading

---

## 8. Product Impact

- **Positioning**: brings Putz to parity with iTerm2 / Warp / Wave. Persisting tabs is table-stakes for "modern terminal" positioning claimed in `README.md:5`.
- **Scope**: incremental — does not require new UI surfaces beyond a Settings toggle and one toast. No marketing copy change required for T1–T3.
- **Roadmap dependencies**:
  - Composes with #144 (T5 integration tests), #119 (cwd consumer migration to OSC 7) — T2 of this spec aligns with #119 by reading from `cwdRegistry`.
  - Composes cleanly with #93 (T7 first-launch migration epic) — extends the same `migratePersistence` schema discipline.
- **User-facing communication**:
  - CHANGELOG: "Tabs and workspaces now restore across restarts. Toggle off in Settings → Restore tabs on launch."
  - First post-upgrade launch: no special prompt; behavior simply improves on the second launch.

---

## Open Questions

1. **OQ-1 [P2 / T4]** — Does Copilot CLI expose a `--resume <session-id>` flag and a stable session id while running? If yes, what's the API to discover it from outside the process?
2. **OQ-2 [P2 / T4]** — Same as OQ-1 for `claude` CLI and the `agency` wrapper.
3. **OQ-3** — When a region's previously-active tab no longer exists post-migration, fall back to first tab (proposed) or last tab? Default proposed: first.
4. **OQ-4** — Should "Restore tabs on launch" be a single global toggle or per-workspace? Proposed: global (simpler UX, matches every other terminal).
5. **OQ-5** — Save cadence: 1 s debounce + flush-on-close (proposed) vs. per-action sync. Proposed wins on perf; OQ-5 only revisits if SC1 testing shows lag.

---

## References

- `src/stores/workspaceStore.ts:80-86` — the clamp this spec removes
- `src/stores/layoutStore.ts:32-82` — `SpawnTabOptions` (already wires `cwd`/`args`/`env`/`tabId` end-to-end)
- `src/stores/tabStore.ts` — legacy in-memory tab store (NOT touched by this spec)
- `src/components/Terminal/cwdRegistry.ts` — cwd source of truth
- `src/utils/migratePersistence.ts` — existing schema v2; this spec ships v3
- `src-tauri/src/pty/manager.rs:91-230` — PTY spawn with cwd/args validation
- `src-tauri/src/ipc/terminal.rs:19-60` — `pty_spawn` IPC signature (already accepts cwd/args)
- Issue #119 — cwd consumer migration to OSC 7 (composes with T2)
- Issue #93 — first-launch migration epic (composes with T3)
