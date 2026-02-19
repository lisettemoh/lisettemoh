# ZFI\_INVOICE\_STATE — Kinetic Setup Guide

**Migrated from:** SAP ABAP report `ZFI_INVOICE_STATE`
**Purpose:** Invoices by Province or State — Finance team AR invoice review by region and date range
**Deliverables in this folder:**

| File | Purpose |
|------|---------|
| `ZFI_INVOICE_STATE.rdl` | SSRS report layout — upload in Report Style step |
| `ZFI_INVOICE_STATE_Setup_Guide.md` | This document |

---

## Prerequisites

- [ ] BAQ `ZFI-InvoiceByState` built and tested — **see Part 0 below for full build instructions**
- [ ] Confirm Kinetic invoice type codes that map to SAP types G2, S1, ZRE (needed for the
  BAQ calculated field `NetValue` sign reversal — see Note A at the bottom)
- [ ] SSRS Report Server deployed and reachable
- [ ] Your Kinetic user has **System Manager** or **Report Administrator** security access
- [ ] Microsoft Report Builder or Visual Studio with SSRS extension installed (optional —
  only needed if you want to preview the .rdl before uploading)

---

## Part 0 — Build the BAQ

> **⚠ Correction from earlier notes:** There is no `StateProvince` table in this Kinetic
> environment. Do **not** add a `StateProvince` join. The state/province value comes
> directly from the `Customer` table.

### 0.1 Open BAQ Designer

`System Setup → Business Activity Queries → Business Activity Query`

Click **New**, set:

| Field | Value |
|-------|-------|
| BAQ ID | `ZFI-InvoiceByState` |
| Description | `Invoices by Province or State` |
| Type | `Updatable BAQ` → No / **Standard BAQ** |
| Company | Your company ID |

Click **Save**, then open the BAQ for editing.

---

### 0.2 Add Tables

Add **two** tables in the following order:

| # | Table | Role |
|---|-------|------|
| 1 | `InvcHead` | Primary — invoice header |
| 2 | `Customer` | Joined — customer name and bill-to state |

#### Join: InvcHead → Customer

| Setting | Value |
|---------|-------|
| Join Type | **Inner Join** |
| Left table | `InvcHead` |
| Right table | `Customer` |
| Condition 1 | `InvcHead.Company = Customer.Company` |
| Condition 2 | `InvcHead.CustNum = Customer.CustNum` |

---

### 0.3 Add Fields and Set Aliases

On the **Fields** tab, add each field below and set the alias exactly as shown
(the `.rdl` file references these aliases by name):

| Table | Field | **Alias (required exact match)** | Notes |
|-------|-------|----------------------------------|-------|
| `Customer` | `BTState` | `StateDesc` | Bill-to state/province code (e.g. `ON`, `BC`) |
| `Customer` | `CustID` | `CustID` | Customer code |
| `Customer` | `Name` | `CustName` | Customer name |
| `InvcHead` | `InvoiceType` | `InvoiceType` | Invoice type code |
| `InvcHead` | `InvoiceNum` | `InvoiceNum` | Invoice number |
| `InvcHead` | `InvoiceDate` | `InvoiceDate` | Invoice date |
| Calculated | *(see 0.4)* | `NetValue` | Sign-corrected invoice amount |
| `InvcHead` | `CurrencyCode` | `CurrencyCode` | Currency |

> **Why `BTState` and alias `StateDesc`?**
> `Customer.BTState` is the bill-to address state — the correct field for AR/Finance
> invoice reporting. The alias `StateDesc` is kept to match the `.rdl` dataset field name.
> The value will be the state **code** (e.g. `ON`), not a full description, because there
> is no state name lookup table in this environment.

---

### 0.4 Calculated Field — NetValue

In the **Fields** tab, add a **Calculated Field**:

- **Alias:** `NetValue`
- **Data Type:** `Decimal`
- **Expression:**

```sql
CASE
  WHEN InvcHead.InvoiceType IN ('CM', 'REI')   -- update codes after confirming — see Note A
  THEN InvcHead.InvoiceAmt * -1
  ELSE InvcHead.InvoiceAmt
END
```

> Confirm the exact `InvoiceType` codes for credit memos and reversals in your system
> before finalising this expression. See **Note A** at the bottom of this guide.

