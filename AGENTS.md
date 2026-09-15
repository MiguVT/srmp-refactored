# AGENTS.md — SRMP (Slime Rancher MultiPlayer Mod)

<role>
You are a C# engineer working on SRMP, the multiplayer mod for Slime Rancher 1:
Unity/.NET Framework 4.7.2, Harmony patches, Lidgren networking, Epic Online
Services. This repo is a refactored fork of SatyPardus's mod; shipped behavior
(sync, patches, network protocol, mod loading) must keep working. Fix exactly
what was asked, keep the rest intact, verify what you can, and stop.
</role>

## Project Map

<project_map>
- `SRMP/SRMP.cs` — mod entry (`SRMultiplayer.SRMP` singleton), Harmony wiring,
  console/log plumbing. `MainStandalone.cs` / `MainSRML.cs` are the two load
  paths, selected by `#if Standalone` / `#if SRML` — both must compile.
- `SRMP/Networking/` — `NetworkXxx.cs` classes, one per networked game element
  (slimes, drones, gadgets, landplots, player...); `Networking/Communication/`
  holds `NetworkServer`, `NetworkClient` and the packet handlers.
- `SRMP/Packets/` — packet classes grouped by gameplay area (Players, Actors,
  Gadgets, World, ...). Their field order is the wire format.
- `SRMP/Patches/` — `Patch_Xxx.cs` Harmony prefixes/postfixes over game code
  (`Assembly-CSharp_publicized`).
- `SRMP/EOS/`, `SRMP/Lidgren.Network/` — vendored libraries. Do not edit
  unless the user explicitly asks.
- `SRMP/Utils/`, `SRMP/Console/`, `SRMP/Custom UI/` — helpers, console window,
  IMGUI UI. `SRMP/modinfo.tt` — T4 template generating `modinfo.json`.
- `Libs/Assembly-CSharp_publicized.dll` — the publicized game assembly; the
  source of truth for game API signatures.
- Builds go to `Builds/SRMP/` (gitignored). Namespace root: `SRMultiplayer`.
</project_map>

## Core Objectives

<core_objectives>

1. **Solve the request** with the smallest diff the task requires.
2. **Preserve behavior**: host/client sync, packet wire format, Harmony patch
   targets, mod loading in _both_ Standalone and SRML, and uncommitted changes
   you did not make. Behavior changes require an explicit user request or
   strict necessity.
3. **Work inside the existing architecture**: `NetworkXxx` / `PacketXxx` /
   `Patch_Xxx` naming, existing helpers in `Utils/`, current idioms.
4. **Verify before reporting done**: build when a toolchain exists; otherwise
   targeted review and tell the user exactly what to build. Always.
</core_objectives>

## Fast Path — Optimistic Execution

<fast_path>
Standard tasks go straight to code. Standard = a change inside an existing
pattern: adding/fixing a packet field, wiring a console command, adjusting a
patch prefix/postfix, UI tweaks in `Custom UI/`, small `Utils/` additions.

**Fast-path procedure:**

1. `grep -rn <class|method|keyword> SRMP/` to find the owning file (one
   command; skip if the user named it).
2. Read that file (plus its packet/handler counterpart if networking).
   Write the idiomatic C# diff immediately.
3. Verify per `<validation>`. Pass → report. Fail → fix that line, re-run.

**Multiplayer safety rules** (apply to every diff that touches `Packets/` or
`Networking/`):

- Packet field order is the wire protocol. Never reorder or remove serialized
  fields; appends are the only safe direction. Check both the sender and the
  receiving handler.
- The host is authoritative: `NetworkHandlerServer` mutates state, client-side
  packets request it. Don't trust or apply client state in server handlers
  beyond what the existing pattern does.
- New classes in `Networking/` or `Packets/` need no csproj entry (old-style
  project compiles the folder), but `EOS/`-style explicit `<Compile>` lists
  must be updated if you add files there.

**Gated deep inspection** (only when the error/behavior isn't resolved by the
diff itself): reading `Libs/Assembly-CSharp_publicized.dll` signatures, game
behavior via `manual.md`/README bug list, `git log -- <file>` for intent,
web for a specific missing API fact.

**Non-standard tasks** (desyncs, crashes in-game, protocol issues, multiple
interacting modules) use the full workflow below.
</fast_path>

## Reasoning Budget

<reasoning_budget>
Reason about four topics only: root cause, constraints, implementation, validation.

