# Lab: Create a Dataverse Table for Material Records, Add Validation Fields, and Simulate a Naming-Schema Check with Power Automate

## Scenario

Your organization tracks **material records** (raw materials, finished goods, internal SKUs, etc.) in Dataverse so multiple apps and automation can reuse a common master dataset. To improve data quality, every new material must conform to a **naming schema** (e.g., prefix + category + numeric sequence).

In this lab you will:

- Create a Dataverse table for material master records.
- Add **validation fields** that track whether a material is schema-compliant.
- Build a Power Automate flow that simulates a **naming-schema validation check** when a record is created or updated.
- Provide an audit-friendly result (status + message) and demonstrate a remediation loop.

---

## Learning objectives

By the end of this lab, learners will be able to:

1. Create a Dataverse table with appropriate data types and column choices.
2. Implement a validation pattern using boolean/status/message columns.
3. Build a Power Automate flow using the Dataverse trigger and update actions.
4. Validate a naming schema using expressions (prefix, category segment, numeric segment).
5. Implement a "pending validation" → "valid/invalid" lifecycle for data quality automation.

---

## Prerequisites

- Power Apps / Dataverse environment where you have permission to:
  - Create tables and columns
  - Create cloud flows
- Power Automate access (Dataverse connector)
- (Optional) Model-driven app access for quick testing, or use Power Apps maker portal’s **Data** tab.

---

## Naming schema used in this lab

You will validate materials using this pattern:

**`MAT-<CAT>-<NNNN>`**

Examples:

- `MAT-RM-0001` ✅  
- `MAT-FG-1023` ✅  
- `MAT-RM-12` ❌ (numeric segment not 4 digits)  
- `MATERIAL-RM-0001` ❌ (wrong prefix)  
- `MAT-XX-0001` ❌ (unknown category)  

Where:

- Prefix must be `MAT-`
- Category (`<CAT>`) must be one of: `RM`, `FG`, `PKG`
- Numeric segment must be exactly **4 digits**

---

## Part A — Create Dataverse tables (20–25 minutes)

### Step 1: Create the Materials table

