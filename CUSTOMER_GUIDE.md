# Customer guide: Azure Pricing Impact Report

Follow the five stages below in order. Allow approximately 15-30 minutes for
local setup, excluding access approvals, export generation, downloads and
report processing, which depend on your organization and data volume.

**This tool does not download billing data automatically.** You first obtain
Azure billing exports, then run a local Python command to create an Excel
workbook. Azure login is needed only for optional business-entity grouping.

The report estimates the effects of the 28 September 2026 announcement,
effective 1 February 2027. It does not change resources, purchase reservations
or apply prices to your Azure account. Results are indicative; validate them
against Cost Management and your agreement before making financial decisions.

## Dependency order

| Stage | Depends on | Ready when |
| --- | --- | --- |
| 1. Confirm scope and access | EA enrollment or MCA billing profile identified | An authorized person can export billing data |
| 2. Obtain inputs | Stage 1 | Required exports have been downloaded and extracted |
| 3. Prepare the workstation | Python and repository access; can be done while exports are prepared | Dependencies are installed and command help opens |
| 4. Generate the workbook | Stages 2 and 3; Azure login only if grouping is enabled | The command finishes and creates the workbook |
| 5. Review results | Stage 4 and Microsoft Excel | Formulas recalculate and missing/unpriced data is reviewed |

The billing-data owner can perform stages 1-2 and securely provide the files
to a different person who runs stages 3-5. The report operator needs no Azure
access unless using live grouping.

## 1. Confirm the billing scope and permissions

Identify whether the customer uses **Enterprise Agreement (EA)** or
**Microsoft Customer Agreement (MCA)**. Decide which enrollment or billing
profile and which complete calendar month to analyze. Use a closed month with
all daily usage available.

| Task | Suitable read-only access | Who needs it |
| --- | --- | --- |
| Download EA usage, contract pricesheet and reservation inventory | Enterprise Administrator (read only) on the enrollment | Person exporting files |
| Download MCA usage, contract pricesheet and reservation inventory | Billing profile reader on the profile, or inherited Billing account reader | Person exporting files |
| Run the report on supplied files | Local input-file read and output/cache-folder write access | Report operator |
| Optional management-group grouping | Reader on every relevant subscription, directly or inherited from a management group | Identity signed in to Azure CLI |
| Optional EA department grouping | Enterprise Administrator (read only) on the relevant enrollment | Identity signed in to Azure CLI |

See [PERMISSIONS.md](PERMISSIONS.md) for reservation-specific alternatives,
assignment scopes and Microsoft references. Subscription Reader alone does
not provide the billing access needed for the complete export workflow.

## 2. Obtain the input files

### Choose the report scope

| Report capability | Files needed | Azure login during the report run |
| --- | --- | --- |
| Basic VM and Storage impact | Amortized usage | No |
| Contract prices and price protection | Usage + pricesheet | No |
| Reservation inventory enrichment | Usage + reservation inventory | No |
| Full file-based report, including RI forecast and price protection | Usage + pricesheet + reservation inventory | No |
| Add business-entity grouping to any of the above | Same files + live EA department or management-group lookup | Yes |

### Download amortized usage: always required

