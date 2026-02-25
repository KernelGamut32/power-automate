# Lab: Approval Workflow for Valid Materials in Power Automate (Follow-up)

> Follow-up lab to: **Create a Dataverse table for material records, add validation fields, and simulate a naming-schema check using a Power Automate flow.**  
> In the first lab, students built a validation flow that updates **Validation Status** to **Valid** or **Invalid**. In this lab, you will add an **approval workflow** that automatically routes **Valid** materials to approvers and then notifies a downstream mailbox when approvals complete.

---

## Scenario

Your organization requires a **two-person approval** before a newly validated material can be used by downstream processes (procurement, production, catalog publishing, etc.).

When a material’s **Validation Status** becomes **Valid**:

1. Power Automate sends an approval request to **two people**.
2. When **both** approve:
   - The material record is updated to show **Approved**
   - A notification email is sent to a configured email address
3. If **either** rejects:
   - The material record is updated to show **Rejected**
   - A rejection notification email is sent

---

## Learning objectives

By the end of this lab, learners will be able to:

- Add approval-related columns to a Dataverse table
- Build an approval flow that triggers from Dataverse changes
- Use **Start and wait for an approval** with multiple approvers
- Prevent infinite loops using trigger conditions and status fields
- Update Dataverse rows based on approval outcomes
- Send notification emails after the approval completes

---

## Estimated time

45–60 minutes

---

## Prerequisites

- Completed the prior lab (Materials table + validation flow)
- Permissions in the environment to:
  - Edit Dataverse tables and columns
  - Create cloud flows
  - Send approvals and email (Approvals + Outlook connectors are common in M365 tenants)
- Two approver accounts available (can be two students, or instructor + student)

---

## What you will build

- Dataverse enhancements:
  - **Approval Status** (Pending/Approved/Rejected)
  - **Approval Requested On**
  - **Approved On**
  - **Approver Comments**
  - (Optional) **Approver Emails** (for auditing)
- Flow:
  - Trigger: **When a row is modified** (Dataverse)
  - Condition: only run when **Validation Status = Valid** AND **Approval Status = Pending**
  - Action: request approval from two people
  - Update record + email notifications for approve/reject paths

---

## Part A — Add approval tracking columns to the Materials table (10–15 minutes)

### Step A1: Open the Materials table

1. Go to **Power Apps** maker portal: `https://make.powerapps.com`
2. Select your environment (top-right).
3. Navigate to **Dataverse** → **Tables**.
4. Open **Materials**.

### Step A2: Add columns for approval tracking

Add the following columns (names may be prefixed automatically in your environment):

| Display name | Type | Required | Notes / Values |
| --- | --- | --- | --- |
| Approval Status | Choice | Yes | Values: **Pending**, **Approved**, **Rejected**. Default = **Pending** |
| Approval Requested On | Date and time | No | Timestamp when approval is requested |
| Approved On | Date and time | No | Timestamp when approval completes |
| Approver Comments | Multiple lines of text | No | Store approval/rejection comments |
| Approver Emails | Multiple lines of text | No | Optional audit field (store approver emails joined by semicolon) |

**Save** the table changes.

> Tip: If “Choice” column creation asks for a Choice set, you can create a new choice set called **Material Approval Status** and reuse it later.

---

## Part B — Prepare test records (5–10 minutes)

You need at least one material record that is **Valid** but not yet approved.

### Step B1: Create or update a material record

1. Open **Materials** and edit/create a record.
2. Set:
   - **Material Code** to a valid pattern from the first lab (example: `MAT-RM-0001`)
   - **Validation Status** = **Valid** (either by running your validation flow or setting it manually for testing)
   - **Approval Status** = **Pending**

> If your validation flow sets Validation Status automatically, you can create a record and let the first flow mark it **Valid**.

---

## Part C — Create the approval flow (30–40 minutes)

### Step C1: Create an Automated cloud flow

1. Go to **Power Automate**: `https://make.powerautomate.com`
2. Select **Create** → **Automated cloud flow**
3. Flow name: **DVC – Approve Valid Materials**
4. Choose trigger: **Microsoft Dataverse** → **When a row is added, modified or deleted**
5. Click **Create**

### Step C2: Configure the trigger

In the trigger action:

- **Change type**: `Modified`  
- **Table name**: `Materials`  
- **Scope**: `Organization` (or per your lab preference)

> Why Modified? Approvals should fire when the record becomes Valid (a modification), not only when created.

### Step C3: Add trigger conditions to prevent infinite loops

Because the flow updates the same row (which could re-trigger itself), add trigger conditions that only allow the flow to run when approvals are actually needed.

1. On the trigger, select **… (ellipsis)** → **Settings**
2. Find **Trigger conditions**
3. Add **one** trigger condition (copy/paste as-is)

