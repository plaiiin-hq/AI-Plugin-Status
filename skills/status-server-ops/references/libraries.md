# Probe and command libraries

**This is where probes and commands come from.** Nothing ships in the server jar: no built-in
probes, no built-in commands. Both kinds are imported from a **library** — a git repository the
server clones, verifies and offers entries from.

Read this before importing, editing, updating or uninstalling anything in the catalog.

## What a library is

```yaml
# library.yml, at the repository root
name: ops-toolkit
description: Checks and the commands that fix what they find
apiVersion: 1          # only 1 is accepted; anything else and the library loads as INVALID
probes: probes/        # optional, this is the default
commands: commands/    # optional, this is the default
modules: modules/      # optional, this is the default
```

```
<repo>/
├── library.yml
├── probes/<id>/{probe.yml,check.js,…}
├── commands/<id>/{command.yml,run.js,…}
└── modules/                     # shared by BOTH kinds
```

| Rule | |
|---|---|
| A library may publish **both kinds** | And the same id may appear in each — `disk-space` as a check *and* as a command is legitimate |
| `probes:` and `commands:` pointing at the **same** directory | Offers no commands, and says so in `warnings[]` rather than guessing |
| A stray file beside `run.js` (`command.yml.bak`, `.DS_Store`, a fixture) | **Ignored, and named in `warnings[]`** — it reaches no release and no live directory. A library is not ship-blocked by an editor artefact |
| Every path must stay inside the repo | A `probes: ../elsewhere` is refused at verify |

### Shared modules

`modules/` is flattened into each script by the bundler, because the agent sandbox has no module
loader. It is shared by both kinds, which makes the reserved-name rule matter:

⚠️ **A module may not declare `check`, `run` or `action`** at the top level, nor any
ECMAScript keyword, `ctx`, `__ctx`, `arguments` or `eval`. It is refused at bundle time — and
that refusal is load-bearing, not pedantry: the server rewrites `function run(`/`function
action(` to `function check(` **textually, every occurrence**, before a command's bundle reaches
the sandbox. A module declaring one of those would have its declaration renamed while its own
call sites were not, which is a `ReferenceError` on a production host and green in every
server-side test.

## 🚨 The security model, stated plainly

> A probe or command published by a library **runs arbitrary shell on every monitored host**,
> unless that host's agent runs with `AGENT_READONLY`. `ctx.shell` is the same object for both
> kinds, behind the same gate.

So **the perimeter is write access to the library repository**, not the API. Protect its
branches. Adding a library is `STATUS_ADMIN` for exactly this reason: it points your fleet at
someone else's repository.

Libraries do not widen what a probe *can* do — they widen *who can supply the code*.

## Endpoints

Reads need `STATUS_ADMIN` or `INFRA_ADMIN`; every write is **`STATUS_ADMIN`**.

| | |
|---|---|
| `GET /api/libraries` | What is configured, with `lastCommit`, `probeCount`, `commandCount`, `enabled`, `autoUpdate`, `installAll`, `credential`, `lastError`, `warnings[]`, `refreshing` |
| `GET /api/libraries/{name}/probes` · `/commands` | What it offers, each row with `version`, `valid`, `warnings[]`, `collision`, `installed`, `liveVersion`, `liveSource`, `updateAvailable`, `state` |
| `GET /api/libraries/{name}/probes/{id}` · `/commands/{id}` | One entry with its files |
| `GET /api/libraries/{name}/impact` | What turning it off or removing it would strip |
| `POST /api/libraries` | `{name, source, ref?, credential?, autoUpdate?, autoApplyProbeUpdates?, autoApplyCommandUpdates?, installAll?}` |
| `PATCH /api/libraries/{name}` | Any of `{enabled, autoUpdate, autoApplyProbeUpdates, autoApplyCommandUpdates, ref, credential}` |
| `DELETE /api/libraries/{name}` | Removes it — and strips its entries' presets and probe wiring in the **same recorded `infrastructure.yml` change** |
| `POST /api/libraries/{name}/refresh` | Fetch now |
| `POST /api/libraries/{name}/probes/{id}/import` | **Replaces `POST /api/catalog/install/{id}`**, which is `410 Gone` |
| `POST /api/libraries/{name}/commands/{id}/import` | The kind is in the path — never post a command id to the probe endpoint |
| `POST /api/libraries/{name}/probes/{id}/update` · `/commands/{id}/update` | **Replaces `POST /api/catalog/update/{id}`**, also `410 Gone` |

A private library needs a credential from the credentials store; name it in `credential`. See
`references/credentials-store.md`.

