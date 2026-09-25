# Status Server API Endpoints

## Authentication

| Method | Description |
|--------|-------------|
| **Public** | No authentication required |
| **Agent** | Ed25519 signature (verified in controller) |
| **API Key** | `X-API-Key` header (grants ROLE_USER + ROLE_TOWER_USER) |
| **Session** | OAuth2/Keycloak session login |
| **JWT** | OAuth2 JWT bearer token |

> All `/api/**` endpoints require authentication — API key (`X-API-Key` header), Keycloak session, or JWT bearer token.

> ⚠️ **And at least one Status role.** An authenticated account with no role gets `403` on every
> `/api/**` path except `GET /api/user/profile`, which the waiting screen polls until an admin
> grants one. Anonymous gets `401`. So "Auth: API Key/JWT" in the tables below means *a key whose
> owner has a role*; a `403` on a plain read is a missing role, not a missing endpoint.

---

## Health

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/health` | Public | Health check |

## Status & Monitoring

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/status` | API Key/JWT | Full status tree with all projects, hosts, probes |
| GET | `/api/tree` | API Key/JWT | Project navigation tree |
| GET | `/api/global` | API Key/JWT | Global settings (service name, theme) |
| GET | `/api/events` | API Key/JWT | Event log (query: `hours`) |
| GET | `/api/events/stream` | API Key/JWT | SSE real-time event stream |
| GET | `/api/untracked-issues` | API Key/JWT | Errors not linked to a workflow record |
| GET | `/api/auth-config` | Public | Auth provider configuration |

## Probes

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/probes/result` | API Key/JWT | Single probe result (query: `probe`) |
| GET | `/api/probes/history` | API Key/JWT | Probe history data (query: `probe`, `resolution`) |
| GET | `/api/probes/history/list` | API Key/JWT | List probes that have history |
| GET | `/api/probes/snapshot` | API Key/JWT | Current values snapshot |
| GET | `/api/probes/infographic` | API Key/JWT | Rendered SVG infographic (query: `probe`) |
| POST | `/api/probes/debug` | API Key/JWT | Toggle debug logging for a probe |
| GET | `/api/commands/result` | API Key/JWT | Command execution result |

## Agents (Admin)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/agents` | API Key/JWT | List all agents |
| POST | `/api/agents/{name}/approve` | API Key/JWT | Approve pending agent |
| POST | `/api/agents/{name}/reject` | API Key/JWT | Reject/remove agent |
| POST | `/api/agents/{name}/logs` | API Key/JWT | Request log collection |
| POST | `/api/agents/{name}/docker/{action}` | API Key/JWT | Docker container action |
| GET | `/api/agents/{name}/log-subscriptions` | API Key/JWT | List log subscriptions |
| POST | `/api/agents/{name}/log-subscriptions` | API Key/JWT | Add log subscription |
| DELETE | `/api/agents/{name}/log-subscriptions` | API Key/JWT | Remove log subscription |
| GET | `/api/agents/{name}/logs` | API Key/JWT | List available log sources |
| GET | `/api/agents/{name}/logs/{source}` | API Key/JWT | Raw log data |
| GET | `/api/agents/{name}/logs/{source}/parsed` | API Key/JWT | Parsed/structured logs |
| POST | `/api/agents/{name}/retention` | API Key/JWT | Set log retention policy |

## Agents (Agent-facing, Ed25519 Auth)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/agents/register` | Agent | Register new agent |
| POST | `/api/agents/{name}/heartbeat` | Agent | Heartbeat with system metrics |
| POST | `/api/agents/{name}/probe-results` | Agent | Submit probe results |
| POST | `/api/agents/{name}/probe-stream` | Agent | SSE probe result stream |
| GET | `/api/agents/{name}/command-stream` | Agent | SSE command stream |
| POST | `/api/agents/{name}/command-stream` | Agent | Command stream init |
| POST | `/api/agents/{name}/command-result` | Agent | Submit command result |
| POST | `/api/agents/{name}/command-complete` | Agent | Mark command complete |
| POST | `/api/agents/{name}/log-stream` | Agent | Log stream init |

