---
name: status-server-api
description: Use when reading or driving a running Plaiiin Status server from Claude — checking what is currently red, reading the probe tree or a probe's history, opening/transitioning/commenting on incidents (which are workflow records), or authoring probes, commands and their dashboard layouts (widgets, tiles) over the REST API. Covers X-API-Key auth, the /api/** boundary that makes wrong paths look like a login redirect, the role gate on probe authoring, and the version store that makes a plain save refuse. Also covers the credentials store — how a probe authenticates to what it monitors. Your own API key lives in ~/.plaiiin/status-server/env; read that before asking anyone for one.
---

# Driving a live Status board

This skill is for talking to a **running** Status server. For modelling infrastructure and
authoring probe definitions in config, see `status-server-ops`.

## API access — set once, never asked again

Both skills read `~/.plaiiin/status-server/env`. Put your server and key there and no session
needs to ask you for them. (This is *your* access to the API — not to be confused with the
**credentials store**, which holds the secrets probes use to reach the things they monitor.)

```bash
mkdir -p ~/.plaiiin/status-server && chmod 700 ~/.plaiiin/status-server
cat > ~/.plaiiin/status-server/env <<'EOF'
STATUS_URL=https://status.example.com
STATUS_API_KEY=twk_…
EOF
chmod 600 ~/.plaiiin/status-server/env
```

Load it in any shell:

```bash
set -a && . ~/.plaiiin/status-server/env && set +a
```

Environment variables of the same name win if already set, so a one-off override still works.
If the file is absent **and** the variables are unset, that is the only time you should be
asked for a key.

### Getting a key

| Route | How |
|---|---|
| **Admin UI** | Settings → **API keys** → *New key*. The raw key is shown **once** — copy it straight into the file above. |
| **API** (needs an existing key) | `POST /api/user/api-keys` with `{"name":"…"}` |

List or revoke yours with `GET /api/user/api-keys` and
`POST /api/user/api-keys/{id}/revoke`.

**A key inherits the roles of the user who minted it** — there is no per-key scoping. So the
account you are signed in as when you click *New key* decides what the key can do:

| Signed in as | Key can |
|---|---|
| an account with **no** Status role | `GET /api/user/profile` and nothing else — every other `/api/**` path answers `403` |
| an ordinary user (any one role) | read state, read history, work with workflow records |
| `STATUS_ADMIN` / `INFRA_ADMIN` | all of the above **plus** `/api/ide/**` — writing probe and command scripts that execute on every monitored host, and writing config |
| `STATUS_ADMIN` alone | the tier above that: releasing and activating a version, uninstalling, and adding or removing a library |

⚠️ **A `403` on a read as plain as `/api/tree` means the account has no Status role yet** — not
that the endpoint is admin-only. Anonymous still gets `401`; that distinction is the whole
difference between "log in" and "ask an admin for a role".

Prefer a key minted by a non-admin account for anything ambient (dashboards, bots, a
long-running assistant session). Reach for an admin key only while authoring, and revoke it
after.


### Scoping a key to less than yourself

A key inherits its owner's roles by default. Pass `roles` to narrow it:

```bash
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d '{"name":"dashboard","roles":["VIEWER","HISTORY_USER"]}' \
  "$STATUS_URL/api/user/api-keys"
```

Roles: `VIEWER` `HISTORY_USER` `HISTORY_CONFIG` `INCIDENT_RESPONDER` `INCIDENT_MANAGER`
`DRILL_RESPONDER` `DRILL_MANAGER` `PROBE_EDITOR` `WORK_USER` `STATUS_ADMIN` `INFRA_ADMIN`.

`WORK_USER` gates the work lane and the todos probes report, and it **hides** rather than
disables: a key without it does not see the lane at all. If a board's work column looks
empty or missing, check this role before checking the probes.

| Property | Behaviour |
|---|---|
| Omit `roles` | The key inherits everything you have — the historical default |
| Ask for a role you lack | Refused with `roles_not_held` and the list of what you do have |
| Owner loses a role later | Every key they minted loses it too, at the next request |
| A scoped key mints a key | Bounded by **its own** scope, not its owner's — it cannot climb out |

The intersection is computed per request rather than frozen at mint time, which is what makes
the last two rows true.

