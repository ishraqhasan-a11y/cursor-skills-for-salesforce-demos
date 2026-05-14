---
name: salesforce-work-plans
description: Add Work Plans and Work Steps to Salesforce Work Orders via the sf CLI.
---

## Prerequisites

- `sf` CLI authenticated to the target org: `sf org login web --alias <orgAlias>`
- `jq` installed (`brew install jq` if missing)
- The user has shared (or you know) the **org alias** and either a **Work Order Number** or a **Work Order Id**

## Quick Workflow

Run these in order. Each block is copy-paste safe.

### 1. Set variables

```bash
ORG="<orgAlias>"                  # e.g. customerOrg
WO_NUM="<workOrderNumber>"        # e.g. 00001016 (often 8 digits, check the org)
```

### 2. Resolve the Work Order Id

```bash
WO_ID=$(sf data query --target-org "$ORG" \
  --query "SELECT Id FROM WorkOrder WHERE WorkOrderNumber='${WO_NUM}' LIMIT 1" \
  --json | jq -r '.result.records[0].Id')

echo "WO_ID=$WO_ID"
```

If `WO_ID` is `null`, the number format is wrong. List recent Work Orders to find the right format:

```bash
sf data query --target-org "$ORG" \
  --query "SELECT Id, WorkOrderNumber, Subject FROM WorkOrder ORDER BY CreatedDate DESC LIMIT 20"
```

### 3. Create the Work Plan

`WorkPlanTemplateId` is **not directly writable** via the CLI, so create the Work Plan without a template and add steps manually.

```bash
sf data create record \
  --target-org "$ORG" \
  --sobject WorkPlan \
  --values "Name=Field-Service-Execution-Plan ParentRecordId=${WO_ID}"
```

### 4. Grab the new Work Plan Id

```bash
WP_ID=$(sf data query --target-org "$ORG" \
  --query "SELECT Id FROM WorkPlan WHERE ParentRecordId='${WO_ID}' ORDER BY CreatedDate DESC LIMIT 1" \
  --json | jq -r '.result.records[0].Id')

echo "WP_ID=$WP_ID"
```

### 5. Add Work Steps

Customize the step names for the scenario. Example for a generic field service job:

```bash
sf data create record --target-org "$ORG" --sobject WorkStep \
  --values "Name='Confirm site access and contact' WorkPlanId=${WP_ID} ExecutionOrder=1"

sf data create record --target-org "$ORG" --sobject WorkStep \
  --values "Name='Inspect equipment and diagnose' WorkPlanId=${WP_ID} ExecutionOrder=2"

sf data create record --target-org "$ORG" --sobject WorkStep \
  --values "Name='Complete repair or replacement' WorkPlanId=${WP_ID} ExecutionOrder=3"

sf data create record --target-org "$ORG" --sobject WorkStep \
  --values "Name='Customer signoff and ticket closure' WorkPlanId=${WP_ID} ExecutionOrder=4"
```

### 6. Verify

```bash
sf data query --target-org "$ORG" \
  --query "SELECT Id, Name FROM WorkPlan WHERE ParentRecordId='${WO_ID}'"

sf data query --target-org "$ORG" \
  --query "SELECT Id, Name, ExecutionOrder FROM WorkStep WHERE WorkPlanId='${WP_ID}' ORDER BY ExecutionOrder"
```

## Known Platform Limitations

These were discovered the hard way—don't try to "fix" them.

### Milestones cannot be inserted directly

`EntityMilestone` (and legacy `WorkOrderMilestone`) reject direct API inserts with:

```
Error (1): entity type cannot be inserted: Object Milestone
```

Milestones are generated **only** by the Entitlement Process engine. To get milestones to appear:

1. Confirm there's an active `SlaProcess` with `SObjectType='WorkOrder'`:
   ```bash
   sf data query --target-org "$ORG" \
     --query "SELECT Id, Name, SObjectType, IsActive FROM SlaProcess WHERE IsActive=true"
   ```
2. If yes, link the matching Entitlement to the Work Order:
   ```bash
   sf data update record --target-org "$ORG" --sobject WorkOrder \
     --record-id "$WO_ID" --values "EntitlementId=<ENT_ID>"
   ```
3. If no WorkOrder-typed SlaProcess exists, milestones cannot be auto-created. Skip them and use Work Plans instead.

### WorkPlanTemplateId is read-only on insert

Direct create with `WorkPlanTemplateId=...` fails with a field-level security error. Salesforce only allows linking via the standard invocable action. The reliable workaround is to create the Work Plan without a template and add steps manually (see step 3 above).

### Work Order Numbers may be 8 digits, not 7

Orgs vary—`0001014` and `00001014` are not the same string. Always run a `LIMIT 20 ORDER BY CreatedDate DESC` query first if the lookup returns `null`.

### `sf config set` outside a DX project

If you run `sf config set target-org=...` outside a Salesforce DX project directory, you get:

```
Error (InvalidProjectWorkspaceError)
```

Fix: pass `--target-org <alias>` explicitly on every command (recommended), or `cd` into a DX project first.

## Useful Discovery Queries

```bash
# List all Work Plan Templates in the org
sf data query --target-org "$ORG" \
  --query "SELECT Id, Name FROM WorkPlanTemplate ORDER BY Name"

# List all Milestone Types
sf data query --target-org "$ORG" \
  --query "SELECT Id, Name FROM MilestoneType ORDER BY Name"

# List all active Entitlement Processes
sf data query --target-org "$ORG" \
  --query "SELECT Id, Name, SObjectType, IsActive FROM SlaProcess WHERE IsActive=true"

# Describe any object's fields
sf sobject describe --target-org "$ORG" --sobject <ObjectName> --json \
  | jq -r '.result.fields[] | "\(.name) (\(.type)) - \(.label)"'
```

## Tips

- Always pass `--target-org "$ORG"` explicitly; don't rely on the default config.
- If shell variables get wiped (new terminal), re-run the resolution block in steps 1–2 and 4.
- For richer demos, create multiple Work Plans on the same Work Order (e.g., Execution, Safety, Closeout) by repeating steps 3–5 with different `Name=` values.
