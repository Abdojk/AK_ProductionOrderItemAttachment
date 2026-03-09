# CLAUDE.md — AK_ProdOrderItemsAttachments

## Project Identity

| Field | Value |
|---|---|
| **Project Name** | AK_ProdOrderItemsAttachments |
| **Model Name** | AK_ProdOrderItemsAttachmentsExt |
| **GitHub Repo** | AK_ProdOrderItemsAttachments |
| **D365 Module** | Production Control |
| **Solution Type** | Tier C — Partial OOB + Customization (Extensions Only) |
| **Language** | X++ |
| **Author** | Abdo (Abdojk) |

---

## Business Purpose

Provide production and engineering users with a centralized, multi-record-based screen to view all technical attachments (drawings, specifications, documents) linked to configured production items — without opening each item individually.

**Core Flow:**
1. User opens `AK_ProdOrderItemsAttachments` form (standalone, Production Control menu)
2. Grid shows production order lines with item + configuration attributes
3. User selects one or multiple lines → clicks **Attachments** button
4. Custom viewer form opens, listing all attachments for selected items with full item context visible

---

## Data Model — Source of Truth

### Primary Join Chain

```
ProdTable (Production Order)
  └── ProdBOM (BOM/Component Lines)  [ProdBOM.ProdId = ProdTable.ProdId]
        └── InventTable (Item Master)  [ProdBOM.ItemId = InventTable.ItemId]
              └── InventDim (Inventory Dimensions)  [ProdBOM.InventDimId = InventDim.inventDimId]
                    └── EcoResConfiguration (Configuration)  [InventDim.configId = EcoResConfiguration.Name]
                          └── EcoResConfiguration (Extended)  ← CUSTOM FIELDS LIVE HERE
```

### Custom Fields on EcoResConfiguration (via Extension)

| Field Label | AOT Field Name | Notes |
|---|---|---|
| Thickness | `AK_Thickness` | Confirm exact name via Ctrl+Alt+F5 |
| Material Type | `AK_MaterialType` | Confirm exact name via Ctrl+Alt+F5 |
| Document Type | `AK_DocumentType` | Confirm exact name via Ctrl+Alt+F5 |
| Template | `AK_Template` | Confirm exact name via Ctrl+Alt+F5 |

> ⚠️ **OVERWATCH RULE**: Do NOT assume the exact field names above. Before writing any code that references these fields, open D365, navigate to `EcoResConfiguration`, press **Ctrl+Alt+F5**, and confirm the actual AOT field names. Update this section accordingly.

### Attachments Source

Attachments are stored in `DocuRef`. For item-level attachments:

```
DocuRef.RefTableId = tableNum(InventTable)
DocuRef.RefRecId   = InventTable.RecId (for each selected item)
DocuRef.RefCompanyId = curExt()
```

---

## AOT Object Inventory

### Objects to Create (All Extensions or Net-New)

| AOT Object | Name | Type | Purpose |
|---|---|---|---|
| Query | `AK_ProdOrderItemsAttachmentsQuery` | Query | Joins ProdTable → ProdBOM → InventTable → InventDim → EcoResConfiguration |
| Form | `AK_ProdOrderItemsAttachments` | Form | Main grid screen with Attachments button |
| Form | `AK_ProdOrderItemsAttachmentsViewer` | Form | Attachment detail view with item context |
| Class | `AK_ProdOrderItemsAttachmentsController` | Class | Business logic: collect selected items, query DocuRef, pass to viewer |
| Menu Item (Display) | `AK_ProdOrderItemsAttachments` | DisplayMenuItem | Entry point from Production Control menu |
| Security Privilege | `AK_ProdOrderItemsAttachmentsView` | SecurityPrivilege | View access |
| Security Duty | `AK_ProdOrderItemsAttachmentsMaintain` | SecurityDuty | Assigned to relevant role |

### Objects to Extend (Never Modify Base)

| Base Object | Extension Name | Purpose |
|---|---|---|
| Menu `ProdControl` (or sub-menu) | `AK_ProdControl.Extension` | Add new display menu item |

---

## File & Folder Structure

```
AK_ProdOrderItemsAttachmentsExt/
├── CLAUDE.md                          ← This file (project brain)
├── SESSION_LOG.md                     ← Running log of Claude Code sessions
├── AK_ProdOrderItemsAttachmentsExt/
│   ├── Descriptor/
│   │   └── AK_ProdOrderItemsAttachmentsExt.xml
│   ├── Queries/
│   │   └── AK_ProdOrderItemsAttachmentsQuery.xml
│   ├── Forms/
│   │   ├── AK_ProdOrderItemsAttachments.xml
│   │   └── AK_ProdOrderItemsAttachmentsViewer.xml
│   ├── Classes/
│   │   └── AK_ProdOrderItemsAttachmentsController.xml
│   ├── MenuItems/
│   │   └── Display/
│   │       └── AK_ProdOrderItemsAttachments.xml
│   └── Security/
│       ├── Privileges/
│       │   └── AK_ProdOrderItemsAttachmentsView.xml
│       └── Duties/
│           └── AK_ProdOrderItemsAttachmentsMaintain.xml
```

---

## Coding Standards & Conventions

- **Extensions only** — never modify base D365 objects
- **Naming prefix** — all custom objects prefixed `AK_`
- **No inline business logic in forms** — delegate to `AK_ProdOrderItemsAttachmentsController`
- **Multi-select** — use `MultiSelectionHelper` class to iterate selected records on the main grid
- **DocuRef queries** — always filter by `RefCompanyId = curExt()` to respect legal entity scope
- **No hardcoded strings** — use Labels (`@AK_001`, etc.) for all UI-visible text
- **Error handling** — if no attachments found, show `info()` message, do not throw error
- **Security** — all form access must go through the privilege/duty chain; no `AllowDuplicates` shortcuts