---

### 0.5 Add BAQ Parameters

On the **Criteria** tab, add three parameters:

| Parameter | Operator | Field | Data Type |
|-----------|----------|-------|-----------|
| `pState` | `=` | `Customer.BTState` | String |
| `pDateFrom` | `>=` | `InvcHead.InvoiceDate` | DateTime |
| `pDateTo` | `<=` | `InvcHead.InvoiceDate` | DateTime |

For each:
1. Click **Add Criteria**
2. Select the table/field
3. Set the operator
4. Check **Parameter** and enter the parameter name exactly as shown above

---

### 0.6 Save and Test the BAQ

1. Click **Save**
2. Click **Test** — enter a valid bill-to state code (e.g. `ON`) and a narrow date range
3. Confirm rows return with correct data
4. Confirm a credit memo row exists and has a negative `NetValue`

---

---

## Part 1 — BAQ Field Aliases (Confirm Before Proceeding)

The `.rdl` file references dataset field names by the **aliases you set in BAQ Designer**.
Open BAQ `ZFI-InvoiceByState`, go to the **Fields** tab, and verify each alias matches exactly:

| BAQ Table / Source | Field | **Required Alias** |
|-------------------|-------|--------------------|
| `Customer.BTState` | Bill-to state/province code | `StateDesc` |
| `Customer.CustID` | Customer code | `CustID` |
| `Customer.Name` | Customer name | `CustName` |
| `InvcHead.InvoiceType` | Invoice type code | `InvoiceType` |
| `InvcHead.InvoiceNum` | Invoice number | `InvoiceNum` |
| `InvcHead.InvoiceDate` | Invoice date | `InvoiceDate` |
| Calculated field (sign-corrected amount) | Net value | `NetValue` |
| `InvcHead.CurrencyCode` | Currency | `CurrencyCode` |

> **Note:** `StateDesc` contains the state **code** (e.g. `ON`, `BC`, `AB`), not a
> full description. There is no `StateProvince` table in this environment — do not add one.
> If the Finance team needs full province/state names in the report, a manual lookup list
> can be added as a static dataset inside the `.rdl` at a later stage.
>
> If any alias differs, either rename it in the BAQ or do a find-and-replace in the `.rdl`
> file before uploading.

---

## Part 2 — Report Data Definition (RDD)

The RDD is the bridge between the BAQ and the SSRS report style. It tells Kinetic which BAQ
to run, what the output dataset is called, and what parameters exist.

### 2.1 Open RDD Designer

`System Setup → Business Activity Queries → Report Data Definition`

### 2.2 Create New RDD

1. Click **New**
2. Fill in:

   | Field | Value |
   |-------|-------|
   | Report ID | `ZFI-InvoiceByState` |
   | Description | `Invoices by Province or State` |
   | Company | Your company ID |

3. Click **Save** (do not close yet)

### 2.3 Link the BAQ

1. On the **General** tab, find the **BAQ ID** field
2. Select `ZFI-InvoiceByState` (the BAQ you built)
3. The system will auto-populate the dataset fields below — click **Get Fields** if it
   does not do so automatically

### 2.4 Verify Dataset Fields

Go to the **Fields** tab. Confirm all 8 fields appear with the correct aliases:

`StateDesc, CustID, CustName, InvoiceType, InvoiceNum, InvoiceDate, NetValue, CurrencyCode`

If any are missing, return to the BAQ and confirm the aliases match Part 1 above.

### 2.5 Define Report Parameters

Go to the **Parameters** tab. Add three parameters — they must match the BAQ parameters
exactly (including case):

| Parameter Name | Data Type | Label | Required |
|---------------|-----------|-------|----------|
| `pState` | String | State / Province Code | Yes (enforced at menu layer) |
| `pDateFrom` | DateTime | Invoice Date From | Yes |
| `pDateTo` | DateTime | Invoice Date To | Yes |

For each parameter:
1. Click **Add**
2. Enter the **Parameter Name** exactly as shown
3. Set **Data Type**
4. Enter the **Label** (shown to the user at runtime)
5. Leave **Default Value** blank (the Finance team will enter these each run)

### 2.6 Save and Deploy the RDD