**Use this for anything ambient.** A key in a config file, a dashboard, a bot or a
long-running assistant session should be `VIEWER` + `HISTORY_USER`, not an admin key — an
admin key can `POST /api/ide/probe-save`, which executes JavaScript on every monitored host.
Keep an admin key for authoring sessions and revoke it after.

⚠️ Servers built before 2026-08-27 have no `roles` column; a `roles` list is ignored there and
the key inherits everything. Check with `GET /api/user/api-keys` — scoped keys report their
`roles`.

⚠️ **If a key returns `401 {"error":"User not found"}`**, the key itself is fine — the account
that owns it no longer resolves in the identity directory. That happens when the directory is
re-imported and user ids change. Mint a fresh key as a current user; revoking and re-issuing
against the old account will not help.

### ⚠️ The `/api/**` boundary — the confusing failure

`ApiKeyAuthFilter` is registered **only** on `securityMatcher("/api/**")`. A key presented
on any other path is not merely rejected — the request falls through to the browser
security chain and you get a **302 to the login form**, which reads like a broken key or a
down server. It is neither: it is the wrong path.

If a call redirects to `/app/login`, check the path before you check the key.

### 🚨 A `302` does not always mean auth

`302 → /app/login` has **three** causes, and only one is about your key:

| Cause | Tell |
|---|---|
| Path is outside `/api/**` | Check the path first |
| Key invalid / owner unresolvable | Body is `{"error":"Invalid API key"}` or `{"error":"User not found"}` — a 401, not a 302 |
| **A required query parameter is missing** | The request never reaches the handler; the error dispatch redirects |

**Fixed on builds from 2026-08-27.** A missing parameter now returns a proper
`400 {"status":400,"error":"Bad Request","path":"…"}`.

On an older build it redirects instead: `GET /api/workflows/attachable` lands on the login
page, while the same call with `?kind=probe` returns `200`. Nothing in the response says "you
forgot a parameter" — it looks exactly like being logged out. **If you get a 302 on a server
that old, check the endpoint's required parameters before suspecting your credentials.**


**A 302 can also mean a wrong query-param name**, not a wrong path. Spring's missing-parameter
error is an ERROR dispatch that re-enters the filter chain, so on a chain that does not permit
that dispatch it comes back as a redirect to the login form — indistinguishable from a bad key,
and it sends you to check credentials that were never the problem. **On any 302, check the
endpoint's parameter names before suspecting your key.**

`/api/ide/probe-source` was the worst case, taking `?name=<probe-id>` while its neighbour
`/api/ide/probe-definition` takes `?id=`. It now accepts either spelling and answers a missing
one with a 400 that names both; on a server older than 2026-08-27, `?id=` 302s there.

Every JSON endpoint lives under `/api/**` — that invariant holds codebase-wide as of
2026-08-27. On older builds a few served JSON from outside it (notably the infrastructure
config, at `/admin/infrastructure/api/config`) and were unreachable with a key.

## Reference files

| File | Covers |
|---|---|
| `references/api-surface.md` | **Every endpoint** (172), grouped by area, generated from the server source, with 🔒 marking the role-gated ones. Start here to find something. |
| `references/api-endpoints.md` | Hand-written detail on the main endpoints — params and response shapes. |
| `references/endpoints.md` | The wider surface, including admin and agent routes. |

The tables below are the working subset; the references are authoritative.

## Start here: `GET /api/capabilities`

One request returns every vocabulary you need to write a valid config, read from **this**
server rather than from documentation that may not match it:

```bash
curl -s -H "$K" "$STATUS_URL/api/capabilities"
```

| Key | Contains |
|---|---|
| `probeTypes` · `probeStates` · `dataTypes` | `HTTP_HEALTH`…`MAPPED_JSON`; `OK`/`WARNING`/`ERROR`/`UNKNOWN`; history types |
| `paramTypes` · `outputTypes` | Read from the installed catalog, so they describe what this server actually has |
| `widgets` | `card` · `plate` · **`both`** · `cardOnly` · `plateOnly` — see the note below |
| `tileSizes` · `panelWidthUnits` | Canonical spans; the panel is 4 units wide |
| `serviceTypes` | Which `type:` values exist — **check here first** if a `type:` seems to do nothing |
| `probeCatalog` | Every installed probe id |

