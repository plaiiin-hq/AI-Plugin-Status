# The live probe and command folder

## Overview

Every probe and command that runs has a **live** copy in the config path, and that copy is what
agents get:

```
{config-path}/probes/{probe-id}/
├── probe.yml         # definition (name, params, outputs, layout, metadata)
├── check.js          # script source
├── versions.yml      # this entry's version index — live, origin, releases, drafts
├── .versions/
│   ├── rel/<version>/    # immutable release snapshots
│   └── dev/<label>/      # drafts
└── infographic/      # optional: template.svg, template-dark.svg, bindings.yml
```

```
{config-path}/commands/{command-id}/
├── command.yml       # same parser as probe.yml — the kind comes from the directory
├── run.js
├── versions.yml
└── .versions/…       # identical layout; commands are versioned exactly like probes
```

```
{config-path}/
├── probes/ · commands/
├── libraries.yml     # the git libraries this server tracks
├── library-cache/    # their clones — server-managed, not hand-edited
├── infrastructure.yml
└── notifications.yml
```

⚠️ `.versions/` and `versions.yml` are versioning metadata, not content. They are excluded from
every snapshot copy, so a release never contains a copy of its own history.

## ⛔ There is no built-in catalog

Nothing ships in the jar. `resources/catalog/` holds no `probes/` and no `commands/` directory;
`autoInstallBuiltins()` is gone; and the two endpoints that existed to copy a built-in out of the
classpath answer **`410 Gone`**:

| Retired | Replacement |
|---|---|
| `POST /api/catalog/install/{id}` | `POST /api/libraries/{name}/{probes\|commands}/{id}/import` |
| `POST /api/catalog/update/{id}` | `POST /api/libraries/{name}/{probes\|commands}/{id}/update` |
| `POST /api/ide/toggle-dev` · `/api/ide/probe-dev` | `*-version-create` / `*-version-release` |
| "Reset to builtin" (delete the folder, restart) | Activate an earlier release, or re-import from the library |

Anything computing provenance from `builtinIds`, `availableProbes` or `availableCommands` in
`GET /api/catalog` now computes it from a **permanently empty** collection. Read `liveSource` /
`liveLibrary` on the entry instead, or `origin` from `GET /api/ide/probe-versions?id=`, and call
an entry with no origin *yours*.

## Lifecycle

### Import from a library

1. `libraries.yml` names a git repository; the server clones it into `library-cache/<name>/`.
2. `POST /api/libraries/{name}/probes/{id}/import` **copies the entry's files** into
   `probes/{id}/` — verbatim, so comments and key order survive — and records a release in
   `versions.yml` with an `origin` naming the library, path, commit and imported version.
3. A library with `installAll` re-imports anything missing after each successful refresh. An
   entry you deliberately uninstalled is on the library's `excluded` list and is not brought
   back; that is what makes uninstall stick.

### Author from scratch

`POST /api/ide/probe-create` (or `command-create`) creates the directory with a default script
and manifest. Such an entry has **no `origin`**, so a `live` write to it is allowed.

### Edit

| Target | What happens |
|---|---|
| `ref: dev/<label>` | Writes into `.versions/dev/<label>/`. No agent is affected |
| `ref: live` on an entry with no library origin | Writes the live files, marks the entry `liveDirty`, reloads. Agents pick it up on the next heartbeat |
| `ref: live` on a library-sourced entry | **`409`** — `<id> runs <library> <version> — edits go into a dev version` |
| `ref: rel/<version>` | **`409`** — releases are immutable |

So the ordinary loop is `probe-version-create` → edit at `dev/<label>` →
`probe-version-release` (with `activate: true` when it should go live) — and rollback is
`probe-version-activate` with an older version, the same verb.

### Update

A library offering a newer version shows up in `GET /api/catalog` under `updates[]` for probes
and **`libraryCommandUpdates[]`** for commands — deliberately separate keys, because every
button in the probe Updates list posts to the *probe* update endpoint.

Each row carries what you need to decide rather than just a version bump:

| Field | |
|---|---|
| `offer` | `update` = import the library version and activate it · `activate` = it is already recorded as a release, only activation is left |
| `requiresChoice` | **`true` when what is live is a local release**, i.e. you edited it. Taking the library version replaces yours; your release stays in `.versions/rel/` and is one `activate` away |
| `fromVersion` / `toVersion`, `fromCommit` / `toCommit` | What moves |
| `changelog` / `newChanges` | The library's changelog entries the installed copy does not carry |

`autoApplyProbeUpdates` and `autoApplyCommandUpdates` apply these without asking. Both are off by
default, independent of each other, and API-only — the UI shows neither.

### Uninstall

`GET /api/catalog/uninstall/{id}/impact` first — it names the releases, drafts and wiring that
would go. `POST /api/catalog/uninstall/{id}` is `STATUS_ADMIN` only and is refused `409` while
the entry is wired in `infrastructure.yml` (`wiredChecks`) or, for a command, while a saved
preset or a queued dispatch names it.

⚠️ **The catalog key is `(kind, id)`.** A probe and a command may share an id. Both endpoints
above take `?kind=probe|command`; with both kinds installed and no `kind`, nothing is deleted and
the `409` names them.

## How an agent actually gets a script

**Not through `/api/catalog/sync`.** An agent receives `scriptSource`, `actionScripts` and
`shell` **inline in its heartbeat's probe assignments**; a command's `run.js` is attached at
dispatch time. `/api/catalog/sync` still exists but no agent calls it, and it is now
`STATUS_ADMIN`/`INFRA_ADMIN` — it used to be on the public chain, serving every installed
probe's source to anyone.

Timing: a file change is detected by a `WatchService` (500 ms debounce) plus a 30 s
modification-time poll, then delivered on the next heartbeat (default 30 s) — **up to roughly
60 s**, not the 3 seconds older notes claim.

## Probe IDE endpoints

They live under `/api/ide/**`, not `/admin/scripts/api/**` — those old JSON paths are gone, and
an `X-API-Key` on one `302`s to the login form rather than failing cleanly.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/ide/probes` · `/api/ide/commands` | List what is installed |
| GET | `/api/ide/probe-source?id={id}&ref=` | `check.js` (`?name=` also accepted) |
| GET | `/api/ide/probe-definition?id={id}&ref=` | `probe.yml` |
| POST | `/api/ide/probe-save` · `/api/ide/probe-definition` | Write, at `ref` |
| POST | `/api/ide/probe-create` | New probe from scratch |
| GET | `/api/ide/probe-versions?id={id}` | `live`, `liveDirty`, `origin`, `releases[]`, `dev[]` |
| POST | `/api/ide/probe-version-create` · `-delete` | Drafts |
| POST | `/api/ide/probe-version-release` · `-activate` | `STATUS_ADMIN` only |
| POST | `/api/ide/test` | Run server-side against sample params |
| POST | `/api/ide/test-on-agent` → GET `/api/ide/test-on-agent/{id}` | Run on a real agent; `{agent, id, ref}` or `{agent, source}` |

Swap `probe-` for `command-` for the command side of every row.

There is still no Probe IDE endpoint for `action-*.js`, `detect.js` or `icon.svg`. Put those in
the library (or on the filesystem under the entry's directory); the loader rescans the directory
on every reload.
