# Check Access (Effective Permissions)

Conceptual guide and scope-specific workflows for `uip admin authorization check-access`. For the full flag/argument table and output code, see [authorization-commands.md — Check Access](authorization-commands.md#check-access--uip-admin-authorization-check-access).

## Concept

`check-access` is the **Policy Decision Point (PDP)**. It answers: *"What can this principal actually do at this scope, right now?"*

Unlike `roles assignments list` (which reads stored assignments from the PAP catalog), `check-access` evaluates **effective** access — it folds in server-side rules from services that manage their own role catalogs (`orchestrator`, `dataservice`, `insights`, `taskmining`, `testmanager`, `automationops`, `casemanagement`, `processmining`) that the PAP catalog alone may not reflect. The catalog *does* surface these services' roles and assignments (`roles list --service <svc>` / `roles assignments list --service <svc>`), but the PDP is the source of truth for "what permissions resolve right now".

Returned `Data`:

- `roleAssignments` — paginated list of effective assignments.
- `grantedServicesMetadata` — services the principal has any access to.
- `grantedRolesMetadata` — roles contributing to the result.

## Identity Argument

The principal is the **positional** first argument — UUID, name, or email. The CLI resolves names and emails via the identity API.

```bash
uip admin authorization check-access <USER_GUID>
uip admin authorization check-access alice@example.com
uip admin authorization check-access "Alice Smith"
```

There is no `--identity-id` flag. With `--file`, the identity is set in the request body (`SecurityPrincipalId`) and the positional argument is omitted.

## Choosing the Scope

`check-access --scope` accepts only **`Tenant`** (default) or **`Folder`** — narrower than the scope vocab on `roles create` or `roles assignments create`.

| Scope | When to use | Required flags beyond identity |
|-------|-------------|--------------------------------|
| `Tenant` (default if omitted) | Per-tenant access | `--tenant-id <GUID>` optional (defaults to login tenant) |
| `Folder` | Folder-scoped access | `--scope Folder --folder-id <FOLDER_ID>`. `--tenant-id` becomes the owning tenant ParentId (defaults to login tenant) |

> **No `Organization` / `Project` scope on `check-access`.** The PDP doesn't expose those directly. To approximate:
> - **Org-wide entitlement check** — loop `check-access <USER> --tenant-id <TID>` over every tenant (`tenants list`), then aggregate `grantedRolesMetadata` per service.
> - **Project-level access** — use `--scope Folder` with the project's owning folder, or filter the default `Tenant` result to the service that owns the project (`--service documentunderstanding` etc.).

> `--folder-id` replaces the older `--scope-id` / `--parent-folder-id` pair. For Folder scope, `--folder-id` is the Folder's `Id` and `--tenant-id` is the owning tenant's `ParentId`.

## Workflow: Check a User's Effective Access at the Login Tenant

```bash
uip admin authorization check-access <USER_GUID> --output json
```

Default scope is `Tenant`, default tenant is the login tenant.

## Workflow: Check Across Tenants

```bash
uip admin authorization check-access <USER_GUID> --tenant-id <OTHER_TENANT_ID> --output json
```

## Workflow: Restrict to One Service

Especially useful for services that manage their own roles, where the PAP catalog may not reflect server-side rules:

```bash
uip admin authorization check-access <USER_GUID> --service orchestrator --output json
```

Known service names: `apps, authz, automationops, casemanagement, dataservice, documentunderstanding, identity, insights, licensing, oms, orchestrator, platform, processmining, reinfer, studio, taskmining, testmanager`. Other names are accepted free-form (passed through verbatim as `ServiceName`).

> **`--service centralizedaccess` is not in the list.** `check-access` does pass arbitrary `--service` values through verbatim, but the PDP has no separate "centralizedaccess" service to score — querying it returns no useful access info. For the multi-service / Centralized Access view, omit `--service` and let the default Tenant evaluation return every role the principal holds across services.

## Workflow: Folder Scope

```bash
uip admin authorization check-access <USER_GUID> \
  --scope Folder \
  --folder-id <FOLDER_ID> \
  --output json
```

`--tenant-id` defaults to the login tenant for the ParentId; override only when targeting a folder in a different tenant.

## Workflow: Advanced — File-Based Request

Use `--file <PATH>` for filters not exposed inline (e.g. `RoleNameStartsWith`). With `--file`, omit the positional identity and the inline scope flags — they're set in the body.

`check-access.json`:

```json
{
  "SecurityPrincipalId": "<PRINCIPAL_ID>",
  "RoleNameStartsWith": "Admin",
  "ServiceName": "orchestrator",
  "ScopeIdentifier": {
    "ScopeType": "Tenant",
    "Value": { "Id": "<TENANT_ID>", "ParentId": "<TENANT_ID>" }
  }
}
```

For Folder scope, `Value.Id` is the folder UUID and `Value.ParentId` is the owning tenant UUID.

```bash
uip admin authorization check-access --file ./check-access.json --output json
```

## Resolving Principal IDs

If you only have a UUID and want to verify the identity exists first, or if you need IDs for non-User principals, see [role-assignment-management.md — Resolving Principal IDs](role-assignment-management.md#resolving-principal-ids).