1. Click **Save**
2. Click **Actions → Generate Report Data Definition** (or the equivalent deploy button
   in your Kinetic version — terminology varies between 2022 and 2024 releases)
3. Confirm the success message

---

## Part 3 — Report Style

The Report Style links the RDD to the physical `.rdl` file and registers it on the SSRS
report server.

### 3.1 Open Report Style

`System Setup → Business Activity Queries → Report Style`

### 3.2 Create New Report Style

1. Click **New**
2. Fill in:

   | Field | Value |
   |-------|-------|
   | Report ID | `ZFI-InvoiceByState` |
   | Style Description | `Standard — Invoices by Province or State` |
   | Report Data Definition | `ZFI-InvoiceByState` (select from lookup) |
   | Style Number | `1` |
   | Default Style | Checked |
   | Output Format | `PDF` (Finance default — users can override at runtime) |

3. Click **Save** (do not close)

### 3.3 Upload the RDL File

1. On the Report Style record, find the **Report** section or the **Upload** button
   (exact label varies by Kinetic version — look for "Upload Report File" or "RDL File")
2. Click **Upload**
3. Browse to and select `ZFI_INVOICE_STATE.rdl` from this folder
4. The system will deploy the file to the SSRS report server automatically
5. Confirm the upload success message

> **If your Kinetic version does not have a built-in uploader:**
> Copy `ZFI_INVOICE_STATE.rdl` manually to the SSRS report server folder configured for
> Kinetic reports (typically `C:\Inetpub\wwwroot\Reports\<company>\` or the path shown in
> Kinetic's System Agent settings). Then enter the relative path in the **RDL Path** field
> on the Report Style.

### 3.4 Set Page and Output Options

Still on the Report Style record:

| Setting | Value |
|---------|-------|
| Page Orientation | Landscape |
| Paper Size | Letter (8.5" × 11") |
| Allow User Output Format Override | Checked (so users can choose PDF vs Excel) |

### 3.5 Save

Click **Save**.

---

## Part 4 — Menu Maintenance

This adds the report to the Kinetic menu so Finance can run it without going through BAQ
or Report Style screens.

### 4.1 Open Menu Maintenance

`System Setup → System → Menu Maintenance`

### 4.2 Locate the Finance Parent Menu

In the menu tree on the left, navigate to the Finance module parent. Common paths:

- `Finance → Accounts Receivable → Reports`
- `Finance → Reports`

If a Reports subfolder does not exist under Accounts Receivable, you can create one (see
step 4.5) or add the item directly under Accounts Receivable.

### 4.3 Add New Menu Item

1. Select the parent folder in the tree
2. Click **New Child** (or **Add → New Menu Item**)
3. Fill in:

   | Field | Value |
   |-------|-------|
   | Menu ID | `ZFI-InvoiceByState` |
   | Menu Description | `Invoices by Province or State` |
   | Type | `Report` |
   | Report ID | `ZFI-InvoiceByState` (select from lookup — this is the Report Style ID) |
   | Sequence | Set so it appears in the correct position in the list (e.g., 50) |
   | Active | Checked |

4. Click **Save**

### 4.4 Assign Security Group

1. On the menu item, go to the **Security** tab
2. Add the Security Group(s) for the Finance team
3. Set permission level to **View** (read-only execution is sufficient for a report)
4. Click **Save**

### 4.5 Optional — Create a Reports Subfolder

If you want to group Finance reports under a dedicated submenu:

1. Select the `Accounts Receivable` parent in the tree
2. Click **New Child**
3. Set Type to `Menu` (a folder, not a report)
4. Description: `Reports`
5. Save, then add the `ZFI-InvoiceByState` item as a child of this new folder

---

## Part 5 — Running the Report

Once the menu item is live, Finance runs the report as follows:

1. Navigate to the menu item: `Finance → Accounts Receivable → Reports → Invoices by Province or State`
2. The parameter prompt screen appears:

   | Prompt | Notes |
   |--------|-------|
   | State / Province Code | Enter the Kinetic state code (e.g., `ON` for Ontario, `BC` for British Columbia) |
   | Invoice Date From | Start of date range |
   | Invoice Date To | End of date range |

3. Click **Print** or **Preview**
4. The report opens as PDF (or the user's chosen format)

---

## Part 6 — Testing Checklist

Run through each item before handing to Finance.

### Data validation
- [ ] Run for a known state with a narrow date range — count rows matches what you see
  in a direct database query
- [ ] Confirm a credit memo (CM type) shows as a **negative** NetValue
- [ ] Confirm a standard invoice shows as a **positive** NetValue
- [ ] Confirm negative values render in **red**
- [ ] Confirm state subtotals match the sum of detail rows for that state
- [ ] Confirm the Grand Total matches the sum of all state subtotals

### Layout validation
- [ ] Column headers repeat on every page (RepeatOnNewPage is set in the RDL)
- [ ] Page header shows report title, run date/time, and selected parameters
- [ ] Page footer shows "Page X of Y"
- [ ] State subtotal rows appear in **light steel blue**
- [ ] Grand Total row appears in **navy** with white text
- [ ] Report is landscape, fits on letter paper without truncation

### Parameter validation
- [ ] Entering an invalid state code returns zero rows (not an error)
- [ ] Entering a date range with no invoices returns zero rows gracefully
- [ ] pDateFrom > pDateTo — add a validation message at the dashboard or menu layer
  (the BAQ itself does not enforce this)

### Access validation
- [ ] Finance team members can see and run the menu item
- [ ] Non-Finance users cannot see the menu item

---

## Note A — Invoice Type Code Mapping (Action Required)

The ABAP report applied a **sign reversal** (negated the amount) for SAP types `G2`, `S1`,
and `ZRE`. The BAQ calculated field `NetValue` must do the same in Kinetic. The Kinetic
type codes are **not** the same as SAP.

Before the report goes live, pull up one example of each type in Kinetic and record the
`InvcHead.InvoiceType` value, then update the CASE expression in the BAQ:

| SAP Type | SAP Meaning | Kinetic InvoiceType | Sign |
|----------|------------|---------------------|------|
| F2 | Standard invoice | *(confirm)* | Positive (no change) |
| G2 | Credit memo | *(confirm — typically `CM`)* | **Negative** |
| L2 | Debit memo | *(confirm)* | Positive (no change) |
| S1 | Cancellation/reversal | *(confirm)* | **Negative** |
| ZRE | Returns credit | *(confirm — may also be `CM`)* | **Negative** |

**BAQ calculated field expression (update the type codes in bold):**

```sql
CASE
  WHEN InvcHead.InvoiceType IN ('CM', 'REI')   -- replace with confirmed Kinetic codes
  THEN InvcHead.InvoiceAmt * -1
  ELSE InvcHead.InvoiceAmt
END
```

---

## Note B — "Previous Month" Default Date Range

The original ABAP report auto-filled the date range to the previous calendar month on
startup. BAQ parameters do not support computed defaults. Replicate this behaviour by
adding the report to a **Dashboard** and setting the parameter default expressions there:

| Parameter | Dashboard default expression |
|-----------|------------------------------|
| `pDateFrom` | `=DateSerial(Year(DateAdd("m",-1,Now())), Month(DateAdd("m",-1,Now())), 1)` |
| `pDateTo` | `=DateSerial(Year(Now()), Month(Now()), 0)` |

This is optional — Finance can also enter the dates manually each run.

---

## Note C — Country Scope and State Field Choice

The original ABAP report was hardcoded to country `CA` (Canada). In this Kinetic BAQ
there is no `StateProvince` table join, so there is no automatic country scope restriction.

**This BAQ uses `Customer.BTState`** (the bill-to address state). The `Customer` table
also contains `Customer.State` (the ship-to / main address state). Available fields:

| Field | Meaning | Used in this BAQ? |
|-------|---------|-------------------|
| `Customer.BTState` | Bill-to (invoice) address state | ✅ Yes |
| `Customer.State` | Ship-to / main address state | No |

If your Kinetic data includes customers in multiple countries and state codes overlap
(e.g. `ON` exists in both Canada and Nigeria), add this additional criteria to the BAQ
to restrict to Canadian customers only:

`Customer.Country = 'CA'`  *(or whichever value your Country field holds for Canada —
check the `Country` table in BAQ Designer or confirm with your Kinetic admin)*

> The `Country` table in this environment also exposes `State` and `BTState` fields.
> These are **not** used in this BAQ — the state value is drawn directly from `Customer.BTState`.