## Catalog

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/catalog` | API Key/JWT | Full catalog (probes + commands). `builtinIds`, `availableProbes` and `availableCommands` are **always empty** — read `liveLibrary`/`liveSource` for provenance |
| GET | `/api/catalog/sync` | **`STATUS_ADMIN`/`INFRA_ADMIN`** | Installed entries + hash. It used to be public "for agents"; agents get their scripts in the heartbeat and never called it |
| POST | `/api/catalog/install/{id}` | — | **410 Gone** — use `/api/libraries/{name}/{probes\|commands}/{id}/import` |
| POST | `/api/catalog/update/{id}` | — | **410 Gone** — use `/api/libraries/{name}/{probes\|commands}/{id}/update` |
| GET | `/api/catalog/uninstall/{id}/impact` | `STATUS_ADMIN` | What uninstalling loses. `?kind=command` for a command |
| POST | `/api/catalog/uninstall/{id}` | `STATUS_ADMIN` | Uninstall. `?kind=` when a probe and a command share the id — with both installed and no `kind`, `409` |

## Probe and command libraries

Where both kinds come from since the jar stopped shipping a catalog. All `STATUS_ADMIN` except
the reads, which take `STATUS_ADMIN` or `INFRA_ADMIN`.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/libraries` | Configured libraries, their commits and counts |
| GET | `/api/libraries/{name}/probes` · `/commands` | What it offers |
| GET | `/api/libraries/{name}/probes/{id}` · `/commands/{id}` | One entry |
| GET | `/api/libraries/{name}/impact` | What turning it off would strip |
| POST | `/api/libraries` | Add one |
| PATCH | `/api/libraries/{name}` | `{enabled, autoUpdate, autoApplyProbeUpdates, autoApplyCommandUpdates, ref, credential}` |
| DELETE | `/api/libraries/{name}` | Remove it, and its entries' presets and wiring |
| POST | `/api/libraries/{name}/refresh` | Re-fetch now |
| POST | `/api/libraries/{name}/probes/{id}/import` · `/update` | Replaces `catalog/install` · `catalog/update` |
| POST | `/api/libraries/{name}/commands/{id}/import` · `/update` | Same for commands — the kind is in the path |

## Credentials Store

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/admin/credentials` | API Key/JWT | List credentials |
| GET | `/api/admin/credentials/{id}` | API Key/JWT | Get credential |
| POST | `/api/admin/credentials` | API Key/JWT | Create credential |
| PUT | `/api/admin/credentials/{id}` | API Key/JWT | Update credential |
| DELETE | `/api/admin/credentials/{id}` | API Key/JWT | Delete credential |
| GET | `/api/admin/credentials/{id}/log` | API Key/JWT | Credential access log |

## IDE / Script Playground

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/ide/list` | API Key/JWT | List all scripts |
| GET | `/api/ide/probes` | API Key/JWT | List probes for IDE |
| GET | `/api/ide/probe-source?name=<id>` | API Key/JWT | Get probe JS source. `?name=` and `?id=` are both accepted; a missing one is a 400 naming both. `?ref=` reads a version |
| GET | `/api/ide/probe-definition` | API Key/JWT | Get probe YAML definition (`?ref=`) |
| POST | `/api/ide/probe-definition` | API Key/JWT | Save probe YAML at `ref` |
| POST | `/api/ide/probe-save` | API Key/JWT | Save probe JS source at `ref` |
| POST | `/api/ide/probe-create` | API Key/JWT | Create new probe |
| GET | `/api/ide/probe-svg` | API Key/JWT | Get probe SVG template (`?ref=`) |
| POST | `/api/ide/probe-svg` | API Key/JWT | Save probe SVG template at `ref` |
| GET | `/api/ide/probe-bindings` | API Key/JWT | Get probe SVG bindings (`?ref=`) |
| POST | `/api/ide/probe-bindings` | API Key/JWT | Save probe SVG bindings at `ref` |
| POST | `/api/ide/toggle-dev` | — | **410 Gone** for probes and commands — the version store replaced `dev: true` |
| POST | `/api/ide/probe-dev` | — | **410 Gone** — alias of the above |

### Versions