```text
@and(
  equals(triggerOutputs()?['body/<VALIDATIONSTATUS_LOGICALNAME>'], <VALID_VALUE>),
  equals(triggerOutputs()?['body/<APPROVALSTATUS_LOGICALNAME>'], <PENDING_VALUE>)
)
```

#### How to fill in the placeholders

You must replace the 4 placeholders based on your environment:

- `<VALIDATIONSTATUS_LOGICALNAME>` = the logical name for **Validation Status**  
- `<APPROVALSTATUS_LOGICALNAME>` = the logical name for **Approval Status**
- `<VALID_VALUE>` = the numeric value for the **Valid** choice
- `<PENDING_VALUE>` = the numeric value for the **Pending** choice

##### Method to discover these values (no guessing)

- Add a temporary step after the trigger: **Compose** named `cmpTriggerBody`
- In **Inputs**, use the expression:

  ```text
  triggerOutputs()?['body']
  ```

- Save the flow.
- Modify a Materials record and run the flow once.
- Open the run history → open `cmpTriggerBody` output.
- Search within the JSON output for your two columns:
  - Find the property name used for **Validation Status**
  - Find the property name used for **Approval Status**
- Note the integer values when they are set to **Valid** and **Pending**.

Then:

- Update the trigger condition placeholders with the discovered logical names and numbers
- Delete the temporary Compose action after you’re done

> Instructor note: This “inspect the trigger payload” technique is a key troubleshooting skill in Power Automate.

### Step C4: Initialize variables (recommended)

Add these actions:

#### 1) Initialize variable — `varMaterialName` (String)

- Value: **Material Name** (dynamic content from trigger)

#### 2) Initialize variable — `varMaterialId` (String)

- Value: the record ID (Row ID) from trigger

#### 3) Initialize variable — `varApproverList` (String)

- Value: two email addresses separated by semicolon, for example:
  - `approver1@contoso.com;approver2@contoso.com`

> If students don’t have two distinct accounts, use instructor + student, or two instructors.

#### 4) Initialize variable — `varNotifyTo` (String)

- Value: an email address that should be notified after approval, e.g.:
  - `materials-notify@contoso.com`
  - or your own email for testing

---

## Part D — Request approval (10–15 minutes)

### Step D1: Stamp “Approval Requested On” before creating the approval

Add action: **Dataverse** → **Update a row**

- **Table name**: `Materials`
- **Row ID**: `varMaterialId`
- Set:
  - **Approval Requested On** = `utcNow()`
  - **Approver Comments** = *(leave blank)*
  - **Approved On** = *(leave blank)*

> Do **not** change Approval Status here; it should already be Pending.

### Step D2: Add the approval action

Add action: **Approvals** → **Start and wait for an approval**

Configure:

- **Approval type**: `Approve/Reject - Everyone must approve`
- **Title**: `Material approval required: @{variables('varMaterialName')}`
- **Assigned to**: `varApproverList`
- **Details** (suggested):

  ```text
  A material has passed naming schema validation and needs approval before use.

  Material: @{variables('varMaterialName')}
  Please review the record in Dataverse and approve or reject.

  If rejecting, provide a reason in the comments.
  ```

- **Item link** (optional but helpful):  
  If you have a model-driven app link pattern, paste it here. Otherwise, leave blank.

- **Item link description**: `Open material record`

> Teaching note: “Everyone must approve” ensures both approvers must respond. If you choose “First to respond,” only one person needs to approve.

---

## Part E — Handle approve / reject outcomes (15–20 minutes)

### Step E1: Add a Condition for the approval outcome

Add a **Condition**:

For multiple approvals, you cannot simply compare the **Outcome** to a value as it will be "Approve, Approve, Approve, Reject, ..." depending on number of target reviewers.

Instead, after the **Start an wait for an approval action**, add a **Filter array** action:

Use the **Responses** array from the approval action and filter where **approverResponse** is equal to **Reject**. Then on the condition use `length(body('Filter_array'))` is equal to 0. For the **True** branch (no approver response was **Reject**), handle all approved branch. Otherwise, at least one rejected and that can be handled in the **False** branch.

- Left: **Outcome** (dynamic content from approval action)
- Operator: **is equal to**
- Right: `Approve`

You will build **two branches**:

- **If yes** → Approved path
- **If no** → Rejected path (includes Reject and any non-Approve outcome)

---

### Step E2: Approved path (If yes)

#### 1) Update the Dataverse row as Approved

Add **Dataverse** → **Update a row**

- **Table name**: `Materials`
- **Row ID**: `varMaterialId`
- Set:
  - **Approval Status** = `Approved`
  - **Approved On** = `utcNow()`
  - **Approver Comments** = **Responses Comments** (dynamic content from approval)
  - **Approver Emails** = `varApproverList`

#### 2) Send a notification email

Add **Office 365 Outlook** → **Send an email (V2)**

