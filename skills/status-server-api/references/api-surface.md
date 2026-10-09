# Every API endpoint

Generated from the server source, so it reflects what exists rather than what was
written down.

| Mark | Gate |
|---|---|
| 🔒 | `STATUS_ADMIN` **or** `INFRA_ADMIN` |
| 🔑 | `STATUS_ADMIN` **only** — one tier tighter, for the writes that ship code or delete history |
| — | any valid key **that carries at least one Status role** |

⚠️ **"Authenticated" is no longer enough.** A signed-in account with no Status role gets a
`403` on every `/api/**` path but one: `GET /api/user/profile`, which the waiting screen polls
until an admin grants a role. Anonymous still gets `401`. So a `403` on a read as plain as
`/api/tree` means *this account has no role yet*, not *this endpoint is admin-only*.

Ask a running server what it supports with `GET /api/capabilities` — that beats this file
if they ever disagree.


## Reading state

| Endpoint | |
|---|---|
| `GET /api/auth-config` | Auth discovery — always public so the mobile app knows how to authenticate |
| `GET /api/commands/result?command=<id>` 🔒 | Latest structured result for a command. `command` is required — without it, `400` |
| `GET /api/events` |  |
| `GET /api/global` | Global state — replaces GlobalModelAdvice |
| `GET /api/history` |  |
| `GET /api/history/{id}` |  |
| `POST /api/history/{id}/revert` |  |
| `GET /api/presence` |  |
| `POST /api/presence/ping` |  |
| `POST /api/probes/debug` | Toggle debug mode for a probe — enables ctx |
| `GET /api/probes/history?probe=<name>` | History at a given `resolution` and `path`. **`probe` is required** — without it, `400` |
| `GET /api/probes/history/list` | List all probes with history data |
| `GET /api/probes/infographic?probe=<name>` | Infographic SVG + resolved patches |
| `GET /api/probes/result?probe=<name>` | Latest structured result. **`probe` is required** |
| `GET /api/probes/snapshot?probe=<name>` | Latest value of every path. **`probe` is required** |
| `GET /api/status` | The whole board (both trees, sparklines). 2.8 MB on a 75-probe board |
| `GET /api/status/summary` | Counters, mutes, work, untracked issues, flat probe list; no trees. About 20 KB |
| `GET /api/tree` | Path-based tree built from all probe names + history probes |
| `GET /api/untracked-issues` | Degraded probes in mapped projects — used by the topbar alert badge |

## Capabilities

| Endpoint | |
|---|---|
| `GET /api/capabilities` |  |

## Infrastructure config

| Endpoint | |
|---|---|
| `GET /api/infrastructure/config` | Current infrastructure config |
| `POST /api/infrastructure/config` | Save the config, reload the scheduler, record a history entry |
| `GET /api/infrastructure/hosts` | Host names, for pickers and for resolving where a probe would run |
| `GET /api/infrastructure/types` | Service types from the catalog, with their param metadata |

## Probe authoring (Probe IDE)

⚠️ **Every content endpoint here takes a `ref`** — `live` (the default), `dev/<label>` or
`rel/<version>`. A GET takes it as `?ref=`, a POST as a `ref` field in the body. Two refusals
follow from that and they are the ones that surprise a caller written before versioning:

| Write | Answer |
|---|---|
| `ref=rel/<version>` | `409 Release versions are immutable — branch a dev version from it` |
| `ref=live` on an entry whose live version came from a library | `409 <id> runs <library> <version> — edits go into a dev version` |

Since nothing ships in the jar any more, **almost everything installed is library-sourced**, so
a bare `probe-save`/`command-save` with no `ref` is the refused case, not the working one. The
authoring loop is `*-version-create` → edit at `ref=dev/<label>` → `*-version-release`
→ `*-version-activate`. See *Versions* below and `SKILL.md` → *Probe authoring over the API*.