1. Go to **Power Apps** (<https://make.powerapps.com>) → **Tables** → **New table** → **Table (advanced properties)**.
2. Display name: **Materials**
3. Plural name: **Materials**
4. Primary column:
   - Display name: **Material Name**
   - Schema name: auto-generated (leave as default)
5. Select **Save**.

### Step 2: Add business columns to Materials

Open the **Materials** table and add these columns:

| Display name | Type | Suggested details |
| --- | --- | --- |
| Material Code | Single line of text | Business required, set max length 20–40 |
| Category | Choice | Choices: RM, FG, PKG |
| Description | Multiple lines of text | Optional |
| Unit of Measure | Choice | Example: EA, KG, L (keep small) |
| Is Active | Yes/No | Default = Yes |
| Effective Date | Date only | Optional |

> **Tip:** In Dataverse, "Material Name" can be your friendly name, while "Material Code" is what you validate and what downstream systems use.

### Step 3: Add validation fields to Materials

Add the following columns that support validation and auditability:

| Display name | Type | Suggested details |
| --- | --- | --- |
| Validation Status | Choice | Values: Pending, Valid, Invalid; default: `Pending` |
| Validation Message | Multiple lines of text | Store why invalid |
| Last Validated On | Date and time | Store when the check ran |
| Validation Correlation ID | Single line of text | Store a GUID per validation run |
| Schema Version | Single line of text | Optional version for tracking |

### Step 4: (Optional but recommended) Create a Naming Rules reference table

This makes the lab more "enterprise-like" and future-proof.

1. Create a new table: **Material Naming Rules**
2. Add columns:

| Display name | Type | Example |
| --- | --- | --- |
| Prefix | Single line of text | `MAT` |
| Allowed Categories | Multiple lines of text | `RM,FG,PKG` |
| Numeric Length | Whole number | `4` |
| Schema Version | Single line of text | `v1` |
| Is Active | Yes/No | Yes |

1. Create **one** rule row with the example values above.

> If you skip this table, you’ll hardcode the rules in the flow.

---

## Part B — Create test data (10 minutes)

### Step 1: Create a few materials

Create 4–5 records in **Materials**. Use these examples (or similar):

| Material Name | Material Code | Category | Schema Version | Expected |
| --- | --- | --- | --- | --- |
| Steel Rod | MAT-RM-0001 | RM | v1 | Valid |
| Cardboard Box | MAT-PKG-0042 | PKG | v1 | Valid |
| Widget A | MAT-FG-120 | Empty | v1 | Invalid (not 4 digits) |
| Mystery Item | MAT-XX-0005 | RM or FG | v1 | Invalid (category segment XX) |
| Wrong Prefix | PROD-RM-0002 | RM | v1 | Invalid (prefix) |

Ensure **Validation Status** remains **Pending** for all new items (default).

---

## Part C — Build the Power Automate validation flow (35–45 minutes)

### Step 1: Create the flow

1. Go to **Power Automate** (<https://make.powerautomate.com>) → Verify in **Dev One** → Create** → **Automated cloud flow**
2. Name: **DVC – Validate Material Naming Schema (v1)**
3. Trigger: **Dataverse** → **When a row is added, modified or deleted**
4. Change type: **Added or Modified**
5. Table name: **Materials**
6. Scope: **Organization** (or your preference)

> Using Added/Modified makes this flow reusable when users fix codes later.

### Step 2: Add a trigger condition (avoid infinite loops)

Because the flow will update the same row, add a trigger condition so it runs only when validation is needed.

Open the trigger’s **Settings** → **Trigger conditions** and add one condition (copy/paste):

```text
@or(equals(triggerOutputs()?['body/pfx_validationstatus'], 0),equals(triggerOutputs()?['body/pfx_validationstatus'], 1))
```

**What this does:**

- Runs only when Validation Status is Pending (depending on how the choice value is represented in your environment).

> If your choice values map differently, you can simplify and instead run the flow only when **Material Code** changes, but that’s a more advanced condition.

### Step 3: Initialize variables

Add actions - right-click and choose parallel branch + scope:

#### 3.1 Initialize variable: varCorrelationId

- Name: `varCorrelationId`
- Type: String
- Value: `guid()`

#### 3.2 Initialize variable: varCode

- Name: `varCode`
- Type: String
- Value: `triggerOutputs()?['body/pfx_materialcode']` (choose Material Code dynamic content)

#### 3.3 Initialize variable: varMessage

- Name: `varMessage`
- Type: String
- Value: *(blank)*

#### 3.4 Initialize variable: varIsValid

- Name: `varIsValid`
- Type: Boolean
- Value: `true`

### Step 4: (Optional) Load naming rules from reference table

If you created **Material Naming Rules**:

1. Add action: **List rows** (Dataverse)
2. Table name: **Material Naming Rules**
3. Filter rows:
   - `pfx_isactive eq true and pfx_schemaversion eq 'v1'`
4. Add **Compose** `cmpRule`:

   ```text
   first(outputs('List_rows')?['body/value'])
   ```

If you skipped the reference table, proceed with the hardcoded rules below.

### Step 5: Parse the code into segments

We’ll split `MAT-RM-0001` by `-` to get segments:

- Segment 0: `MAT`
- Segment 1: `RM`
- Segment 2: `0001`

Add a **Compose** action: `cmpSegments`
Expression:

```text
split(variables('varCode'), '-')
```

Add a **Compose** action: `cmpSegmentCount`
Expression:

```text
length(outputs('cmpSegments'))
```

### Step 6: Validate required structure (3 segments)

Add a **Condition**:

- Left: `outputs('cmpSegmentCount')`
- Operator: **is equal to**
- Right: `3`

If **No**:

1. Set `varIsValid` = `false`
2. Set `varMessage` = `Invalid format. Expected MAT-<CAT>-<NNNN> (3 segments).`
3. Go to **Step 10: Update row**

If **Yes**, continue.

### Step 7: Validate prefix segment

Add **Compose**: `cmpPrefix`

```text
first(outputs('cmpSegments'))
```

Add a **Condition**:

- Left: `outputs('cmpPrefix')`
- Operator: **is equal to**
- Right: `outputs('cmpRule')?['pfx_prefix']`

If **No** - use copy/paste:

1. Set `varIsValid` = `false`
2. Set `varMessage` = `Invalid prefix. Expected MAT-`
3. Go to **Step 10: Update row**

If **Yes**, continue.

### Step 8: Validate category segment

Add **Compose**: `cmpCategory`

```text
outputs('cmpSegments')?[1]
```

Add a **Condition** (hardcoded categories):

- If `cmpCategory` is equal to `RM` OR `FG` OR `PKG`

`split(outputs('cmpRule')?['pfx_allowedcategories'],',')`
contains
`outputs('cmpCategory')`

If **No**:

1. Set `varIsValid` = `false`
2. Set `varMessage` = `Invalid category segment. Allowed: outputs('cmpRule')?['pfx_allowedcategories'].`
3. Go to **Step 10: Update row**

If **Yes**, continue.

> **Optional alignment check:** Compare cmpCategory with the table’s **Category** choice to ensure they match. This is a great instructor-led extension.

### Step 9: Validate numeric segment (length + digits)

Add **Compose**: `cmpNumber`

```text
last(outputs('cmpSegments'))
```

#### 9.1 Condition: numeric segment length is 4

- Left: `length(outputs('cmpNumber'))`
- Operator: **is equal to**
- Right: `outputs('cmpRule')?['pfx_numericlength']`

If **No**:

1. Set `varIsValid` = `false`
2. Set `varMessage` = `Invalid numeric segment length. Expected outputs('cmpRule')?['pfx_numericlength'] digits.`
3. Go to Step 10

If **Yes**, continue.

#### 9.2 Condition: numeric segment contains only digits

Use condition with `isInt(outputs('cmpNumber'))` is equal to `true`

Power Automate doesn’t have a perfect regex check everywhere, but a reliable simulation is:

- Convert to int, then back to string, and compare length
- Handle leading zeros carefully

A safer "simulation" for teaching is: reject if the segment contains any non-digit characters by removing digits and checking what remains.

Add **Compose**: `cmpNonDigits`
Expression:

```text
replace(
  replace(
    replace(
      replace(
        replace(
          replace(
            replace(
              replace(
                replace(
                  replace(outputs('cmpNumber'),'0',''),
                '1',''),
              '2',''),
            '3',''),
          '4',''),
        '5',''),
      '6',''),
    '7',''),
  '8',''),
'9','')
```

Add **Condition**:

- Left: `length(outputs('cmpNonDigits'))`
- Operator: **is equal to**
- Right: `0`

If **No**:

1. Set `varIsValid` = `false`
2. Set `varMessage` = `Numeric segment must contain only digits (0–9).`
3. Go to Step 10

If **Yes**, continue.

#### 9.3 If all checks passed

Set `varIsValid` = `true`
Set `varMessage` = `Schema check passed (v1).`

---

## Part D — Update the Dataverse row with validation results (10 minutes)

### Step 10: Update the material row

Add action: **Update a row** (Dataverse)

- Table name: **Materials**
- Row ID: **Materials** row id from trigger (dynamic content) - triggerOutputs()?['body/pfx_materialsid']

Set fields:

- **Validation Status**:
  - If `varIsValid` is true → `Valid`
  - Else → `Invalid`
  - `if(variables('varIsValid'), 2, 3)`
- **Validation Message**: `varMessage` [`variables('varMessage')`]
- **Last Validated On**: `utcNow()`
- **Validation Correlation ID**: `varCorrelationId` [`variables('varCorrelationId`)`]
- **Schema Version**: `v1` (or keep existing)

---

## Part E — Test and observe results (10–15 minutes)

### Test 1: Valid codes

1. Create or edit a material with `MAT-RM-0001`.
2. Confirm the flow runs.
3. Confirm the material record updates:
   - Validation Status = Valid
   - Validation Message indicates pass
   - Last Validated On is set

### Test 2: Invalid codes

1. Update a material to `MAT-FG-12A4` or `PROD-RM-0001`.
2. Confirm:
   - Validation Status = Invalid
   - Validation Message explains what failed

### Test 3: Remediation loop

1. Pick an invalid record and correct the Material Code.
2. Set Validation Status back to **Pending** (manually) to trigger the flow again.
3. Confirm it becomes Valid.

> This models real-world remediation: humans fix data, automation validates again.

---

### Optional: "Short-Circuit" flow where possible

Copy/paste update action and terminate action

---

## Optional challenge activities (choose 1–3)

### Challenge 1: Align category segment with Category column

Add a check:

- If `cmpCategory` does not match the record’s Category choice, mark Invalid with message:
  - "Category mismatch: code segment RM but Category field is FG."

### Challenge 2: Enforce uniqueness of Material Code

Add a **List rows** action:

- Filter: `materialcode eq '<current code>' and materialsid ne '<current row id>'`
If any results exist → Invalid with message "Duplicate Material Code."

### Challenge 3: Add a "Validation Queue" table

Create a second table **Material Validation Queue** and write a row when status is Invalid:

- Material reference
- Failure reason
- Assigned validator (optional)
- Queue status (Open/Closed)

This connects well to later PL-7002 topics about orchestration and queue-based processing.

### Challenge 4: Add schema versioning

Extend the naming rules table to include v2 pattern (e.g., `MAT-<CAT>-<PLANT>-<NNNN>`), then branch logic based on Schema Version.

---

## Troubleshooting tips

- **Flow triggers repeatedly:** ensure trigger conditions restrict runs to Pending status only, or add a "Do not run if correlation id is present" pattern.
- **Choice fields not updating:** verify you selected the correct choice option in Update a row (sometimes requires reselecting after table changes).
- **Null Material Code:** add a validation step early:
  - If empty → Invalid message "Material Code is required."
- **Non-digit simulation too complex:** you can simplify by validating only segment count and length for teaching, then present digit-check as an extension.

---

## Deliverables checklist

Learners should have:

- [ ] Dataverse table **Materials** with business + validation columns
- [ ] (Optional) **Material Naming Rules** table with active rule row
- [ ] Automated flow that:
  - [ ] Parses `Material Code` segments
  - [ ] Validates prefix/category/numeric segment
  - [ ] Updates the row with Valid/Invalid + message + timestamp
- [ ] Test records showing both pass and fail outcomes

---

## Cleanup (optional)

- Delete test rows from **Materials**
- Disable or delete the validation flow
- Keep the Materials table for later labs (approvals, exceptions, governance, reporting)