- **To**: `varNotifyTo`
- **Subject**: `Material approved: @{variables('varMaterialName')}`
- **Body** (example):

  ```html
  <p>A material was approved and is ready for downstream use.</p>
  <ul>
    <li><b>Material</b>: @{variables('varMaterialName')}</li>
    <li><b>Approved On (UTC)</b>: @{utcNow()}</li>
    <li><b>Approvers</b>: @{variables('varApproverList')}</li>
  </ul>
  <p>You may now proceed with onboarding steps.</p>
  ```

---

### Step E3: Rejected path (If no)

#### 1) Update the Dataverse row as Rejected

Add **Dataverse** → **Update a row**

- **Table name**: `Materials`
- **Row ID**: `varMaterialId`
- Set:
  - **Approval Status** = `Rejected`
  - **Approved On** = `utcNow()` *(optional: still track completion time)*
  - **Approver Comments** = **Responses Comments** (dynamic content)
  - **Approver Emails** = `varApproverList`

#### 2) Send a rejection notification email

Add **Office 365 Outlook** → **Send an email (V2)**

- **To**: `varNotifyTo`
- **Subject**: `Material rejected: @{variables('varMaterialName')}`
- **Body** (example):

  ```html
  <p>A material was <b>rejected</b> during approval.</p>
  <ul>
    <li><b>Material</b>: @{variables('varMaterialName')}</li>
    <li><b>Rejected On (UTC)</b>: @{utcNow()}</li>
    <li><b>Approvers</b>: @{variables('varApproverList')}</li>
    <li><b>Comments</b>: @{outputs('Start_and_wait_for_an_approval')?['body/responses/comments']}</li>
  </ul>
  <p>Please correct issues and re-submit for approval (set Approval Status back to Pending if required by your process).</p>
  ```

> Note: The exact dynamic content path for comments can vary. If you can’t find it in the picker, use the **Responses Comments** token if available. Otherwise, inspect the approval action output in run history.

---

## Part F — Test the end-to-end workflow (10–15 minutes)

### Test 1: Approval request fires when a record becomes Valid

1. Choose a Materials record with **Approval Status = Pending**.
2. Change **Validation Status** to **Valid** (or trigger your first validation flow to set it to Valid).
3. Confirm:
   - The approval request appears for the two approvers (Approvals center / email / Teams depending on tenant settings).
   - The record is stamped with **Approval Requested On**.

### Test 2: Both approvers approve

1. Approver 1 selects **Approve**.
2. Approver 2 selects **Approve**.
3. Confirm:
   - **Approval Status** becomes **Approved**
   - **Approved On** is populated
   - A notification email is sent to `varNotifyTo`

### Test 3: Rejection path

1. Set **Approval Status** back to **Pending** (for testing) and then toggle Validation Status (e.g., Valid → Pending → Valid) to re-trigger.
2. Have one approver **Reject** with a comment.
3. Confirm:
   - **Approval Status** becomes **Rejected**
   - Comments are saved
   - Rejection email is sent

---

## Troubleshooting tips

- **Flow triggers repeatedly / loops**
  - Ensure your trigger condition includes **Approval Status = Pending**
  - Avoid updating fields that might toggle your trigger criteria back into true
- **Approver never receives request**
  - Confirm Approvals connector is available and enabled
  - Verify the approver emails are valid and licensed for approvals in your tenant
- **Choice values don’t match your trigger condition**
  - Use the “Compose trigger body” method to discover correct logical names and numeric values
- **Emails not sending**
  - Confirm Office 365 Outlook connection is signed in and allowed
  - Use your own email address for `varNotifyTo` during testing

---

## Optional challenge activities (choose 1–3)

1. **Parallel approvals as two separate steps**  
   Build two “Start and wait for an approval” actions (Approver 1 then Approver 2) and require both. Compare this to “Everyone must approve.”

2. **Escalation after timeout**  
   Use “Create an approval” + “Wait for an approval” with a timeout pattern and send an escalation email if no response within X hours.

3. **Prevent approval if Category mismatch exists**  
   Add logic that checks the material code segment (e.g., `RM`) against the **Category** choice and fails fast if mismatched. (This extends the first lab.)

4. **Write to an Approval Queue table**  
   Create a new Dataverse table **Material Approval Queue** and create a row when approval starts and when it completes.

---

## Deliverables checklist

Learners should have:

- [ ] Updated **Materials** table with approval tracking columns
- [ ] Flow **DVC – Approve Valid Materials**
  - [ ] Triggers only when Validation Status becomes Valid and Approval Status is Pending
  - [ ] Sends approval to two approvers
  - [ ] Updates the material row to Approved/Rejected
  - [ ] Sends notification email on completion
- [ ] Run history demonstrating both the Approved and Rejected paths

---

## Cleanup (optional)

- Delete test material records created for this lab
- Disable the approval flow if you don’t need it for future labs
- Keep the approval fields for later labs (exception handling, governance, monitoring, reporting)