| Endpoint | |
|---|---|
| `POST /api/ide/command-create` 🔒 | Create a new command from scratch |
| `GET /api/ide/command-definition` 🔒 | Get command definition YAML |
| `POST /api/ide/command-definition` 🔒 | Save command definition YAML |
| `POST /api/ide/command-save` 🔒 | Save command script (run |
| `GET /api/ide/command-source` 🔒 | Get script source for a command definition |
| `GET /api/ide/commands` 🔒 | List installed command definitions |
| `GET /api/ide/list` 🔒 | List all scripts with their custom builtin flag |
| `GET /api/ide/probe-bindings` 🔒 | Get infographic bindings YAML for a probe |
| `POST /api/ide/probe-bindings` 🔒 | Save infographic bindings YAML |
| `POST /api/ide/probe-create` 🔒 | Create a new probe from scratch |
| `GET /api/ide/probe-definition` 🔒 | Get probe definition YAML |
| `POST /api/ide/probe-definition` 🔒 | Save probe definition YAML |
| `POST /api/ide/probe-dev` 🔒 | **410 Gone** — alias of `toggle-dev`, retired with it |
| `POST /api/ide/probe-save` 🔒 | Save probe script (`check.js`) at `ref` — 409 on a library-sourced `live` |
| `GET /api/ide/probe-source?name=<id>` 🔒 | Get `check.js` source. `?name=` and `?id=` are both accepted now; a missing one is a 400 naming both |
| `GET /api/ide/probe-svg` 🔒 | Get infographic SVG templates (light + dark) for a probe |
| `POST /api/ide/probe-svg` 🔒 | Save infographic SVG templates (light + optional dark) |
| `GET /api/ide/probes` 🔒 | List installed probe definitions from the catalog (the editable probe types, not running instances) |
| `POST /api/ide/save` 🔒 | Save a custom legacy script |
| `GET /api/ide/script/{name}` 🔒 | Get script source |
| `POST /api/ide/test` 🔒 | Test-run: fetch URL, run script, return result tree |
| `POST /api/ide/test-on-agent` 🔒 | Queue a test-probe execution on a specific agent |
| `GET /api/ide/test-on-agent/{id}` 🔒 | Poll for test-probe result |
| `POST /api/ide/toggle-dev` 🔒 | **410 Gone** for both kinds — `dev: true` was replaced by the version store. The body names that kind's own version endpoints |

### Versions

The same five verbs for each kind, over one implementation. `probe-…` and `command-…` are the
only difference; a probe and a command may share an id, so the endpoint name carries the kind
rather than a `?kind=` param.

| Endpoint | |
|---|---|
| `GET /api/ide/probe-versions?id=` 🔒 | `{id, kind, live, liveDirty, origin, releases[], dev[]}` — the whole version bar in one read |
| `GET /api/ide/command-versions?id=` 🔒 | The same for a command |
| `POST /api/ide/probe-version-create` 🔒 | `{id, label, basedOn}` — new draft; `basedOn` defaults to `live`. Returns `{ref:"dev/<label>"}` |
| `POST /api/ide/command-version-create` 🔒 | |
| `POST /api/ide/probe-version-delete` 🔒 | `{id, ref:"dev/<label>"}` — drafts only; `live` and `rel/*` are refused |
| `POST /api/ide/command-version-delete` 🔒 | |
| `POST /api/ide/probe-version-release` 🔒🔑 | `{id, ref, version, note, activate}` — **`STATUS_ADMIN` only**, one tier above the rest of `/api/ide/**`: a release plus `activate: true` ships code to every agent in one call |
| `POST /api/ide/command-version-release` 🔒🔑 | Same, and for a sharper reason — a command runs with `ctx.shell` the moment it is activated |
| `POST /api/ide/probe-version-activate` 🔒🔑 | `{id, version, discardLiveEdits}` — this is also rollback: activating an older release is the same verb |
| `POST /api/ide/command-version-activate` 🔒🔑 | |

`origin` on the read tells you provenance: `{kind:"library", library, path, commit,
importedVersion}` for something imported, `null` for something authored here. ⚠️ **Decide "who
wrote this" from `origin` (or a release's `source`/`library`), never from `builtinIds`** — see
the catalog section.

## Probe and command catalog

| Endpoint | |
|---|---|
| `GET /api/catalog` | Full catalog state — see the key list below |
| `POST /api/catalog/install/{id}` | **410 Gone** — there is no built-in catalog to install from. Use `POST /api/libraries/{name}/{probes\|commands}/{id}/import` |
| `POST /api/catalog/update/{id}` | **410 Gone** — a former built-in's update arrives as a library update now |
| `GET /api/catalog/sync` 🔒 | The installed catalog + its hash. **Admin only** — it used to be public "for agents", but an agent receives its scripts inside its heartbeat's probe assignments and has never called this |
| `GET /api/catalog/uninstall/{id}/impact` 🔒🔑 | What uninstalling loses: `releases`, `devVersions`, what is using it, and a `message` built from them. `?kind=command` asks about the command of that id |
| `POST /api/catalog/uninstall/{id}` 🔒🔑 | Uninstall. `STATUS_ADMIN` only — it takes the whole version history with it. Refused 409 while wired in `infrastructure.yml` (`wiredChecks`) or, for a command, while a preset or a queued dispatch names it |

### The catalog key is `(kind, id)`, not `id`

A probe and a command **may share an id**. So on `/api/catalog/uninstall/{id}` and its
`/impact`: an explicit `?kind=probe`/`?kind=command` wins; with exactly one kind installed that
kind is used; **with both installed and no `kind`, nothing happens and you get a `409` whose
`kinds` array names them**. A client that sends a bare id and expects success is the one that
breaks here.

### `GET /api/catalog` — the keys that matter

| Key | |
|---|---|
| `installedProbes` · `installedCommands` | Maps of what is live, keyed by id |
| `libraryProbes[]` · `libraryCommands[]` | What every configured library offers, each row with `library`, `version`, `installed`, `liveVersion`, `liveSource`, `updateAvailable`, `state` |
| `dormantProbes` · `dormantCommands` | Installed, but their library is switched off — a dormant command refuses at dispatch with a stated reason rather than running a script with no source |
| `updates[]` | Probe updates. ⚠️ Command updates are under **`libraryCommandUpdates[]`**, deliberately not merged in: every button in the probe Updates list posts to the *probe* update endpoint |
| `modifiedEntries[]` | `probe:<id>` / `command:<id>` — locally edited. The older `modified[]` is bare ids and cannot distinguish the kinds |
| `catalogHash` | Change detection |
| `builtinIds` · `availableProbes` · `availableCommands` | ⛔ **Always empty.** Kept so an old client still decodes, but nothing is built in any more. **A "Built-In" badge computed from `builtinIds` silently disappears** — this is exactly the bug the iOS client shipped. Read `origin.library` / `liveLibrary` instead, and show "Yours" when there is none |

An entry in `installedProbes`/`installedCommands` also carries `liveSource` (`library` or
`local`), `liveLibrary`, `version` and `deliverable` — enough to label provenance without a
second call.

## Probe and command libraries

**This is where probes and commands come from.** A library is a git repository holding
`library.yml` plus `probes/<id>/` and `commands/<id>/` directories; the server clones it,
verifies it, and offers its entries for import. Both kinds come from one, and one library may
publish both.

| Endpoint | |
|---|---|
| `GET /api/libraries` 🔒 | Configured libraries, each with `source`, `tracking`, `lastCommit`, `probeCount`, `commandCount`, `enabled`, `autoUpdate`, `autoApplyProbeUpdates`, `autoApplyCommandUpdates`, `installAll`, `credential`, `lastError`, `warnings[]` |
| `GET /api/libraries/{name}/probes` 🔒 | What it offers, with per-entry `installed` / `updateAvailable` / `collision` |
| `GET /api/libraries/{name}/probes/{id}` 🔒 | One entry, with its files |
| `GET /api/libraries/{name}/commands` 🔒 | The same for commands |
| `GET /api/libraries/{name}/commands/{id}` 🔒 | |
| `GET /api/libraries/{name}/impact` 🔒 | What turning it off or removing it would strip |
| `POST /api/libraries` 🔑 | Add one |
| `PATCH /api/libraries/{name}` 🔑 | Any of `{enabled, autoUpdate, autoApplyProbeUpdates, autoApplyCommandUpdates, ref, credential}`. The two auto-apply flags are **API-only** — the UI shows neither |
| `DELETE /api/libraries/{name}` 🔑 | Remove it — and its entries' presets and wiring, in one recorded `infrastructure.yml` change |
| `POST /api/libraries/{name}/refresh` 🔑 | Re-fetch now |
| `POST /api/libraries/{name}/probes/{id}/import` 🔑 | **The replacement for `catalog/install`** — records a release, an origin and a diff |
| `POST /api/libraries/{name}/commands/{id}/import` 🔑 | |
| `POST /api/libraries/{name}/probes/{id}/update` 🔑 | **The replacement for `catalog/update`** |
| `POST /api/libraries/{name}/commands/{id}/update` 🔑 | ⚠️ Do **not** post a command id to the probe endpoint — the kind is in the path, and the catalog's `libraryCommandUpdates[]` is kept separate precisely so a UI cannot make that mistake |

Three things worth knowing before writing against these:

| | |
|---|---|
| A library's entries run on your hosts | `ctx.shell` is live on an agent unless `AGENT_READONLY` is set, so **write access to a library repository is write access to every monitored host**. The perimeter is the repo, not the endpoint |
| `autoApplyCommandUpdates` is independent of `autoApplyProbeUpdates` | Off by default, and the probe flag never carries a command with it |
| A switched-off library leaves its entries **dormant** | They stay installed and listed under `dormantProbes`/`dormantCommands`; a dormant command refuses at dispatch (`refused: <id> is not runnable: library <name> is off`) rather than running a script whose source is gone |

## Agents

Registration, heartbeat, approval and command dispatch for the per-host agent. **Registration
is open** — an agent self-registers — but everything after that requires the agent to be
approved and to sign requests with its Ed25519 key. `autoApproveAgents` defaults to `false`;
approve deliberately. Agents also fetch probe credentials here, over a signed request.

| Endpoint | |
|---|---|
| `GET /api/agents` | List all agents (admin) — parses heartbeat data and maps to UI-friendly field names |
| `POST /api/agents/action` | Trigger a probe action on the agent that runs the probe |
| `GET /api/agents/action/{executionId}` | Poll action execution status + streamed log entries |
| `POST /api/agents/register` | Agent registers itself on first startup |
| `POST /api/agents/{name}/approve` | Approve a pending agent (admin) |
| `POST /api/agents/{name}/command-complete` | Agent reports command execution completion with results |
| `POST /api/agents/{name}/command-result` | Agent returns the result of a command (e |
| `POST /api/agents/{name}/command-stream` | Agent streams a command execution log entry (line-by-line) |
| `POST /api/agents/{name}/docker/{action}` | Send a Docker control command to an agent (restart stop start) |
| `POST /api/agents/{name}/heartbeat` | Agent pushes host metrics and container inventory |
| `PUT /api/agents/{name}/labels` | Set server-side labels for an agent (admin) |
| `GET /api/agents/{name}/log-entries` | Read log entries by canonical URI |
| `POST /api/agents/{name}/log-stream` | Agent pushes log stream batches |
| `DELETE /api/agents/{name}/log-subscriptions` | Remove a log subscription (admin) |
| `GET /api/agents/{name}/log-subscriptions` | Get log subscriptions for an agent (admin) |
| `POST /api/agents/{name}/log-subscriptions` | Add or toggle a log subscription (admin) |
| `GET /api/agents/{name}/logs` | List all log sources for an agent |
| `POST /api/agents/{name}/logs` | Request logs from an agent for a specific container |
| `GET /api/agents/{name}/logs/{source}` | Read raw log lines for an agent+source |
| `GET /api/agents/{name}/logs/{source}/parsed` | Read parsed structured log entries |
| `POST /api/agents/{name}/probe-results` | Agent posts probe execution results |
| `POST /api/agents/{name}/probe-stream` | Agent streams probe debug log entries |
| `POST /api/agents/{name}/reject` | Reject remove an agent (admin) |
| `POST /api/agents/{name}/retention` | Set log retention for an agent (admin) |

## Sites & floor plans

Read-only views of the `sites:` section of `infrastructure.yml` — geographic placement and
drawn floors, plus the one thing the config editor cannot do: uploading and serving the floor
**images**. Editing sites themselves goes through the infrastructure config, same YAML.

| Endpoint | |
|---|---|
| `GET /api/sites` |  |
| `POST /api/sites/images` |  |
| `GET /api/sites/images/{filename:.+}` |  |
| `GET /api/sites/{name}` |  |

## Topology layouts

Saved arrangements of the 3D topology view — the plate visualisation, not the probe board.
Layouts are named; one is active at a time.

| Endpoint | |
|---|---|
| `GET /api/topology/layouts` |  |
| `POST /api/topology/layouts/activate/{name}` | Switch the active layout (persisted, so GET returns it next time) |
| `DELETE /api/topology/layouts/{name}` | Remove a layout (including the default one) |
| `PUT /api/topology/layouts/{name}` | Replace the named layout's pinned list |

## Drills

Scheduled practice alerts that ask responders to acknowledge, with a leaderboard of who
answered and how fast. `GET /api/drills/active` is what the nav badge polls.

⚠️ `totalResponders` counts members of the responder role in the identity directory, so it
reads **0** if that role has no members — producing "1 of 0 responded". If the count looks
wrong, check role membership before suspecting the drill.

| Endpoint | |
|---|---|
| `GET /api/drills` |  |
| `GET /api/drills/active` | Active drill only — polled by the nav badge |
| `GET /api/drills/config` |  |
| `POST /api/drills/config` |  |
| `POST /api/drills/trigger` |  |
| `POST /api/drills/{id}/accept` |  |
| `POST /api/drills/{id}/close` |  |

## Handled

One reference per board node or todo saying somebody is on it, optionally pointing at a ticket in
any tracker. Status clears it after every covered probe is continuously `OK` for `clearAfter`
(default 1 h), when a todo stops being reported, or 24 h after the target disappears. It does not
silence alerts. When several cover one probe, the nearest wins (probe or dependency, service, app,
project or host; the newer on a tie).

| Endpoint | |
|---|---|
| `GET /api/handled` | `{active:[…]}`; `?history=true` adds `history:[…]` (newest 200) with `clearedAt` and `clearedReason` (`green` `done` `target-gone` `manual` `tracker`). Each reference: `id, target, url, title, system, externalId, note, createdBy, createdAt, clearAfterSeconds, greenSince, missingSince, clearedAt, clearedReason, active` |
| `PUT /api/handled` | `{target:{kind,type,path,todoId?}, url?, title?, system?, externalId?, note?, clearAfter?}` (`clearAfter`: seconds, or a duration such as `"30m"`; at least 1 s). Replaces the active reference on that target; the same `system`+`externalId` again changes nothing. Errors `{error:{code,message}}`: `400` `missing_target`, `missing_fields` (+`missing:[…]`), `bad_target`, `bad_clear_after`, `invalid_field` (`url` 2048, `title` 300, `externalId` 200, `note` 2000 characters; the message names the field), `bad_request`; `404 unknown_target` (nothing on the board under that target; nothing stored); `409 conflict` (someone else marked it at the same moment); `500 not_stored` (the write itself failed; nothing stored) |
| `DELETE /api/handled?kind=&type=&path=&todoId=` | `{ok:true}`, or `{ok:false}` when nothing was active. Reason `tracker` for an API key, `manual` for a session or a bearer token |
| `GET /api/handled/trackers` | enabled connectors `[{id, displayName, kind}]` |
| `GET /api/handled/trackers/{id}/search?q=` | `[{id, title, url, state}]`; `404 unknown_tracker`; `502 tracker_error` when the tracker refuses, cannot be reached or does not answer within 10 s |
| `POST /api/handled/trackers/{id}/tickets` | `{target, title?, note?}` → `{ticket, ref}`: creates the ticket and sets the reference. No ticket filed on `400 invalid_field`, `404 unknown_tracker` or `404 unknown_target` (checked before the tracker is asked). `409 conflict` and `not_marked` (400 refused, 500 store failed) carry the created `ticket` beside `error`: do not retry. `502 tracker_error` marks nothing; a timed-out create says the ticket may still have been created |
| `GET /api/handled/trackers/config` 🔒 | every connector's settings, disabled ones included, never the key |
| `PUT /api/handled/trackers/config/{id}` 🔒 | create or replace one connector: `{kind?, displayName?, serverUrl, workspace, ticketType?, credentialName, enabled?}`. `400`: `missing_fields`, `bad_tracker_id`, `unsupported_kind`, `bad_server_url`, `unknown_credential`, `invalid_config` |
| `DELETE /api/handled/trackers/config/{id}` 🔒 | `{ok:true}`, or `{ok:false}` when there was none |

No `GET /api/handled/trackers/config/{id}` exists; `config` is a reserved tracker id.

## Workflows

User-defined record types with a state machine — the system that replaced legacy incidents. A
*type* declares fields, nodes (states) and edges (transitions); an *instance* is one record
moving through it. `GET /api/workflows/types` lists what exists, `POST /api/workflows/{type}`
creates a record, and `POST /api/workflows/{type}/{id}/transitions` moves it along an edge.

⚠️ **`name` is an i18n map, not a string** — `{"en":"Incident","de":"Vorfall"}`. A client that
renders it directly prints an object. The same applies to type and field labels throughout this
subsystem.

An edge may carry `requireFields`, `requireRole` and a `when` expression. ⚠️ `when` is a **hard
gate** in this implementation: a transition whose condition is false is refused with
`when_condition_false`, not allowed through with a warning.

| Endpoint | |
|---|---|
| `GET /api/workflows/attachable?kind=` | Types that declare an attach point for that ref kind. ⚠️ `kind` is **required** — omit it and you get a 400 |
| `GET /api/workflows/field-types` | The field-type definitions this server resolved, for the SPA |
| `GET /api/workflows/types` |  |
| `POST /api/workflows/types` 🔒 | |
| `GET /api/workflows/types/{id}` | One type's whole state machine — nodes, edges, fields |
| `PUT /api/workflows/types/{id}` 🔒 | |
| `GET /api/workflows/{type}/scripts/{filename}` 🔒 · `PUT` 🔒 | A type's action scripts |
| `GET /api/workflows/{type}` | List instances |
| `POST /api/workflows/{type}` | Create one — `{id?, fields:{…}}`, answers `201` |
| `PATCH /api/workflows/{type}/{id}/fields` | `{fields:{…}, version}` — edit without transitioning |
| `DELETE /api/workflows/{type}/{id}` |  |
| `GET /api/workflows/{type}/{id}` |  |
| `POST /api/workflows/{type}/{id}/actions/{actionId}` |  |
| `GET /api/workflows/{type}/{id}/archived` |  |
| `POST /api/workflows/{type}/{id}/attach` |  |
| `DELETE /api/workflows/{type}/{id}/attach/{refKind}/{refId}` |  |
| `POST /api/workflows/{type}/{id}/comments` | `{text}` — ⚠️ the field is `text`, not `content` or `comment`; a wrong name posts an empty comment and answers `204` |
| `POST /api/workflows/{type}/{id}/fields/{fieldName}/files` |  |
| `DELETE /api/workflows/{type}/{id}/fields/{fieldName}/files/{filename:.+}` |  |
| `GET /api/workflows/{type}/{id}/fields/{fieldName}/files/{filename:.+}` |  |
| `GET /api/workflows/{type}/{id}/fields/{fieldName}/files/{filename:.+}/thumbnail` |  |
| `POST /api/workflows/{type}/{id}/references` | Attach a reference (e |
| `DELETE /api/workflows/{type}/{id}/references/{kind}` | Remove a reference from an instance's { |
| `POST /api/workflows/{type}/{id}/transitions` | `{to, version, edgeId?, fields?, override?}`. `override` is opt-in — absent or false means a failed flow rule is a hard error |
| `POST /api/workflows/{type}/{id}/unarchive` |  |

## Messaging

In-app conversations, including the surface a chat assistant uses to talk to operators.

| Endpoint | |
|---|---|
| `GET /api/messages/conversations` |  |
| `POST /api/messages/conversations/start` |  |
| `GET /api/messages/conversations/{id}` |  |
| `POST /api/messages/conversations/{id}` |  |
| `POST /api/messages/conversations/{id}/read` |  |
| `POST /api/messages/conversations/{id}/typing` | Notify that the current user is typing in a conversation |
| `POST /api/messages/page` | Send an urgent page to a user |
| `POST /api/messages/page/{conversationId}/respond` | Respond to a page with an ETA |
| `GET /api/messages/unread-count` |  |
| `GET /api/messages/users` | List all users available for messaging (any authenticated user can call this) |

## Notifications

| Endpoint | |
|---|---|
| `DELETE /api/push/register` | Unregister APNs device token — fire-and-forget |
| `POST /api/push/register` | Register APNs device token — proxied to Push relay |
| `POST /api/telegram/webhook` | Receives Telegram Bot API webhook updates |

## Identity & roles

What the active identity backend can do, so a client can hide what it does not support, plus
TOTP/MFA enrolment. Roles themselves come from the identity provider — the server maps them onto
`STATUS_ADMIN`, `INFRA_ADMIN`, `PROBE_EDITOR`, `VIEWER` and the rest. See *Getting a key* in
`status-server-ops` for how a key inherits and can narrow them.

| Endpoint | |
|---|---|
| `GET /api/iam/capabilities` |  |
| `DELETE /api/iam/mfa/totp` | Removes TOTP from the authenticated user's account |
| `POST /api/iam/mfa/totp/confirm` | Confirms enrollment by verifying the first code from the authenticator app |
| `POST /api/iam/mfa/totp/enroll` | Begins TOTP enrollment |

## Your account

| Endpoint | |
|---|---|
| `POST /api/user/account/delete` |  |
| `GET /api/user/api-keys` |  |
| `POST /api/user/api-keys` |  |
| `POST /api/user/api-keys/{id}/revoke` |  |
| `GET /api/user/page-prefs` |  |
| `POST /api/user/page-prefs` |  |
| `GET /api/user/profile` | Current user profile and roles |
| `GET /api/user/schedules` |  |
| `POST /api/user/schedules` |  |
| `POST /api/user/schedules/timezone` |  |
| `DELETE /api/user/schedules/{id}` |  |
| `POST /api/user/test-email` |  |
| `POST /api/user/update-email` |  |
| `POST /api/user/update-name` |  |
| `POST /api/user/update-password` |  |
| `POST /api/user/update-phone` |  |

## Administration

Server info, users and roles, notification config, retention presets, and the storage tools —
including `/api/admin/storage/stale` and `/api/admin/storage/cleanup`, covered in
`status-server-ops`.

| Endpoint | |
|---|---|
| `GET /api/admin/agents/{name}/commands` | Catalog commands the runtime menu offers for this agent |
| `GET /api/admin/agents/{name}/configured-instances` | Configured command instances for an agent — operator presets from { |
| `GET /api/admin/agents/{name}/execution/{execId}` | Get execution log entries + result for a running completed command |
| `POST /api/admin/agents/{name}/run-script` | Queue a catalog command for execution on the named agent |
| `GET /api/admin/credentials` |  |
| `POST /api/admin/credentials` |  |
| `DELETE /api/admin/credentials/{id}` |  |
| `GET /api/admin/credentials/{id}` |  |
| `PUT /api/admin/credentials/{id}` |  |
| `GET /api/admin/credentials/{id}/log` |  |
| `GET /api/admin/notifications` |  |
| `POST /api/admin/notifications/{section}` |  |
| `POST /api/admin/notifications/{section}/toggle` |  |
| `GET /api/admin/retention-presets` |  |
| `POST /api/admin/retention-presets` |  |
| `DELETE /api/admin/retention-presets/{name}` |  |
| `GET /api/admin/server-info` | version, `commit` (from 2026-10-08), uptime, heap, `probeErrors` `probeWarnings` `probeUnknown` |
| `GET /api/admin/storage` |  |
| `POST /api/admin/storage/cleanup` | Delete abandoned series |
| `POST /api/admin/storage/delete` |  |
| `GET /api/admin/storage/stale` | History series that look abandoned: nothing has written to them for { |
| `GET /api/admin/users` |  |
| `DELETE /api/admin/users/{userId}` |  |
| `POST /api/admin/users/{userId}/roles` |  |