⚠️ **Pinning forces auto-update off.** A library with a `ref` set always has
`autoUpdate: false`, and sending `ref` together with `autoUpdate: true` is **refused** rather
than silently dropped — so nobody gets one without noticing.

## Import, update, uninstall

### Import

Copies the entry's files into `{config-path}/probes/<id>/` **verbatim** — comments, key order and
`category:` all survive, unlike the old built-in install — and records a release plus an `origin`
in the entry's `versions.yml`.

A library with `installAll: true` re-imports anything missing after each successful refresh. That
is why a deploy that wipes the probe folder heals itself, and why uninstall has to record an
exclusion rather than just delete a directory.

### Update

`GET /api/catalog` reports what is available:

| Key | Holds |
|---|---|
| `updates[]` | Probe updates |
| `libraryCommandUpdates[]` | **Command** updates — a separate key on purpose, because every button in the probe Updates list posts to the *probe* endpoint, so a command row there would send a command id to it |

Each row: `offer` (`update` = import and activate · `activate` = the release is already recorded
and only activation is left), `fromVersion`/`toVersion`, `fromCommit`/`toCommit`, the changelog
entries the live copy does not carry, and **`requiresChoice: true` when what is live is a local
release** — i.e. you edited it, and taking the library's version replaces yours. Your release
stays in `.versions/rel/` and is one `activate` away.

`autoApplyProbeUpdates` and `autoApplyCommandUpdates` skip the asking. Both default to off, are
**independent** of each other — the probe flag never carries a command with it — and are API-only:
the UI shows neither.

### Turning a library off — dormancy

A disabled library leaves its entries **installed but dormant**: they appear under
`dormantProbes` / `dormantCommands` in `GET /api/catalog`. A dormant command is refused **at
dispatch**, with the reason stated (`refused: <id> is not runnable: library <name> is off`),
rather than dispatching a script whose source is gone. The check is at both the queue and the
dispatcher, because a command can be queued while the library is on and reach the dispatcher
after it was turned off.

### Uninstall

`GET /api/catalog/uninstall/{id}/impact` first — it names the releases, drafts and whatever is
using the entry. Then `POST /api/catalog/uninstall/{id}`, `STATUS_ADMIN` only, because it takes
the whole version history with it. It is refused `409` when:

| Kind | In use means |
|---|---|
| Probe | Still wired in `infrastructure.yml` — the refusal lists `wiredChecks` |
| Command | A **saved preset** names it (`agentCommands.commands` or `agentCommandOverrides[<agent>]`), or a dispatch is queued or in flight |

⚠️ The runtime Commands menu is deliberately **not** a use. It offers every command to every
agent, so counting it would make every command undeletable.

A library-sourced entry is added to the library's `excluded` list on uninstall, so `installAll`
does not bring it straight back. In `libraries.yml` an excluded probe is a **bare id** and an
excluded command is **`command:<id>`**.

## ⚠️ The catalog key is `(kind, id)`

A probe and a command may share an id, so an endpoint taking a bare id has to be told which:

| | |
|---|---|
| Explicit `?kind=probe` / `?kind=command` | Wins |
| Exactly one kind installed | That kind is used |
| **Both installed, no `kind`** | **Nothing happens** — `409`, with a `kinds` array naming both |

This applies to `/api/catalog/uninstall/{id}` and its `/impact`. Everywhere else the kind is in
the path or the endpoint name (`probe-versions` vs `command-versions`,
`/libraries/{n}/probes/…` vs `/commands/…`), which is why those cannot get it wrong.

## Provenance — do not read `builtinIds`

⛔ **`builtinIds`, `availableProbes` and `availableCommands` in `GET /api/catalog` are
permanently empty.** They are kept only so an older client still decodes the response. Anything
that decided "Built-In" from `builtinIds` now shows nothing at all, with no error.

Read provenance from the entry instead:

| Source | Field |
|---|---|
| `GET /api/catalog` → an installed entry | `liveSource` (`library` / `local`) and `liveLibrary` |
| `GET /api/ide/probe-versions?id=` | `origin` — `{kind:"library", library, path, ref, commit, importedVersion}`, or `null` |
| A release inside `releases[]` | `source` and `library` |

**An entry with no origin is yours.** Label it that way — "Yours", not "Built-In".

## See also

- `references/probe-active-folder.md` — the live folder, `versions.yml`, `.versions/`, and how a
  script actually reaches an agent (not through `/api/catalog/sync`).
- `status-server-api` → *Probe authoring over the API* — the `ref` rules, the two `409`s, and the
  create → edit → release → activate loop.