Every write above takes `ref`: `live` (default), `dev/<label>` or `rel/<version>`. A `live`
write on a library-sourced entry is `409`, and `rel/*` is `409` (immutable). Swap `probe-` for
`command-` for the command side of each row.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/ide/probe-versions?id=` | API Key/JWT | `{id, kind, live, liveDirty, origin, releases[], dev[]}` |
| GET | `/api/ide/command-versions?id=` | API Key/JWT | |
| POST | `/api/ide/probe-version-create` | API Key/JWT | `{id, label, basedOn}` → `{ref:"dev/<label>"}` |
| POST | `/api/ide/probe-version-delete` | API Key/JWT | `{id, ref:"dev/<label>"}` — drafts only |
| POST | `/api/ide/probe-version-release` | **`STATUS_ADMIN`** | `{id, ref, version, note, activate}` |
| POST | `/api/ide/probe-version-activate` | **`STATUS_ADMIN`** | `{id, version, discardLiveEdits}` — also how you roll back |
| GET | `/api/ide/script/{name}` | API Key/JWT | Get script source |
| POST | `/api/ide/save` | API Key/JWT | Save script |
| POST | `/api/ide/test` | API Key/JWT | Test script locally |
| POST | `/api/ide/test-on-agent` | API Key/JWT | Test script on agent |
| GET | `/api/ide/test-on-agent/{id}` | API Key/JWT | Poll test execution result |
| GET | `/api/ide/commands` | API Key/JWT | List commands |
| GET | `/api/ide/command-source` | API Key/JWT | Get command source |
| GET | `/api/ide/command-definition` | API Key/JWT | Get command YAML |
| POST | `/api/ide/command-save` | API Key/JWT | Save command source |
| POST | `/api/ide/command-definition` | API Key/JWT | Save command YAML |
| POST | `/api/ide/command-create` | API Key/JWT | Create new command |

## User Profile & Settings

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/user/profile` | API Key/JWT | Current user profile |
| POST | `/api/user/update-email` | API Key/JWT | Update email |
| POST | `/api/user/update-name` | API Key/JWT | Update display name |
| POST | `/api/user/update-phone` | API Key/JWT | Update phone number |
| POST | `/api/user/update-password` | API Key/JWT | Update password |
| POST | `/api/user/test-email` | API Key/JWT | Send test email |
| GET | `/api/user/schedules` | API Key/JWT | List digest schedules |
| POST | `/api/user/schedules` | API Key/JWT | Add digest schedule |
| DELETE | `/api/user/schedules/{id}` | API Key/JWT | Remove schedule |
| POST | `/api/user/schedules/timezone` | API Key/JWT | Set timezone |
| GET | `/api/user/api-keys` | API Key/JWT | List API keys |
| POST | `/api/user/api-keys` | API Key/JWT | Create API key |
| POST | `/api/user/api-keys/{id}/revoke` | API Key/JWT | Revoke API key |
| GET | `/api/user/page-prefs` | API Key/JWT | Get page preferences |
| POST | `/api/user/page-prefs` | API Key/JWT | Save page preferences |
| POST | `/api/user/account/delete` | API Key/JWT | Delete account |

## Messaging

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/messages/unread-count` | API Key/JWT | Unread message count |
| GET | `/api/messages/conversations` | API Key/JWT | List conversations |
| GET | `/api/messages/conversations/{id}` | API Key/JWT | Get conversation |
| POST | `/api/messages/conversations/{id}` | API Key/JWT | Send message |
| POST | `/api/messages/conversations/{id}/read` | API Key/JWT | Mark as read |
| POST | `/api/messages/conversations/start` | API Key/JWT | Start new conversation |
| POST | `/api/messages/page` | API Key/JWT | Create page/alert message |
| POST | `/api/messages/page/{id}/respond` | API Key/JWT | Respond to page |
| GET | `/api/messages/users` | API Key/JWT | List users for messaging |

## Push Notifications

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/push/register` | API Key/JWT | Register device for push |
| DELETE | `/api/push/register` | API Key/JWT | Unregister device |