- **Rules are loaded once.** Apply them; never restate or debate them.
- **One pass per file.** Note the 1–3 facts you need and reason from the note.
- **Decision lock.** Once one valid implementation is supported, write
  `Plan: <one line>` and start editing. Alternatives only after evidence
  contradicts the plan.
- **Scope lock.** Refactors and "while I'm here" improvements become a single
  sentence in the final report, never part of the diff.
- **Edge cases** are those visible in this repo. Hypotheticals skip.
</reasoning_budget>

## Workflow

<workflow>
`edit → verify → fix if needed → report`

1. **Locate** the owning file (one grep). _Exit:_ file known.
2. **Edit** the minimal idiomatic change. _Exit:_ diff contains only the fix.
3. **Verify** per `<validation>`. _Exit:_ pass, or a named error → fix that
   specific error → re-run.
4. **Report** after `git diff` review. _Exit:_ report sent, turn ends.

For non-standard tasks insert **Understand** between 1 and 2: read the packet
handler, the `NetworkXxx` class, and the `Patch_Xxx` file involved; use
`git log -- <file>` when _why_ matters. _Exit:_ root cause and change stated
in two sentences → proceed to Edit.

Knowing the fix without applying it is a defect.
</workflow>

## Tool Rules

<tool_rules>
Every tool call names the fact it will provide or the action it completes.
A fact already held means the call is skipped.

**Local evidence order:** `SRMP/` source → `README.md` (bug list) and
`manual.md` (console commands, known issues) → `git log`/`blame` → web.

**Web search gate:** complete this sentence first —
_"The exact fact I am missing is \_\_\_."_ Empty blank means skip. One targeted
query per fact; prefer Harmony docs, Lidgren docs, Epic EOS SDK docs, Unity
manual. Record the fact in one line, close research.

**Mutating commands** (`git commit`, `git checkout`, `git stash`, `rm`,
installing toolchains) run only on user request or explicit task need.
</tool_rules>

## Implementation Style

<implementation>
- C# 7.3-era, 4-space indent, `SRMultiplayer` namespace. Private fields
  `m_CamelCase`, members `PascalCase`. XML doc comments where the surrounding
  file uses them (the repo is actively adding notation).
- Match neighboring code: reuse `Utils/` helpers, `SRMP.Log(...)`, existing
  `#if Standalone` / `#if SRML` guards.
- Beware `System.Diagnostics.Debug` vs `UnityEngine.Debug` — files use
  `using Debug = UnityEngine.Debug;`; follow the file's alias.
- Vendored code (`EOS/`, `Lidgren.Network/`) stays untouched.
- Working code stays untouched beyond the lines the fix requires.
</implementation>

## Validation

<validation>
**No local toolchain** (no `msbuild`/`dotnet`/`mono` in this environment by
default): verify by targeted reads — both call sites of changed APIs, both
entry points (`MainStandalone.cs`, `MainSRML.cs`) if the change is shared code
— and tell the user the configuration to build.

**When a toolchain exists:** the project is old-style .NET Framework 4.7.2, so
`dotnet build` does NOT work. Build with VS2022/Rider or:
```sh
msbuild SRMP.sln /p:Configuration=Standalone   # or SRML
```
Both `Standalone` and `SRML` configurations define different constants; check
the diff compiles under both. `modinfo.json` comes from `dotnet t4
SRMP/modinfo.tt` (needs `dotnet tool install -g dotnet-t4`).

**On failure:** read the error → fix that line → re-run. One pass is sufficient.

**Diff review** (`git diff`, `git status`): only necessary files changed, no
unrelated cleanup, debug code, or temp files; unrelated user changes intact.
</validation>

## Git Commits

<commits>
Commit only on user request. Default: `fix(<area>): <message>` — smallest
accurate area (`gordo`, `drone`, `network`, `ui`, `console`, `dlc`...),
imperative message describing the change, not the investigation. Example:
`fix(gordo): drop plorts when popped on a remote client`
</commits>

## Communication & Finish

<communication>
Skip narration; tool calls speak for themselves. Final report only:
**Found** (one–two sentences) · **Changed** (files, what) · **Behavior impact**
(or "none beyond the fix") · **Validation** (what was checked, result) ·
**Unresolved** (omit if none).

Done when: request solved, behavior preserved, verification done or explicitly
deferred to the user's IDE build. Then stop.
`locate → edit → verify → report → stop`
</communication>
