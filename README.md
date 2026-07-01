# CoreView.ConfigurationManager.Baselines

Industry baseline definitions for Microsoft 365 configuration management. Each baseline maps industry security controls (such as CIS or Essential Eight) to concrete configuration settings across Microsoft 365 workloads.

## Repository structure

```
Definitions/
├── Content/          # Desired configuration state for individual resources
└── Tags/             # Industry baseline manifests that reference content files
```

### Content definitions (`Definitions/Content/`)

Content files hold the target configuration for a single resource instance. They are organized by provider and resource type:

```
Content/
├── MSGraph/
│   ├── Groups/
│   ├── Identity/ConditionalAccess/Policies/
│   ├── Policies/RoleManagementPolicies/Rules/
│   └── ...
├── ExchangeOnline/
├── Teams/
├── SharePoint/
└── ...
```

Each JSON file contains the properties required to configure that resource. File names and property values together identify the resource — for example, a group is identified by its `displayName`, while an Exchange policy may use `Identity`.

Content files can reference other resources and tenant context using placeholder syntax:

| Placeholder | Purpose |
|---|---|
| `${ResourceContext:TenantDomainName}` | Resolves to the tenant's primary domain at deployment time |
| `${urn:resource:Provider:ResourceType/Name?id}` | References another resource defined in the baseline by its logical identifier |

Example group definition:

```json
{
  "displayName": "Baseline - Guest Users",
  "groupTypes": ["DynamicMembership"],
  "mailEnabled": false,
  "membershipRule": "(user.userType -eq \"Guest\")",
  "securityEnabled": true
}
```

### Baseline tags (`Definitions/Tags/`)

Each file in `Definitions/Tags/` defines one industry baseline. A tag file links content resources to the security controls they satisfy.

| Property | Description |
|---|---|
| `name` | Full baseline name |
| `label` | Short display label |
| `description` | Summary of the baseline and its source framework |
| `tags` | Array of resource entries included in the baseline |

Each entry in the `tags` array has:

| Property | Required | Description |
|---|---|---|
| `path` | Yes | Relative path to a content file under `Definitions/Content/` |
| `description` | Yes | Array of control references from the industry framework (e.g. CIS recommendation IDs, ISM controls) |
| `$friendlyNameOverride` | No | Custom display name for the resource (see below) |

Example tag entry:

```json
{
  "path": "Content/ExchangeOnline/AdminAuditLogConfig/Configuration.json",
  "description": [
    "3.1.1 (L1) Ensure Microsoft 365 audit log search is Enabled (Automated)"
  ]
}
```

A single content file can appear in multiple baselines, and a single tag entry can map to multiple controls via the `description` array.

### Dependency resources

Some content files exist only to support other baseline configurations (for example, groups used as PIM approvers, or role management policies referenced by their rules). These are included in a baseline with a dependency description:

```json
{
  "path": "Content/MSGraph/Groups/Baseline - PIM Approvers.json",
  "description": [
    "This configuration is a dependency for other industry baseline configurations."
  ]
}
```

## Friendly names and `$friendlyNameOverride`

Every resource in a baseline has a **friendly name** — the human-readable label shown when browsing baseline resources. By default, the friendly name is derived from the resource's identifying properties in its content file (such as `displayName`, `Identity`, or a composite key built from multiple fields).

For most resources this works well. Some resource types — particularly those with composite identifiers — produce technical names that are hard to read. A common example is Entra ID Privileged Identity Management (PIM) role management policy rules, where the identifier combines the OData type, parent policy reference, and rule ID.

### When to use `$friendlyNameOverride`

Add `$friendlyNameOverride` to a tag entry when the automatically derived friendly name is not meaningful to operators reviewing the baseline. This is optional and only affects the display name — it does not change the resource content, its logical identifier, or how it is deployed.

### How to set it

Add the property alongside `path` and `description` in the tag entry:

```json
{
  "path": "Content/MSGraph/Policies/RoleManagementPolicies/Rules/#microsoft.graph.unifiedRoleManagementPolicyApprovalRule--${urn%3Aresource%3AMSGraph%3APolicies%3ARoleManagementPolicies%2FGlobal Administrator%3Fid}--Approval_EndUser_Assignment.json",
  "$friendlyNameOverride": "Global Administrator - Approval_EndUser_Assignment",
  "description": [
    "5.3.4 (L1) Ensure approval is required for Global Administrator role activation (Automated)"
  ]
}
```

### Naming conventions

When setting an override, use a concise, descriptive name that identifies the resource in context:

- Combine the parent resource name with the distinguishing property: `"Global Administrator - Approval_EndUser_Assignment"`
- Use the same separator style (`" - "`) consistently across related resources in a baseline
- Keep the name stable — changing it affects how the resource appears in baseline views

### Resolution order

1. If `$friendlyNameOverride` is set on the tag entry, that value is used as the friendly name.
2. Otherwise, the friendly name is derived from the resource's identifying properties in the content file.

The logical identifier of the resource is always determined by the content file and is not affected by `$friendlyNameOverride`.

## Adding or updating a baseline

1. **Create or update the content file** under `Definitions/Content/`, following the existing folder structure for the target provider and resource type.
2. **Reference it in the appropriate tag file** under `Definitions/Tags/`, adding the relevant control references to the `description` array.
3. **Set `$friendlyNameOverride`** if the resource's default friendly name would be unclear to operators.
4. **Include dependency resources** when a new configuration references other baseline resources via `${urn:resource:...}` placeholders.

## Available baselines

| File | Framework |
|---|---|
| `CIS M365 Foundations 6.0.1.json` | CIS Microsoft 365 Foundations Benchmark v6.0.1 |
| `Essential 8 Maturity Level 1.json` | ACSC Essential Eight Maturity Model — Level 1 |
| `Essential 8 Maturity Level 2.json` | ACSC Essential Eight Maturity Model — Level 2 |
| `Essential 8 Maturity Level 3.json` | ACSC Essential Eight Maturity Model — Level 3 |