⚠️ **Use `widgets.both`** for tiles on the probe card. A widget the card cannot render
produces an empty cell, silently — the response says so too.

If this endpoint 404s you are on a build from before 2026-08-27.

## Reading state

| Endpoint | Use |
|---|---|
| `GET /api/status/summary` | **Start here** for "is anything wrong": `errors` `warnings` `unknown` `muted`, active mutes, work, `untrackedIssues`, and a flat `probeDetails` list (name, state, uptime, last message). About 20 KB. Builds from 2026-10-08 on; older ones answer 404, use `/api/status` there. |
| `GET /api/status` | The whole board as the web app draws it: both trees with every probe's sparklines. **2.8 MB** on a 75-probe board, every probe twice — too big to read whole; use the summary unless you need a node's `result`. |
| `GET /api/tree` | The **full** probe tree. The authoritative view: use it to confirm a `ref` actually resolved and a probe actually ran. |
| `GET /api/global` | Tab list / global SPA state. |
| `GET /api/events` | Recent state transitions. |
| `GET /api/probes/history?probe=<name>&resolution=5s` | Time series for one probe. Resolutions step up (`5s`, `1m`, …) — ask for the coarsest that answers the question. |
| `GET /api/probes/history/list` | Which probes have history at all. **A probe with no history has never run** — that is the trap-1 signature from `status-server-ops`. |
| `GET /api/probes/snapshot?probe=<name>` | Current value of every path for one probe. |
| `GET /api/untracked-issues` | Things failing that no workflow record covers yet — the natural triage queue. |
| `GET /api/presence` | Who is online. |
| `GET /api/agents` | The agents, with their heartbeat data. |
| `GET /api/infrastructure/config` | The whole declared infrastructure — hosts, projects, dependencies, thresholds. |
| `GET /api/infrastructure/types` · `GET /api/infrastructure/hosts` | Service-type catalog with param metadata; host names. |
| `GET /api/capabilities` | Vocabularies, incl. `probeCatalog` — every installed probe id. |

⚠️ **`/api/hosts`, `/api/types` and `/api/active` do not exist** (404). Earlier revisions of
this file listed them. Use `/api/infrastructure/hosts`, `/api/infrastructure/types` and
`/api/tree` respectively. Anything taking a `probe` parameter answers `400` without it, not an
empty result.

Probe names in the tree are **whitespace-sensitive path strings**:
`Agents / app-01.example.com / Web Reachable`. Copy them from `/api/tree` rather than
retyping — a near-miss returns empty, not an error.

## Incidents — a workflow type, not an endpoint of their own

⛔ **There is no `/api/incidents`.** It answers `404`. The workflow engine replaced the legacy
incident model: an incident is a *record of type `incident`* moving through a declared state
machine, and the same endpoints serve every other record type a deployment declares.

**Read the type before you write an instance.** `GET /api/workflows/types/incident` returns the
nodes, the edges between them and the fields each node shows. On a stock deployment that is
`open` `investigating` `identified` `monitoring` `mitigating` `resolved`, with `* → resolved`
reachable from anywhere; the fields are `title` `severity` `assignee` `eta` `summary`
`services` `postmortem`. **Read it rather than trusting that list** — a deployment can edit its
own types, and a `to` that is not a node is refused.

```bash
K="X-API-Key: $STATUS_API_KEY"

# what types exist, and what this one's state machine looks like
curl -s -H "$K" "$STATUS_URL/api/workflows/types"
curl -s -H "$K" "$STATUS_URL/api/workflows/types/incident"

# list
curl -s -H "$K" "$STATUS_URL/api/workflows/incident"

# open — fields are whatever the type declares
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d '{"fields":{"title":"Checkout latency elevated","severity":"minor"}}' \
  "$STATUS_URL/api/workflows/incident"

# comment — the field is `text`
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d '{"text":"Traced to the payments upstream."}' \
  "$STATUS_URL/api/workflows/incident/<id>/comments"

# move it along — `to` is a node id from the type, `version` is the instance's current version
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d '{"to":"resolved","version":3,"fields":{"summary":"Upstream recovered; p99 under 400ms."}}' \
  "$STATUS_URL/api/workflows/incident/<id>/transitions"

# attach the probe that is red, so the record and the board point at each other
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d '{"kind":"probe","id":"<probe-id>"}' \
  "$STATUS_URL/api/workflows/incident/<id>/references"
```

