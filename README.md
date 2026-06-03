# Deny Azure Databricks Web Auth Private DNS Registration

Azure Policy that denies registering the Azure Databricks `browser_authentication` (Web Auth) private endpoint into the `privatelink.azuredatabricks.net` private DNS zone.

## Why

When you deploy a Databricks workspace with Private Link, the workspace exposes two private endpoint sub-resources:

| groupId | Purpose | Must be in private DNS zone? |
|---|---|---|
| `databricks_ui_api` | Workspace UI + REST API + back-end | **Yes** — register in `privatelink.azuredatabricks.net` |
| `browser_authentication` | Entra ID SSO callback for the UI (`*.auth.azuredatabricks.net`) | **No** — must resolve via public DNS |

If the `browser_authentication` PE is registered in the private DNS zone, the SSO redirect resolves to a private IP that the user's browser cannot reach, and **users can no longer sign in to the workspace UI**. Microsoft's Private Link reference architecture explicitly calls this out.

This policy guardrails against that misconfiguration.

## What it denies

The policy's `if` block fires on either of:

1. A `Microsoft.Network/privateEndpoints` resource whose `privateLinkServiceConnections` target a `Microsoft.Databricks/workspaces` resource with `groupIds` containing `browser_authentication`, **and** that has an inline `privateDnsZoneGroups` child registering it.
2. A standalone `Microsoft.Network/privateEndpoints/privateDnsZoneGroups` deployment that references the `privatelink.azuredatabricks.net` zone where the parent PE name contains `browser` or `auth` (name heuristic — see Caveats).

## Files

- [policy.json](policy.json) — policy definition (mode `All`, default effect `Deny`).

## Parameters

| Name | Type | Allowed values | Default |
|---|---|---|---|
| `effect` | String | `Deny`, `Audit`, `Disabled` | `Deny` |

## Deploy

```bash
# Create the definition at subscription scope
az policy definition create \
  --name deny-adb-webauth-dns \
  --display-name "Deny private DNS registration for Azure Databricks browser_authentication private endpoint" \
  --rules policy.json \
  --mode All

# Assign it (subscription scope shown; use --management-group for MG scope)
az policy assignment create \
  --name deny-adb-webauth-dns \
  --policy deny-adb-webauth-dns \
  --scope "/subscriptions/<subscriptionId>"
```

To dry-run before enforcing, assign with `--params '{"effect":{"value":"Audit"}}'`.

## Caveats

- **Name heuristic for case 2.** Azure Policy aliases on a child `privateDnsZoneGroups` resource cannot traverse to the parent private endpoint's `groupIds`. The policy therefore matches on the PE name containing `browser` or `auth`. If your naming convention differs (e.g. `pe-adb-webauth-*`, `pe-adb-bauth-*`), adjust the `contains` tokens in [policy.json](policy.json).
- **Inline vs. standalone.** Most IaC (Bicep/Terraform AzureRM ≥ 3.x) deploys the DNS zone group as a separate child resource — case 2 covers this. ARM templates that embed the DNS zone group in the PE body fall under case 1.
- **Existing non-compliant resources** are not modified. To find them, switch the effect to `Audit` and review the compliance report.

## Test

Try to deploy a Databricks `browser_authentication` PE with a DNS zone group pointing at `privatelink.azuredatabricks.net`. The deployment should be rejected with a policy violation referencing this definition.