---

## Implementation Sequence (Phases)

### Phase 1 — Query
Build `AK_ProdOrderItemsAttachmentsQuery` with the full join chain. Validate in AOT before proceeding.

### Phase 2 — Main Form (`AK_ProdOrderItemsAttachments`)
- Data source: `AK_ProdOrderItemsAttachmentsQuery`
- Grid columns: ProdId, ItemId, AK_Thickness, AK_MaterialType, AK_DocumentType, AK_Template
- Multi-select enabled on grid
- Action Pane: **Attachments** button wired to `AK_ProdOrderItemsAttachmentsController::openAttachments()`

### Phase 3 — Controller Class (`AK_ProdOrderItemsAttachmentsController`)
```xpp
// Pseudocode — do not generate literal code until Phase 3 is active
public static void openAttachments(FormDataSource _fds)
{
    // 1. Use MultiSelectionHelper to get all selected records
    // 2. Collect Set of InventTable RecIds
    // 3. Query DocuRef where RefTableId = tableNum(InventTable) AND RefRecId IN collected set
    // 4. Pass result container to AK_ProdOrderItemsAttachmentsViewer form
}
```

### Phase 4 — Viewer Form (`AK_ProdOrderItemsAttachmentsViewer`)
- Temp table or in-memory datasource populated by controller
- Columns: ItemId, ProdId, Document Name, Document Type, File Name, Notes, Created Date
- **View** button per row → calls `DocuAction` or opens attachment inline
- Read-only grid (no editing)
- Close button returns user to main form

### Phase 5 — Menu Item & Menu Extension
- Create Display Menu Item pointing to `AK_ProdOrderItemsAttachments`
- Extend Production Control menu (confirm exact menu AOT name via Ctrl+Alt+F5)

### Phase 6 — Security
- Privilege: `AK_ProdOrderItemsAttachmentsView` — grants Read on form + menu item
- Duty: `AK_ProdOrderItemsAttachmentsMaintain` — wraps the privilege
- Assign duty to the appropriate production role (confirm role name with client)

---

## OVERWATCH Protocol — Run Before Every Commit

Challenge every generated output with these 12 questions before accepting:

1. Are all referenced table names verified against actual D365 AOT names (not assumed)?
2. Are all custom field names on `EcoResConfiguration` confirmed via Ctrl+Alt+F5?
3. Is the join chain `ProdTable → ProdBOM → InventDim → EcoResConfiguration` valid and complete?
4. Does the `DocuRef` query filter by `RefCompanyId = curExt()`?
5. Is `MultiSelectionHelper` used correctly (not just reading the active record)?
6. Are all string literals replaced with Labels?
7. Is there a graceful no-results path (info message, not error)?
8. Does the viewer form receive data via a temp table or container — not a direct table datasource?
9. Are extension objects named with `AK_` prefix and never modifying base objects?
10. Is security wired correctly — Privilege → Duty — not directly on the form?
11. Does the menu item extension target the correct base menu (verified, not assumed)?
12. Has the Descriptor XML been updated with the correct model name and publisher?

> **OVERWATCH verdict**: If ANY answer is "No" or "Unsure" → STOP. Resolve before generating further code.

---

## Guardrails — What Claude Code Must NOT Do

- ❌ Do NOT modify `ProdBOM`, `InventTable`, `EcoResConfiguration`, `DocuRef`, or any base table
- ❌ Do NOT assume field names on `EcoResConfiguration` — always verify
- ❌ Do NOT use `DocuRef` without `RefCompanyId` filter
- ❌ Do NOT put business logic directly in form methods — use the controller class
- ❌ Do NOT create a new table to store attachments — use the existing `DocuRef` framework
- ❌ Do NOT skip the Query object — do not embed joins directly in form datasource code
- ❌ Do NOT generate all phases at once — execute one phase at a time with review between

---

## Reusable Prompt Templates

### Add a New Column to the Main Grid
```
TASK: Add a new column to the AK_ProdOrderItemsAttachments main grid.
Column source: [TABLE].[FIELD] — confirm field exists via Ctrl+Alt+F5 before proceeding.
Steps:
1. Add the field/datasource to AK_ProdOrderItemsAttachmentsQuery.xml
2. Add a grid control bound to it in AK_ProdOrderItemsAttachments.xml
3. Run OVERWATCH check before outputting.
Do NOT modify any base object. Use extensions only.
```

### Debug: Attachments Not Showing
```
TASK: Investigate why attachments are not appearing in AK_ProdOrderItemsAttachmentsViewer.
Check:
1. DocuRef query — is RefTableId = tableNum(InventTable)? Print actual value.
2. Is RefRecId matching the correct InventTable.RecId for selected items?
3. Is RefCompanyId = curExt()?
4. Are there actually attachments on these items in standard DocuRef view?
Output a diagnostic X++ job that prints DocuRef count for a given ItemId. Do not modify production code.
```

### Add a New Security Role Assignment
```
TASK: Assign AK_ProdOrderItemsAttachmentsMaintain duty to [ROLE_NAME].
Confirm [ROLE_NAME] AOT name via Ctrl+Alt+F5 before proceeding.
Create a role extension: [ROLE_NAME].Extension
Add duty reference only — do not modify role permissions or base role XML.
Run OVERWATCH items 10–12 before output.
```