A reference is only accepted by a type that declares it accepts that kind — ask which do with
`GET /api/workflows/attachable?kind=probe` (the `kind` parameter is **required**; omitting it is
a `400`).

| Trap | |
|---|---|
| `name` is an **i18n map** | `{"en":"Incident","de":"Vorfall"}`. Rendering it directly prints an object. True of type and field labels throughout |
| `version` is optimistic locking | Send the instance's current `version` on a transition or a field patch. Omit it and you send `0` |
| An edge's `when` is a **hard gate** | A transition whose condition is false is refused with `when_condition_false`, not let through with a warning. `override: true` is opt-in and separate |
| Comment field | `text`. `content` or `comment` posts an empty comment and still answers `204` |

Prefer opening a record over letting a red sit unexplained — an unexplained red is how people
learn to ignore red. Full endpoint list: `references/api-surface.md` → *Workflows*.

## Writing configuration

`POST /api/infrastructure/config` with the full config object. It saves, reloads the scheduler
and records a history entry in one call — `200 {"status":"ok"}`, or `409`/`400` with
`{"error": {"code", "message"}}`. Requires `STATUS_ADMIN` or `INFRA_ADMIN`.

Add `?dryRun=true` to get the same report **without writing anything** — refs resolved and
missed, probes bound to agents that do not exist, how many probes the config would generate.
Do that before every change.

⚠️ A real save does not refuse a broken config; it applies it and reports the problems
alongside `status: "ok_with_warnings"`. Read the response body, not just the status code.
Details and the lossy round-trip caveat: `status-server-ops`.

## Probe authoring over the API

The Probe IDE's backend is fully scriptable under `/api/ide/*`:

| Endpoint | Use |
|---|---|
| `GET /api/ide/probes` · `GET /api/ide/list` | What is installed. |
| `GET /api/ide/probe-source` · `GET /api/ide/probe-definition` | Read a probe's `check.js` / `probe.yml`. Both accept `?name=` or `?id=`, plus `?ref=`. |
| `POST /api/ide/probe-create` | New catalog probe, authored here rather than imported. |
| `POST /api/ide/probe-save` · `POST /api/ide/probe-definition` | Write `check.js` / `probe.yml` **at a `ref`**. |
| `GET /api/ide/probe-versions?id=` | The version bar: `live`, `liveDirty`, `origin`, `releases[]`, `dev[]`. |
| `POST /api/ide/probe-version-create` · `-release` · `-activate` · `-delete` | The authoring loop below. |
| `POST /api/ide/test` | Run a script server-side against sample params. |
| `POST /api/ide/test-on-agent` → `GET /api/ide/test-on-agent/{id}` | Run it **on a real agent** and poll the result. Async: the POST returns an id. `{agent, id}` runs the INSTALLED probe; `{agent, ref}` runs that version; `{agent, source}` runs an inline script and wins over both. |
| `GET/POST /api/ide/probe-bindings` | Which hosts a probe is bound to. |
| `GET/POST /api/ide/probe-svg` | The probe's infographic. |
| `GET/POST /api/ide/command-*` | The same surface for agent commands, **including versions** — `command-versions`, `command-version-create`, `-release`, `-activate`, `-delete`. |

### 🚨 A plain save is refused now — probes AND commands are versioned

Every entry has three kinds of version, named by a `ref`: `live` (what agents run),
`dev/<label>` (a draft) and `rel/<version>` (an immutable release). A write with no `ref`
means `live`, and that is the refused case more often than not:

| You post | You get |
|---|---|
| `{id, source}` on something that came from a library | `409 <id> runs <library> <version> — edits go into a dev version` |
| `{id, source, ref:"rel/1.0.0"}` | `409 Release versions are immutable — branch a dev version from it` |
| `{id, source, ref:"dev/my-fix"}` | `200` — and no agent is affected until you release and activate |

Nothing ships in the server jar any more, so on a typical board **almost every installed entry
is library-sourced** and the bare save is the one that fails. The loop:

