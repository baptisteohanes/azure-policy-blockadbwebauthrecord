# Deny Azure Databricks Web Auth Private DNS Registration

Two complementary Azure Policy definitions that prevent the Azure Databricks `browser_authentication` (Web Auth) private endpoint from being registered in the `privatelink.azuredatabricks.net` private DNS zone.

## Why

When you deploy a Databricks workspace with Private Link, the workspace exposes two private endpoint sub-resources:

| groupId | Purpose | Must be in private DNS zone? |
|---|---|---|
| `databricks_ui_api` | Workspace UI + REST API + back-end | **Yes** — register in `privatelink.azuredatabricks.net` |
| `browser_authentication` | Entra ID SSO callback for the UI (`*.auth.azuredatabricks.net`) | **No** — must resolve via public DNS |

If the `browser_authentication` PE is registered in the private DNS zone, the SSO redirect resolves to a private IP that the user's browser cannot reach, and **users can no longer sign in to the workspace UI**. Microsoft's Private Link reference architecture explicitly calls this out.

These policies guardrail against that misconfiguration.

## Two approaches

The repo ships two policies that target different layers of the deployment chain. They are designed to work **together** as defense-in-depth — assign both for full coverage.

### Approach 1 — Block at the private endpoint / DNS zone group layer

**File:** [policy.json](policy.json)

Inspects `Microsoft.Network/privateEndpoints` and `Microsoft.Network/privateEndpoints/privateDnsZoneGroups` resources at deployment time.

The policy's `if` block fires on either of:

1. A `Microsoft.Network/privateEndpoints` resource whose `privateLinkServiceConnections` target a `Microsoft.Databricks/workspaces` resource with `groupIds` containing `browser_authentication`.
2. A standalone `Microsoft.Network/privateEndpoints/privateDnsZoneGroups` deployment referencing the `privatelink.azuredatabricks.net` zone where the parent PE name contains `browser` or `auth` (name heuristic — see Caveats).

**Pros:** stops the issue at the source (typical IaC deployment path); blocks before any DNS record is written.

**Cons:** case 2 relies on naming conventions because policy aliases on a child DNS zone group cannot traverse back to the parent PE's `groupIds`.

### Approach 2 — Block at the private DNS zone layer

**File:** [policy-dnszone.json](policy-dnszone.json)

Inspects `Microsoft.Network/privateDnsZones/A` records.

Denies creating any A record whose name contains `pl-auth` inside `privatelink.azuredatabricks.net`. Both the zone name and the forbidden token are parameterized.

**Pros:** catches the misconfiguration regardless of how the record was created — IaC, manual portal action, scripts, drift, or a PE deployed under an unexpected name. Independent of PE naming conventions.

**Cons:** record-name pattern (`pl-auth`) must match what your platform actually emits. Databricks-managed PE deployments use this token; if you create PEs manually with arbitrary record names, adjust the `recordNameToken` parameter.

### When to use which

| Scenario | Approach 1 | Approach 2 |
|---|---|---|
| Standard IaC (Bicep, Terraform) deploys PE + DNS zone group together | yes | yes |
| Someone creates an A record manually in the portal | no | yes |
| PEs use non-standard names (no `browser`/`auth` token) | adjust tokens | yes |
| You want a single low-maintenance deny point | — | yes (recommended) |
| You want to block the PE itself, not just the DNS record | yes | no |

**Recommended:** assign both. Approach 2 is the lower-maintenance backstop; approach 1 gives an earlier and clearer failure to IaC pipelines.

## Parameters

### policy.json

| Name | Type | Allowed values | Default |
|---|---|---|---|
| `effect` | String | `Deny`, `Audit`, `Disabled` | `Deny` |

### policy-dnszone.json

| Name | Type | Allowed values | Default |
|---|---|---|---|
| `effect` | String | `Deny`, `Audit`, `Disabled` | `Deny` |
| `recordNameToken` | String | any substring | `pl-auth` |
| `zoneName` | String | any DNS zone name | `privatelink.azuredatabricks.net` |

## Deploy

```bash
# Approach 1 — PE / DNS zone group layer
az policy definition create \
  --name deny-adb-webauth-dns \
  --display-name "Deny private DNS registration for Azure Databricks browser_authentication private endpoint" \
  --rules policy.json \
  --mode All

az policy assignment create \
  --name deny-adb-webauth-dns \
  --policy deny-adb-webauth-dns \
  --scope "/subscriptions/<subscriptionId>"

# Approach 2 — private DNS zone A-record layer
az policy definition create \
  --name deny-adb-webauth-dns-record \
  --display-name "Deny pl-auth A records in privatelink.azuredatabricks.net private DNS zone" \
  --rules policy-dnszone.json \
  --mode All

az policy assignment create \
  --name deny-adb-webauth-dns-record \
  --policy deny-adb-webauth-dns-record \
  --scope "/subscriptions/<subscriptionId>"
```

To dry-run before enforcing, assign with `--params '{"effect":{"value":"Audit"}}'`.

## Caveats

- **Name heuristic in approach 1, case 2.** Azure Policy aliases on a child `privateDnsZoneGroups` resource cannot traverse to the parent private endpoint's `groupIds`. The policy therefore matches on the PE name containing `browser` or `auth`. If your naming convention differs (e.g. `pe-adb-webauth-*`, `pe-adb-bauth-*`), adjust the `contains` tokens in [policy.json](policy.json).
- **Inline vs. standalone.** Most IaC (Bicep / Terraform AzureRM ≥ 3.x) deploys the DNS zone group as a separate child resource — approach 1 case 2 covers this. ARM templates embedding the DNS zone group in the PE body fall under approach 1 case 1.
- **Token in approach 2.** `pl-auth` matches the record name pattern emitted by Databricks-managed PE creation. Override `recordNameToken` if your environment uses a different pattern.
- **Existing non-compliant resources** are not modified. Switch the effect to `Audit` and review the compliance report to find them.

## Test

Try to deploy a Databricks `browser_authentication` PE with a DNS zone group pointing at `privatelink.azuredatabricks.net` (approach 1), or manually create an A record `pl-auth-xxxx` in that zone (approach 2). Both deployments should be rejected with a policy violation referencing the corresponding definition.
