---
name: uipath-admin
description: "UiPath Admin via `uip admin` — Identity Server (users, groups, robot accounts, external OAuth2 apps, secrets), Authorization (custom roles, role assignments, permission catalog, effective-access via check-access PDP), OMS (org read/update, tenant lifecycle, service provisioning, regions, async operation polling), APMS (IP allowlist, enforcement, bypass rules, lockout safety), Audit (event sources, paginated queries, ZIP exports — login history, compliance dumps, who-did-what-when-where on a resource). For Orchestrator-specific roles/permissions/folders/jobs→uipath-platform. For RPA workflows→uipath-rpa."
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion
---

# UiPath Admin

> **Preview** — Under active development. Command coverage will expand.

Administrative operations on UiPath via `uip admin` — Identity Server (users, groups, robot accounts, external OAuth2 apps), Authorization (custom roles, assignments, permission catalog, `check-access` PDP), OMS (org read/update, tenant lifecycle, service provisioning), APMS (IP allowlist + enforcement), Audit (event queries + ZIP exports). Per-area detail lives in [Task Navigation](#task-navigation) — this file is the entry contract.

## When to Use This Skill

### Identity

- **Manage identity users** — list, create, invite, update, delete
- **Manage groups** — CRUD + add/remove members
- **Manage robot accounts** — create, update, delete unattended robot identities
- **Manage external apps** — OAuth2 clients, generate/rotate secrets
- **Onboard human user** — invite, assign to groups
- **Onboard robot account** — create account, assign to groups
- **Identity concepts** — partitions, organizations, OAuth2 scopes
- **Generate Client ID/Secret** — credentials for API or robot authentication

### Authz

- **Manage custom roles** — CRUD on Authorization service role definitions (scope shapes: `Organization`, `TenantGlobal`, `Tenant`, `Project`)
- **Manage role assignments** — assign roles to users/groups/robot accounts at `Organization`, `Tenant`, `TenantGlobal`, `Project`, `Folder`, or `App` scope
- **List permission definitions** — read-only catalog of permissions across services
- **Check effective access** — compute what a principal can actually do at a given scope (Policy Decision Point)
- **Grant permission(s) to a principal** — ad-hoc "grant me X" / "give <user> Y, Z" requests resolved via the scope/service intersection flow

### OMS

- **Inspect / update the current organization** — `uip admin organizations` (read + update only; no CLI create/delete)
- **Manage tenant lifecycle** — create, enable, disable, delete tenants in the caller's org
- **Provision org-level or tenant-level services** — `services list`, `list-available`, `add`, `enable`, `disable`, `remove`
- **Poll async OMS operations** — `tenants` mutations return `operationId`; poll via `organizations operation get <id>` (the canonical poll endpoint)
- **List available regions** — discover provisioning regions before `tenants create`

### APMS (IP restriction)

- **Manage IP allowlisting** — add / update / delete CIDR entries that gate inbound access
- **Toggle IP-restriction enforcement** — turn the org-wide allowlist switch on or off (with lockout safety)
- **Manage bypass rules** — URL-pattern exceptions to IP allowlisting
- **Look up the caller's public IP** — sanity check before enabling enforcement

### Audit

- **Query audit events** — list event sources, filter events by source / target / type / user / status / time window at org or tenant scope
- **Export audit events** — chunked ZIP download from the long-term store, per UTC day, with atomic abort on any chunk failure
- **Login history** — investigate sign-in events ("who's been signing in", "failed/successful logins") — org-scoped via `audit org events`

## Critical Rules

Each rule is the agent contract. Per-area detail is in the linked reference files.

### Universal

1. **Route Orchestrator-specific role/permission requests to `uip or roles`** (`uipath-platform` skill). `uip admin authorization` does NOT own Orchestrator's role catalog. When in doubt — user said "role" with no service named — assume `uip admin authorization`.
2. **Verify login first.** `uip login status --output json`. If not logged in: `uip login`. Org id is resolved from the active session.
3. **Use `--output json` on every command.** Parse programmatically; present conversationally.
4. **Stop on error.** Show the error verbatim. Never retry auth failures — ask the user to `uip login`.
5. **Resolve every named principal before high-risk ops.** Any command that touches a named user / group / robot account / external app — `roles assignments create/delete`, `users delete`, `groups delete`, `groups members add/revoke`, `robot-accounts delete`, `external-apps delete`, `external-apps generate-secret` — MUST first search the directory and echo `Principal: <displayName> (<userName>) — <id>` back before the mutation runs. Search via `uip admin users|groups|robot-accounts|external-apps list --search "<NAME>"`. **Zero matches → stop and ask; never fall back to the current login user. Multiple matches → numbered list, wait for a digit.** Wrong-principal grants are security incidents.

### Identity

6. **Discover before creating.** `list` before `create` to avoid duplicates (robot accounts, groups, external apps — `users invite` excepted).
7. **Secrets shown only once** on external-app create and `generate-secret` — warn the user to save immediately.
8. **External apps require scopes at creation** — `--scope` is required (e.g., `OR.Folders`, `OR.Assets`, `OR.Queues`, `OR.Jobs`, `OR.Machines`).
9. **Group membership uses user IDs.** Resolve via `users list` per Rule 5, then `groups members add/revoke`.
10. **Confirm before delete** on users / groups / robot accounts / external apps — after resolving the named target per Rule 5.

### Authz

11. **Built-in roles are read-only.** Only `Custom` roles can be created / updated / deleted. The CLI also rejects authoring against service-managed services (`orchestrator`, `dataservice`, `insights`, `taskmining`, `testmanager`, `automationops`, `casemanagement`, `processmining`) and platform-level (`authz`, `oms`, `platform`, `identity`, `licensing`).
12. **`roles create` / `roles update` are PUT-style upserts.** Body is assembled from inline flags + `--file ./actions.json`. Always `roles get` first before updating — omitted flags overwrite that field.
13. **`--service` infers scope** (e.g., `--service studio` → `Tenant`; `--service apps` → `Organization`). Combine with `--scope` only to override (e.g., `--service documentunderstanding --scope Project`).
14. **Listing works for every service; authoring is what's blocked.** `roles list --service <svc>` and `roles assignments list --service <svc>` accept every service. For effective access on a principal use `check-access` (PDP).
15. **Scope vocab differs across verbs.** `roles create --scope`: `Organization|TenantGlobal|Tenant|Project`. `roles assignments create --scope`: those + `Folder|App`. `roles assignments list --scope`: excludes `TenantGlobal`. `check-access --scope`: only `Tenant|Folder` (loop tenants for org-wide; folder-scope for project-level).
16. **`roles assignments create/delete` MUST resolve the principal first** per Rule 5 — `--identity-id` is a raw UUID the CLI does not name-check.
17. **`roles assignments create` MUST match the role's `ownerServiceName` to the scope-path service segment.** Before the call, `roles get <ROLE_ID>` → read `ownerServiceName`: (a) `CentralizedAccess` → path must NOT include a service segment (use `/` or `/tenant/<tid>`); (b) anything else → path MUST include `lowercase(ownerServiceName)` (e.g., `/tenant/<tid>/reinfer/project/<pid>`). Mismatch → stop and surface; never silently substitute a different service. Procedure: [role-assignment-management.md — Validate Role's Owning Service](references/authorization/role-assignment-management.md#validate-roles-owning-service-vs-assignment-scope-path).

### OMS

18. **Async lifecycle: auto-poll, then hand off.** `tenants create/update/delete/enable/disable` return `operationId`. Auto-poll `organizations operation get <OP_ID>` 3× at 5 s; on terminal status (`Succeeded` / `Failed` / `Cancelled`) stop and report; still in-progress after 3 polls → numbered menu, never indefinite loop. Procedure: [organization-management.md — Polling procedure](references/organization-management.md#polling-procedure-auto-poll-then-hand-off). **`organizations create` and `organizations delete` are not exposed by the CLI** — those go through the UiPath Portal / support flow.
19. **`tenants delete` is soft-only.** No hard-delete flag; restoration is via support.
20. **Tenant commands default to the login tenant.** Always pass an explicit `<TENANT_ID>` for destructive ops (`tenants delete`, `tenants disable`, `tenants services remove`).
21. **Resolve region before tenant create.** `--region` is required on `tenants create` — run `organizations regions list` first. Tenant service catalog is region-aware.
22. **`services disable` / `remove` may no-op despite Success** on certain services. Always re-list after mutating. Gap list: [tenants-commands.md — Concepts](references/tenants-commands.md#concepts).

### APMS

23. **IP restriction is lockout-sensitive.** `ip-ranges delete` and `enforcement enable` require `--confirm`. Before `enforcement enable`: run `ip-restriction my-ip`, then verify the IP is covered by an entry in `ip-ranges list`.
24. **Recovery from IP lockout requires platform-side action.** No CLI bypass — either access from an in-allowlist IP and `enforcement disable`, or use the Portal recovery flow.

## What NOT to Do

1. **Never delete built-in groups** (`type: "BuiltIn"`).
2. **Never pass IDs as flags.** Resource IDs are positional: `groups members add <GROUP_ID> --user-ids ...`, NOT `--group-id <GROUP_ID>`.
3. **Do NOT assume audit `events` returns a bare array.** Shape is `{auditEvents, next, previous}`.
4. **Do NOT loop on `--from-date`/`--to-date` to "paginate".** Bump `--limit` — the CLI handles cursor pagination internally.
5. **Do NOT silently default audit scope.** Ask once when ambiguous, then proceed.
6. **Do NOT invent audit source / target / type GUIDs.** Always discover via `sources` first.
7. **Do NOT call audit `events` with no time bound** on a noisy tenant — default to a bounded window.
8. **Do NOT pass `--tenant-id` to `org`-scoped audit commands** — silently ignored.
9. **Do NOT retry on 401.** The token is missing `Audit.Read`; `uip logout && uip login`.
10. **Do NOT call `roles update` with only the flag you want to change.** Re-fetch first; the upsert body overwrites omitted fields (Rule 12).
11. **Do NOT present authz results without provenance** — role name, `scopeType`, `ownerServiceName`, tenant-binding (names not UUIDs). Detail: [authorization-commands.md — Provenance contract](references/authorization/authorization-commands.md#provenance-contract-for-completion-output).
12. **Do NOT conflate provisioned services with the available catalog.** `services list` returns provisioned with status; `services list-available` is the catalog. Present them as separate sections.
13. **Do NOT run an OMS mutation without naming the target.** Echo org name / tenant name + UUID / service type + region before running. Detail: [organization-management.md](references/organization-management.md#before-mutating-the-organization--name-the-target) · [tenant-management.md](references/tenant-management.md#before-mutating-a-tenant-or-tenant-service--name-the-target).

## Quick Start

One row per common goal. Per-area workflows are in the reference files.

| Goal | Entry command(s) |
|---|---|
| **Invite a user → assign to group** | `uip admin users invite --email <E> --name <F> --surname <L> --output json` → `uip admin users list --search <E>` → `uip admin groups members add <GROUP_ID> --user-ids <USER_ID>` |
| **Create a custom role** | `uip admin authorization roles create --scope <Organization\|TenantGlobal\|Tenant\|Project> --name "<NAME>" --file ./actions.json --output json` (actions.json = `["STUDIO.X.Y", ...]`) |
| **Grant permission(s) to a principal** ("grant me X", "give alice Y, Z") | Run the intersection-and-menu flow: [grant-permissions.md](references/authorization/grant-permissions.md) |
| **Assign a role to a principal** | (1) Resolve principal per Rule 5. (2) `roles get <ROLE_ID>` → echo `ownerServiceName` + verify scope-path service segment matches (Rule 17). (3) `roles assignments create --role-id <ROLE_ID> --identity-id <ID> --identity-type <User\|Group\|Robot\|ExternalApplication> --output json` |
| **See what a principal can do** | `uip admin authorization check-access <USER_GUID_OR_EMAIL> --scope <Tenant\|Folder> --output json` (no `Organization` / `Project` — loop tenants or use Folder scope; Rule 15) |
| **Create a tenant** | `organizations regions list` → `tenants create --name <N> --region <R>` → poll `organizations operation get <OP_ID>` (Rule 18) |
| **Add a tenant service** | `tenants services list-available --region <R>` → `tenants services add --tenant-id <TID> --service <SVC>` (verify post-state per Rule 22) |
| **Enable IP allowlist enforcement** | `ip-restriction my-ip` → verify covered by `ip-ranges list` → `ip-restriction enforcement enable --confirm` (Rule 23) |
| **Query audit events / export** | Drive from [audit-workflow-guide.md](references/audit-workflow-guide.md); start with `audit <org\|tenant> sources`, then `events`, then `export`. |

## Key Concepts

### Organization hierarchy

```
Organization (org)
  └── Partition (= org in most cases)
        ├── Users           ← human identities
        ├── Groups          ← role containers (BuiltIn + Custom)
        ├── Robot Accounts  ← unattended automation identities
        └── External Apps   ← OAuth2 clients (Client ID + Secret)
```

### Robot accounts vs external apps

| Concept | Purpose | Managed by |
|---|---|---|
| **Robot account** | Identity — who the robot is | Identity Server (`uip admin`) |
| **Robot credentials** | Per-robot Client ID + Secret for machine auth | Orchestrator (machine connection) |
| **External app** | OAuth2 client for API integrations, CI/CD | Identity Server (`uip admin`) |

Robot credentials are provisioned automatically by Orchestrator on machine connect — not by creating external apps.

### Authz vs Orchestrator roles

`uip admin authorization` **authors** cross-service / platform custom roles only. Services that manage their own catalogs server-side (`orchestrator`, `dataservice`, `insights`, `taskmining`, `testmanager`, `automationops`, `casemanagement`, `processmining`) reject `roles create/update/delete` — mutate via the service's own surface (e.g., `uip or roles` for Orchestrator). Their existing roles **do** surface in `roles list --service <svc>` and `roles assignments list --service <svc>`. For effective access use `check-access` (PDP).

### OMS sync vs async

| Operation | Sync/Async | Confirm completion |
|---|---|---|
| `organizations get`, `update` | **Sync** | Response. No CLI `create` / `delete` — Portal / support flow only. |
| `tenants create / update / delete / enable / disable` | **Async** — returns `operationId` | Poll `organizations operation get <OP_ID>` (single endpoint for all OMS async ops) |
| `tenants services add / enable / disable / remove` | **Sync** | Response — except `disable` (Integration Service, Data Fabric, Insights) and `remove` (Orchestrator, Maestro, Integration Service, Data Fabric, Insights, Test Manager), which return Success but no-op. Re-list. |
| All `*list*`, `get`, `regions list`, `services list-available` | **Sync** | — |

### Audit scope → basePath

`org` → `/{orgId}/orgaudit_/...` · `tenant` → `/{orgId}/{tenantId}/tenantaudit_/...`. Same `QueryApi`; the only difference is the URL segment. Scope-disambiguation table + `Data` shape per verb: [audit-workflow-guide.md](references/audit-workflow-guide.md).

## Output Etiquette

What to surface after each verb. Per-area detail in the reference files; this is the contract.

| Area | Always surface |
|---|---|
| **Identity** mutations | Result + new resource id; for external-app create / `generate-secret`, **highlight the secret + warn to save**; offer a next step (assign to group, generate another secret, etc.). |
| **Authz** reads + mutations | Provenance: role name, `scopeType`, `ownerServiceName` (read directly from response), tenant binding (resolve UUID → name). Group `permissions list` by `serviceDisplayName`. Group `check-access` results **by service**. After a mutation, re-fetch and apply the provenance contract. Full table: [authorization-commands.md — Provenance contract](references/authorization/authorization-commands.md#provenance-contract-for-completion-output). |
| **OMS** reads | Separate **provisioned** (with status) from **available catalog** (no status). Lead with `Organization: <ORG_NAME>` (and tenant name + UUID + lifecycle status for tenant reads). |
| **OMS** mutations | Echo the resolved target before running (Anti-pattern 13). Async: auto-poll 3× at 5 s, then numbered menu (Rule 18). Sync services: re-list to verify post-state (Rule 22). |
| **APMS** mutations | After `enforcement enable`: confirm caller's IP is still covered (re-run `my-ip` + `ip-ranges list`). Offer next step (add a bypass rule, list ranges). |
| **Audit** queries | Operation summary (count, scope, time window, filters, cursor state). Wait for the user's next-step choice; do not chain mutations. |

## Task Navigation

| I need to... | Read first |
|---|---|
| Identity CLI reference | [references/identity-commands.md](references/identity-commands.md) |
| Manage users (list / create / invite / update / delete) | [references/user-management.md](references/user-management.md) |
| Manage groups (CRUD + membership) | [references/group-management.md](references/group-management.md) |
| Manage robot accounts | [references/robot-account-management.md](references/robot-account-management.md) |
| Manage external apps (OAuth2 + secrets) | [references/external-app-management.md](references/external-app-management.md) |
| Authorization CLI reference | [references/authorization/authorization-commands.md](references/authorization/authorization-commands.md) |
| Manage custom roles | [references/authorization/role-management.md](references/authorization/role-management.md) |
| Grant permission(s) to a principal — scope/service intersection flow | [references/authorization/grant-permissions.md](references/authorization/grant-permissions.md) |
| Manage role assignments (incl. role-service vs scope-path validation, Rule 17) | [references/authorization/role-assignment-management.md](references/authorization/role-assignment-management.md) |
| List permission definitions | [references/authorization/permission-catalog.md](references/authorization/permission-catalog.md) |
| Check effective access for a principal | [references/authorization/check-access.md](references/authorization/check-access.md) |
| Organizations CLI reference | [references/organizations-commands.md](references/organizations-commands.md) |
| Tenants CLI reference | [references/tenants-commands.md](references/tenants-commands.md) |
| Manage the organization (read + update, polling, regions, org services read-only) | [references/organization-management.md](references/organization-management.md) |
| Manage tenants (CRUD, enable/disable, tenant services) | [references/tenant-management.md](references/tenant-management.md) |
| IP-restriction CLI reference | [references/ip-restriction/ip-restriction-commands.md](references/ip-restriction/ip-restriction-commands.md) |
| Manage IP allowlist entries | [references/ip-restriction/ip-range-management.md](references/ip-restriction/ip-range-management.md) |
| Toggle enforcement (+ `my-ip` safety check) | [references/ip-restriction/enforcement-management.md](references/ip-restriction/enforcement-management.md) |
| Manage bypass rules | [references/ip-restriction/bypass-rule-management.md](references/ip-restriction/bypass-rule-management.md) |
| Audit CLI reference | [references/audit-commands.md](references/audit-commands.md) |
| Audit investigation workflows | [references/audit-workflow-guide.md](references/audit-workflow-guide.md) |