## Drills

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/drills` | API Key/JWT | List drills |
| POST | `/api/drills/trigger` | API Key/JWT | Trigger new drill |
| POST | `/api/drills/{id}/accept` | API Key/JWT | Accept drill |
| POST | `/api/drills/{id}/close` | API Key/JWT | Close drill |
| GET | `/api/drills/config` | API Key/JWT | Drill configuration |
| POST | `/api/drills/config` | API Key/JWT | Save drill config |
| GET | `/drills/api/active` | Session/JWT | Check for active drill (nav badge) |

## History (Config Changes)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/history` | API Key/JWT | List config change history |
| GET | `/api/history/{id}` | API Key/JWT | Change detail with diff |
| POST | `/api/history/{id}/revert` | API Key/JWT | Revert a change |

## Admin (Server Settings)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/api/admin/retention-presets` | API Key/JWT | Log retention presets |
| POST | `/api/admin/retention-presets` | API Key/JWT | Save retention preset |
| DELETE | `/api/admin/retention-presets/{name}` | API Key/JWT | Delete preset |
| GET | `/api/admin/storage` | API Key/JWT | Storage statistics |
| POST | `/api/admin/storage/delete` | API Key/JWT | Delete storage data |
| GET | `/api/admin/notifications` | `STATUS_ADMIN`/`INFRA_ADMIN` | Notification config, secrets masked |
| POST | `/api/admin/notifications/{section}` | `STATUS_ADMIN`/`INFRA_ADMIN` | Save one section (`telegram`, `email`, …) |
| POST | `/api/admin/notifications/{section}/toggle` | `STATUS_ADMIN`/`INFRA_ADMIN` | Enable or disable that channel |
| POST | `/api/admin/notifications/{section}/test` | `STATUS_ADMIN`/`INFRA_ADMIN` | Send a test through it |

⚠️ `/api/admin/notifications/save`, `/toggle`, `/reload` and `/telegram/set-webhook` **do not
exist** — earlier revisions of this file listed them. The section is a path variable.

⚠️ Everything under `/api/admin/**` needs `STATUS_ADMIN` or `INFRA_ADMIN`, and `/api/admin/users/**`
needs `STATUS_ADMIN`, as a prefix rule in the security chain rather than a per-handler check.
"Session" in older rows here meant "any authenticated caller", which is no longer true of any of
them.

## Admin (Users & Agents, Session-based)

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/admin/users` | Session | STATUS_ADMIN |
| POST | `/admin/users/{userId}/roles` | Session | STATUS_ADMIN |
| POST | `/admin/users/{userId}/delete` | Session | STATUS_ADMIN |
| POST | `/admin/reload` | Session | INFRA_ADMIN |
| POST | `/admin/agents/{name}/approve` | Session | INFRA_ADMIN |
| POST | `/admin/agents/{name}/reject` | Session | INFRA_ADMIN |
| POST | `/admin/agents/{name}/logs` | Session | INFRA_ADMIN |
| POST | `/admin/agents/{name}/stream-toggle` | Session | INFRA_ADMIN |
| POST | `/admin/agents/{name}/add-log-source` | Session | INFRA_ADMIN |
| POST | `/admin/agents/{name}/retention` | Session | INFRA_ADMIN |
| POST | `/admin/agents/{name}/docker/{action}` | Session | INFRA_ADMIN |
| POST | `/admin/agents/{name}/run-script` | Session | INFRA_ADMIN |
| GET | `/admin/agents/{name}/commands` | Session | INFRA_ADMIN |
| GET | `/admin/agents/{name}/execution/{execId}` | Session | INFRA_ADMIN |

## Infrastructure Config

| Method | Path | Auth | Role |
|--------|------|------|------|
| GET | `/admin/infrastructure/api/config` | Session | INFRA_ADMIN |
| GET | `/admin/infrastructure/api/types` | Session | INFRA_ADMIN |
| GET | `/admin/infrastructure/api/hosts` | Session | INFRA_ADMIN |
| POST | `/admin/infrastructure/api/config` | Session | INFRA_ADMIN |

## Telegram

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/api/telegram/webhook` | Public | Incoming Telegram bot updates |

## SPA & Auth

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/app/**` | Public | SPA static files (auth handled client-side) |
| POST | `/login` | Public | Keycloak login form submission |
| GET | `/login/totp` | Public | TOTP verification page |
| GET | `/register` | Public | Registration page |
