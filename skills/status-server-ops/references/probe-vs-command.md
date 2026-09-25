# Probe vs Command Output Comparison

| | Probe | Command |
|---|---|---|
| **State** | OK / WARNING / ERROR | OK / ERROR (completed/failed) |
| **Stream values** | Yes — per-path metrics every tick (state, number, percent, bytes, log, label) | No |
| **Structured result** | `data` + `schema` JSON | `data` + `schema` JSON (same format) |
| **Result file** | `results/probes/{name}.json` | `results/commands/{id}.json` |
| **DB storage** | ProbeHistoryStore (SQLite per probe) | `agent_commands.result` column |
| **API** | `GET /api/probes/result?probe={name}` | `GET /api/commands/result?command={id}` |
| **Lifecycle** | Recurring (interval) | One-shot (triggered) |
| **Execution** | `ProbeSandbox` (GraalJS) on the agent | **The same `ProbeSandbox`** — the server rewrites `function run(`/`function action(` to `function check(` textually before the bundle is sent. (seven ids are answered by a built-in Java executor instead — `restart` `stop` `start` `logs` `script` `probe-action` `test-probe` — so a catalog command may not be named any of them; `command-create` and `command-save` refuse and name the conflict) |
| **`ctx.shell`** | gated by `AGENT_READONLY` | **the same gate, the same object** — a command is not more privileged than a probe |
| **Streaming output** | No | Yes — line-by-line via command-stream |
| **Comes from** | a git library | **a git library too** — since the jar stopped shipping the 25 built-ins |
| **Versioned** | `versions.yml`, releases, drafts, activate, rollback | **identically** — `dev: true` retired for both, and `POST /api/ide/toggle-dev` answers `410 Gone` |
| **Authoring endpoints** | `/api/ide/probe-*` | `/api/ide/command-*`, one-for-one, including `command-versions`, `command-version-create`, `-release`, `-activate`, `-delete` |
| **Uninstall refused while** | wired in `infrastructure.yml` | a **saved preset** or a **queued dispatch** names it (the runtime Commands menu deliberately does not count) |

The `data` + `schema` format is identical between probes and commands — same `CommandResult`-style structure with typed field annotations for UI rendering.

⚠️ **The two are no longer asymmetric in anything but lifecycle and streaming.** The old story —
commands ship in the jar, are unversioned, and are live-edited straight into the path every agent
reads — is gone. If you are deciding between them, decide on *recurring vs triggered*, not on
which one has tooling.

⚠️ `dangerous:` and `confirm:` are read from the manifest now (they were dead code, hardcoded
`false` for every command), so a command that declares `dangerous: true` reaches the UI's
confirm path.

## Command Instances

Commands additionally support **instances** — operator-defined presets that wrap
a catalog command with a custom name and preset parameters. An instance can mark
some params as `frozen`, meaning the operator can't override them at run time
(useful for locking a specific path, container, or script). Instances live in
`infrastructure.yml` under `agentCommands.commands` (defaults) and
`agentCommandOverrides[agentName]` (per-agent extras). See
[infrastructure-model.md](../../docs/infrastructure-model.md#agent-policies).