Open [Azure portal](https://portal.azure.com) > **Cost Management + Billing** >
**Billing scopes**. Select the relevant EA billing account or MCA billing
profile.

| Agreement | Export action |
| --- | --- |
| EA | Open **Invoices**, select the relevant period, and download usage details with **amortized** costs |
| MCA | Create an **amortized usage report/export** for the selected billing profile and full calendar month, then download the generated data files |

Portal labels can vary by agreement and available export experience. The
required result is the detailed **amortized** usage dataset, not a PDF invoice,
a summarized Cost Analysis download or an actual-cost export.

Preserve the original columns. CSV and Parquet are supported. If the export is
compressed, extract it before use. If it is split into multiple data files,
keep all parts for that period in a dedicated folder, without unrelated
exports or copies that would double-count usage.

Use one EA/MCA contract type and one pricing currency per run. Do not combine
several months or overlapping exports in a single input folder.

### Download the contract pricesheet

Within the same billing scope, open **Invoices** and download the **Azure
pricesheet** appropriate to the usage and agreement being analyzed.

Use your contract pricesheet, not a public retail-pricing download. Its
contract type must match the usage: EA with EA, or MCA with MCA. Preserve all
columns, including available base prices and RI/Savings Plan terms. The tool
accepts CSV or Parquet.

### Export reservation inventory

Within the billing scope, open **Reservations** or
**Reservations + Hybrid Benefit** and export the inventory to CSV. With
reservation-specific RBAC access, use **Home > Reservations** instead.

The CSV must contain these exact headers:

```text
Reservation Id
Reservation order Id
Purchase date
Expiration date
Product name
```

Keep all reservations needed to match the selected month's usage, including
relevant expired reservations. Reservation/order ID pairs must be populated
and unique. Preserve the original export rather than recreating IDs or dates
manually. RI forecasting requires VM reservation usage in the usage export;
an inventory alone is not enough.

**Ready to continue:** the usage export exists; for the full report, the
matching pricesheet and reservation inventory also exist.

## 3. Prepare the workstation

### Install software

| Dependency | Required for | Installation |
| --- | --- | --- |
| Python 3.10 or newer, with pip and venv | Every report run | [Python downloads](https://www.python.org/downloads/); ensure `python` is available in a new terminal |
| Git | Cloning the repository | [Git downloads](https://git-scm.com/downloads); alternatively download and extract the repository ZIP |
| Python packages from `requirements.txt` | Reading inputs, calculation, Excel output and optional Azure lookups | Install into the virtual environment using the command below |
| Azure CLI | Only live business-entity grouping | [Azure CLI installation](https://learn.microsoft.com/cli/azure/install-azure-cli) |
| Microsoft Excel | Reviewing the workbook and calculating dashboard formulas | Use your organization's approved installation |

`requirements.txt` installs `pandas`, `pyarrow`, `openpyxl`, `azure-core`,
`azure-identity` and `azure-mgmt-resourcegraph`. The `core` modules are already
part of this repository; do not install or run a separate `core` tool.

The examples below use **Windows PowerShell** from the repository root.
They invoke the virtual environment's Python directly, so activating a
PowerShell script or changing execution policy is unnecessary.

### Download and install

```powershell
python --version
git clone https://github.com/gibet-code/az-pricing-updates.git
Set-Location ".\az-pricing-updates"
python -m venv ".venv"
& ".\.venv\Scripts\python.exe" -m pip install -r ".\requirements.txt"
& ".\.venv\Scripts\python.exe" -m pricing_updates.impact_report --help
```

If using the ZIP instead, extract it and open PowerShell in the extracted
folder containing `README.md` and `requirements.txt`. Skip the `git clone`
and `Set-Location` commands, then run the virtual-environment and install
commands.

### Place the inputs

```powershell
New-Item -ItemType Directory -Force -Path ".\inputs", ".\reports", ".\cache" | Out-Null
```

Copy the exports to these example locations. The filenames are examples;
either rename the files or adjust the report commands to their actual names.

```text
az-pricing-updates\
  inputs\
    usage.csv
    prices.csv
    reservedInstances.csv
  reports\
  cache\
```

Keep billing files in approved local storage. Inputs, reports and caches
contain customer billing information and should not be committed to Git.
The example folders and CSV/Parquet/Excel files are ignored by this repository.

### Optional: convert large CSV inputs

For usage CSVs larger than about 1 GB, convert to Parquet before reporting.
The usage converter streams the CSV in bounded memory; Parquet also speeds
up later runs.

```powershell
& ".\.venv\Scripts\python.exe" -m scripts.convert_usage_details_to_parquet ".\inputs\usage.csv" ".\inputs\usage.parquet"
& ".\.venv\Scripts\python.exe" -m scripts.convert_pricesheet_to_parquet ".\inputs\prices.csv" ".\inputs\prices.parquet"
```

For multipart usage, replace `.\inputs\usage.csv` with a folder containing
only that export's CSV parts, such as `.\inputs\usage-csv`.
After conversion, use `usage.parquet` and `prices.parquet` in place of the
CSV paths in the commands below. Leave reservation inventory as CSV.

**Ready to continue:** `--help` opens successfully, and the input paths match
your files.

## 4. Generate the workbook

Run one of the following commands from the repository root. You do not need
to run the basic report before the full report. Close an existing output
workbook in Excel before rerunning a command that writes to the same path.

### Basic report: usage only

```powershell
& ".\.venv\Scripts\python.exe" -m pricing_updates.impact_report ".\inputs\usage.csv" --output ".\reports\impact-basic.xlsx"
```

### Full file-based report: no Azure login

```powershell
& ".\.venv\Scripts\python.exe" -m pricing_updates.impact_report ".\inputs\usage.csv" `
  --output ".\reports\impact-full.xlsx" `
  --pricesheet ".\inputs\prices.csv" `
  --reservation-inventory ".\inputs\reservedInstances.csv" `
  --price-protection `
  --ri-price-evolution `
  --cache-dir ".\cache"
```

This includes contract prices, price protection, reservation inventory and an
RI forecast with the default **18-month horizon** and **January fiscal-year
start**. Forecasting requires qualifying VM reservation usage and usable
pricing data; supplying flags cannot create missing source data.

To change the forecast, append `--ri-price-evolution-horizon-months 24` or
`--ri-price-evolution-fiscal-year-start-month 4`, for example. Fiscal-year
months are numbered `1` (January) to `12` (December).

By default, the VM and Storage detail tabs retain only impacted rows.
Append `--keep-no-impact-details` if you also need the non-impacted detail
rows; this can increase workbook size.

### Optional: add business-entity grouping

Skip this section if grouping by subscription is sufficient.
Install Azure CLI and confirm the access from stage 1 before signing in:

```powershell
az login --tenant "<tenant-id>"
```

Replace `<tenant-id>` with the customer's tenant ID. For an identity with
billing access but no subscriptions, add `--allow-no-subscriptions`.

Append **one** of these options to the full report command:

| Grouping | Option | Meaning |
| --- | --- | --- |
| EA department | `--business-entity-method ea_department_name` | Current department from EA Billing APIs; EA only |
| Management group | `--business-entity-method management_group_hierarchy_level_1` | Top-level management group below the tenant root; EA or MCA |

Management-group level `0` means the tenant root; levels `1` through `6`
select lower hierarchy levels. Choose a level that exists for the
subscriptions being analyzed. Only one grouping method is supported per run.

**Azure login does not download usage, pricesheets or inventory.** These
remain local inputs. Live lookups use the current CLI identity and read Azure
metadata only.

**Ready to continue:** the report command finishes and the requested workbook
exists in `.\reports`. Read any warnings before interpreting its totals.

## 5. Open and validate the results

Open the generated workbook in Microsoft Excel and allow its formulas to
recalculate. The Python writer requests recalculation but does not calculate
Excel formulas itself; previews may not show calculated dashboard results.

| Worksheet | What to review |
| --- | --- |
| Non-RI Pricing Impact | Source metadata, date range, pricing currency, overall estimated increase, and subscription/entity breakdowns; the pricing-model selector filters the breakdowns |
| Storage / Virtual Machines | Impact classification, impacted costs and price-protection status; VM family and retirement dates where known |
| Storage Pricesheet / VM Pricesheet | Matched contract pricing used by the report |
| Reservations | Inventory records and the source reservation information |
| RI Price Evolution / RI Projection Details / RI Evolution Assumptions | Monthly forecast, fallback scenarios, unpriced results, horizon and assumptions |

Confirm the source scope, period and currency before reading the impact as a
monthly estimate. A full calendar month gives a monthly reference; partial
periods are not automatically normalized into a full month.

Review **Unpriced** forecast results and **Unknown** business entities.
Unpriced does not mean zero cost; it means required data, such as expiration
dates or fallback prices, could not be resolved. Unknown can indicate missing
access, missing department mapping or a hierarchy level that does not exist.

Existing RI costs are excluded from the Non-RI summary. Their expiration
effects are modeled separately in the RI forecast, which covers VM reservation
usage and unused commitment only, not Storage reserved capacity.

The forecast holds the reference month's usage constant. It does not model
future workload growth or migrations. Savings Plan results assume additional
commitment to maintain coverage after price increases. See the
[README limitations](README.md#limitations) and workbook assumptions before
using the results for budgeting.

## Troubleshooting

| Symptom | Action |
| --- | --- |
| `python` not found or version below 3.10 | Install a supported Python version and open a new terminal |
| Missing Python module | From the repository root, rerun the virtual environment's pip install command from stage 3 |
| `No module named pricing_updates` | Open the repository root, not `core` or `pricing_updates`, and use the virtual-environment Python |
| File not found | Check the actual filenames, extensions and extraction folder; update the command paths |
| Contract mismatch or several pricing currencies | Use matching EA/MCA usage and pricesheet inputs; prepare separate runs for different pricing currencies |
| Missing columns, or Purchase/Refund rows rejected during inventory enrichment | Re-export the original supported data, preserving headers; use amortized rather than actual-cost usage |
| Reservation inventory errors | Check the exact required headers, populated IDs and unique reservation/order pairs |
| Forecast missing or Unpriced | Check for VM reservation usage, matching inventory dates and usable pricesheet terms; do not treat missing prices as zero |
| CSV loading is slow or runs out of memory | Convert usage to Parquet with the streaming converter before rerunning |
| Azure authentication error or `Unknown` grouping | Check CLI login, tenant, roles and hierarchy level using [PERMISSIONS.md](PERMISSIONS.md) |
| Cannot write workbook or dashboard values appear blank | Close the output file before rerunning; open the completed workbook in Excel and recalculate formulas |

For a later reporting month, obtain fresh exports, keep that period's files
separate, and rerun stage 4 with the new paths and a distinct output filename.
The optional cache reuses parsed usage and pricesheets; it does not download
new data, and reservation inventory and live business-entity lookups are not
cached.