```bash
K="X-API-Key: $STATUS_API_KEY"; ID=disk-space

# 0. where does it come from, and what versions exist?
curl -s -H "$K" "$STATUS_URL/api/ide/probe-versions?id=$ID"

# 1. branch a draft off what is live
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d "{\"id\":\"$ID\",\"label\":\"my-fix\",\"basedOn\":\"live\"}" \
  "$STATUS_URL/api/ide/probe-version-create"

# 2. edit the draft — ref in the body for a write, ?ref= for a read
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d "{\"id\":\"$ID\",\"ref\":\"dev/my-fix\",\"source\":\"function check(ctx){…}\"}" \
  "$STATUS_URL/api/ide/probe-save"

# 3. run THAT draft on a real agent before it goes anywhere near live
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d "{\"agent\":\"host-01\",\"id\":\"$ID\",\"ref\":\"dev/my-fix\"}" \
  "$STATUS_URL/api/ide/test-on-agent"

# 4. release it, and activate in the same call (STATUS_ADMIN only)
curl -s -X POST -H "$K" -H 'Content-Type: application/json' \
  -d "{\"id\":\"$ID\",\"ref\":\"dev/my-fix\",\"version\":\"1.1.0\",\"note\":\"why\",\"activate\":true}" \
  "$STATUS_URL/api/ide/probe-version-release"
```

**Rollback is the same verb as activate.** `POST /api/ide/probe-version-activate`
`{id, version}` with an older release's version, and `discardLiveEdits: true` if the live copy
was edited in place. Swap `probe-` for `command-` and every step is identical.

⛔ **`POST /api/ide/toggle-dev` and `POST /api/ide/probe-dev` answer `410 Gone`** for both
kinds. `dev: true` in a manifest was the whole dev/live split before the version store; the
410's body names that kind's own version endpoints.

### Where probes and commands come from: libraries

⛔ **`POST /api/catalog/install/{id}` and `POST /api/catalog/update/{id}` answer `410 Gone`.**
There is no built-in catalog. Both kinds are imported from a git **library**:

```bash
curl -s -H "$K" "$STATUS_URL/api/libraries"                      # what is configured
curl -s -H "$K" "$STATUS_URL/api/libraries/<lib>/probes"         # what it offers
curl -s -X POST -H "$K" "$STATUS_URL/api/libraries/<lib>/probes/<id>/import"
curl -s -X POST -H "$K" "$STATUS_URL/api/libraries/<lib>/commands/<id>/import"
```

⚠️ **The catalog key is `(kind, id)`** — a probe and a command may share an id. Endpoints that
take a bare id (`/api/catalog/uninstall/{id}` and its `/impact`) take `?kind=probe|command`;
with both kinds installed and no `kind`, they refuse with a `409` naming both.

⚠️ **`builtinIds`, `availableProbes` and `availableCommands` in `GET /api/catalog` are
permanently empty.** They are kept only so an older client still decodes. Anything that decided
"Built-In" from `builtinIds` shows nothing at all now, with no error — read `liveLibrary` /
`liveSource` on the entry (or `origin` from the versions call) and say "Yours" when there is
none. Details: `references/api-surface.md`.

To change what a probe **displays** rather than what it checks, edit the `layout:` block in
its definition: `GET /api/ide/probe-definition?id=<probe>` → edit → `POST` it back **at a dev
ref**. The widget vocabulary and each widget's fields are in the `status-server-ops` skill
(`references/widgets.md`).

**Always `test-on-agent` before releasing.** A script that passes `test` server-side can
still fail on an agent — the server has no JS sandbox and silently degrades unsandboxed
probes to a plain HTTP check (trap 4 in `status-server-ops`).

### 🔐 These endpoints are role-gated, and that gate matters

`/api/ide/**` requires **`STATUS_ADMIN` or `INFRA_ADMIN`**. This is deliberate and it is not
bureaucracy: `POST /api/ide/probe-save` writes `check.js` into the catalog, and **the agents
on every host then execute it**. An API key that can reach these endpoints can run arbitrary
code on every machine the board monitors.

One tier tighter, **`STATUS_ADMIN` alone**, for the four verbs where that execution actually
happens or history is destroyed:

