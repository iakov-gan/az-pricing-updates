# Permissions for Azure Pricing Impact Report

## Quick answer

**Running the report on files already provided to you requires no Azure
permissions or login**, including price protection and RI forecasting. You
need local read access to the input files and write access to the output
folder and optional cache folder.

Azure access is needed only to download the input files yourself or to use
`--business-entity-method`. These are separate requirements: the person
exporting the files does not have to be the person running the report.

The report does not modify Azure resources, billing settings, reservations or
Savings Plans. No Owner, Contributor, Microsoft Entra Global Administrator or
Microsoft Graph permissions are required to run it.

## Downloading inputs

Use the following read-only roles for the billing scope covered by the report.
These are suitable built-in roles for the full workflow, not a specification
of the smallest possible custom role.

| Input | Enterprise Agreement (EA) | Microsoft Customer Agreement (MCA) |
| --- | --- | --- |
| Full calendar month of amortized usage, CSV or Parquet | Enterprise Administrator (read only) on the relevant EA enrollment | Billing profile reader on the relevant billing profile, or Billing account reader inherited by that profile |
| Contract pricesheet, required for price protection and RI forecasting | Enterprise Administrator (read only) on the relevant EA enrollment | Billing profile reader on the relevant billing profile, or Billing account reader inherited by that profile |
| Reservation inventory CSV, required for RI forecasting | Enterprise Administrator (read only) can view reservations paid for by the enrollment | Billing profile reader can view reservations paid for by the profile |

Find usage and pricesheets under **Cost Management + Billing > Billing scopes**
in the Azure portal. Select the EA billing account or MCA billing profile.
For reservations, use that billing scope's **Reservations + Hybrid Benefit**
view and export the inventory. See the [README prerequisites](README.md#prerequisites)
for input formats and download locations.

Reservation-specific read access is an alternative for obtaining inventory:
**Reservations Reader** at tenant scope, or **Reader** on the relevant
reservation resources. Use **Home > Reservations** for reservation RBAC access.
It does not grant access to usage exports or contract pricesheets.

**Subscription Reader alone is not enough for this end-to-end export
workflow.** Billing roles are separate from resource RBAC, and reservations
do not inherit access from subscriptions. Ensure the exported inventory
includes every reservation referenced by the usage being forecast.

## Running all file-based features

After obtaining the inputs, this command runs price protection and RI
forecasting without contacting Azure:

```powershell
python -m pricing_updates.impact_report ".\inputs\usage.parquet" --output ".\reports\impact.xlsx" --pricesheet ".\inputs\prices.parquet" --price-protection --reservation-inventory ".\inputs\reservedInstances.csv" --ri-price-evolution --cache-dir ".\cache"
```

No Azure CLI, `az login`, app registration, client secret or tenant consent is
needed for this command. The usage and pricesheet must have matching EA/MCA
contract types. Installation and Python prerequisites are in the
[README](README.md#installation).

## Optional live business-entity lookups

`--business-entity-method` is the only report option that contacts Azure.
The tool uses `AzureCliCredential`, so install Azure CLI and sign in to the
tenant containing the billing account or subscriptions:

```powershell
az login --tenant "<tenant-id>"
```

For an identity with billing access but no Azure subscriptions, use:

```powershell
az login --tenant "<tenant-id>" --allow-no-subscriptions
```

Choose one business-entity method per run. It can be appended to the
file-based command above.

### EA department

```powershell
--business-entity-method ea_department_name
```

Use **Enterprise Administrator (read only)** on each relevant EA enrollment
for enrollment-wide access. Subscription Reader or an MCA billing role is
not a substitute for EA billing access.

The implementation makes read-only Azure Billing API requests to:

- List accessible billing accounts and select Enterprise Agreements.
- List enrollment accounts and their department display names.
- List billing subscriptions and join them to enrollment accounts.

This maps subscriptions to their **current** EA department, not necessarily
the department they belonged to during the exported usage month. It does not
provide an MCA department lookup.

### Management-group hierarchy

```powershell
--business-entity-method management_group_hierarchy_level_1
```

Use **Reader** on every subscription included in the report, either assigned
directly or inherited from a parent management group. A Reader assignment on
a common parent management group is suitable when all its descendant
subscriptions must be covered. Billing roles alone do not grant this access.

The implementation queries subscription containers and their management-group
ancestor chains through Azure Resource Graph. Level `0` selects the tenant
root group; levels `1` through `6` select progressively lower groups.

Resource Graph returns only subscriptions the signed-in identity can read.
A successful query does not prove that every subscription in the input was
visible. Missing mappings, including subscriptions without the selected
hierarchy level, appear as `Unknown`.

## Access troubleshooting and data handling

| Symptom | What to check |
| --- | --- |
| Usage or pricesheet cannot be downloaded | Correct EA enrollment or MCA billing profile, and a billing role covering that scope |
| Reservations are missing from the export | Reservation visibility and the selected enrollment/profile; subscription access alone does not guarantee reservation access |
| Live lookup fails with an authentication or authorization error | Azure CLI sign-in, tenant, and the billing or resource role required by the selected method |
| BusinessEntity is `Unknown` | Coverage of subscription permissions, selected management-group level, and whether the subscription has a current EA department mapping |
| Workbook or cache cannot be written | Local folder permissions and whether the output workbook is open in Excel |

Usage, pricesheets and reservation inventory are processed locally and are
not uploaded by the report. Live lookups read Azure metadata only. The
workbook and optional parsed-input cache contain billing data; store them
in locations appropriate for the source files' access restrictions.

## Microsoft documentation

- [EA administrative roles and permissions](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/understand-ea-roles)
- [MCA billing roles and permissions](https://learn.microsoft.com/en-us/azure/cost-management-billing/manage/understand-mca-roles)
- [Permissions to view Azure reservations](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/view-reservations)
- [Azure Resource Graph permissions](https://learn.microsoft.com/en-us/azure/governance/resource-graph/overview#permissions-in-azure-resource-graph)