| Verb | Why it is tighter |
|---|---|
| `probe-version-release` / `command-version-release` | With `activate: true` it ships code to every agent in one call |
| `probe-version-activate` / `command-version-activate` | The moment a command is activated it can run with `ctx.shell` on a production host |
| `POST /api/catalog/uninstall/{id}` | Takes the entry's whole version history with it |
| `POST` / `PATCH` / `DELETE /api/libraries/**` | Adding a library points your hosts at someone else's repository |

⚠️ **Write access to a library repository is write access to every monitored host**, because
`ctx.shell` is live on an agent unless that agent runs with `AGENT_READONLY`. The perimeter is
the repo, not the endpoint — protect its branches accordingly.

| Practice | Why |
|---|---|
| Mint a **separate, read-only key** for dashboards, bots and anything ambient | Read keys cannot reach `/api/ide/**` at all |
| Reserve admin-role keys for deliberate authoring sessions, and rotate them | The blast radius is every agent, not just the board |
| Never paste an admin key into a shared config, CI variable or chat | Same reason |

> **If your deployment predates 2026-08-26**, `/api/ide/**` carried no authorization beyond
> "authenticated", so *any* valid key could write executable probe scripts. Upgrade, then
> rotate every key that existed before the upgrade.

## ⚠️ What you write over the API is not durable

`POST /api/infrastructure/config` and the Probe IDE both take effect immediately — and both can
be **wiped by the next redeploy**, if your deployment ships `infrastructure.yml` and the probe
directory. Many do, and some sync that directory with `--delete`, so a probe that exists only
in the IDE is one deploy from gone. The symptom is `No script source for probe`: the config
still names a probe id whose folder no longer exists.

Iterate over the API, then persist the result wherever your deployment reads from. If a change
"keeps reverting", look for a deployment before suspecting the server.

⛔ **Never deploy in order to apply a config or probe change.** The API already applied it live,
without dropping a session; the repo commit only makes it survive the NEXT deploy. Deploying to
"publish" an API change restarts the server and overwrites live config from the repo — so if the
repo is behind, the deploy reverts the very thing you were publishing. Deploy only for server
code, the Dockerfile or compose.

Credentials and probe
history live in their own databases and are not usually shipped, so those survive.

**A library changes this for probe and command content.** An imported entry's source of truth is
the git repository, and a library with `installAll` re-imports what is missing after each
successful refresh — so a probe that came from a library comes back on its own after a deploy
wipes the probe directory, while one authored only in the IDE does not. (An entry deliberately
uninstalled is on the library's `excluded` list and is *not* re-imported; that is what makes
uninstall stick.) That is the strongest argument for putting a probe you care about in a library
rather than leaving it live-edited.

⚠️ The version store is `.versions/` **inside each entry's directory**, so a `--delete` sync of
the probe folder takes release and draft history with it. The library can restore the entry; it
cannot restore your local releases.

## Config IS writable over the API — but the write is lossy

`POST /api/infrastructure/config` saves and reloads in one call. Two costs, both silent:

- **Every comment in `infrastructure.yml` is erased**, because the save re-serialises from the
  object graph.
- **Any field the model does not represent is dropped.** It is accepted, and gone on read-back.

So verify a write by reading it back and checking your field is still there — `"ok"` means the
write landed, not that it kept what you sent. For anything meant to last, edit the
`infrastructure.yml` your deployment ships; see the durability warning above and
`status-server-ops` → *Where a change actually lives*.

## MCP server — not currently distributed

Status Chat (the desktop client) hosts an MCP server on a Unix socket for its chat responder:
17 tools over the endpoints above. The board tool reads `/api/status/summary`, the tree tool
takes `root` and `problems_only`, and the incident tools go through `/api/workflows/incident`
(list, get, create, comment, `transition_incident`, `resolve_incident`). It is **not part of
this plugin and not shipped to customers**. Everything in this skill works over plain HTTP, so
nothing here depends on it.

`errors` counts probes in ERROR only. Before 2026-10-08 it also counted UNKNOWN (a probe not yet
run, or whose agent went quiet), so right after a restart it read about 3 times too high; on an
older build, count `probeDetails` by `state` instead of trusting the number.

## See also

`status-server-ops` — the infrastructure model, probe catalog authoring, applying config
changes without dropping sessions, and the five wiring traps that fail as silence.
