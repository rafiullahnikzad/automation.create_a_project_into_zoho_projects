# 🚀 Zoho CRM → Zoho Projects: Auto Project Creation (Deluge Script)

> Automatically creates a Zoho Projects project from a Zoho CRM Deal, maps all custom fields, resolves multi-user owners, and links everything back to the CRM record.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Connections Required](#connections-required)
- [CRM Field Mapping](#crm-field-mapping)
- [UDF Field Mapping Reference](#udf-field-mapping-reference)
- [How It Works](#how-it-works)
- [Setup Instructions](#setup-instructions)
- [Usage](#usage)
- [Error Handling](#error-handling)
- [Known Limitations](#known-limitations)
- [Author](#author)

---

## Overview

This Deluge automation function bridges **Zoho CRM** and **Zoho Projects**. When triggered on a Deal record, it:

1. Fetches deal data from Zoho CRM
2. Resolves the deal owner to a Zoho Projects user (zpuid)
3. Creates a new project via Zoho Projects **v3 API**
4. Populates all custom UDF fields (text, date, decimal) via **v1 API**
5. Resolves a multi-user "Project Owners" CRM field → Zoho Projects ZUIDs → updates the `emp` field via **v3 PATCH**
6. Associates the project with the CRM Deal natively
7. Writes the project URL and project ID back onto the CRM Deal record

---

## ✨ Features

- ✅ Creates Zoho Projects project directly from a CRM Deal
- ✅ Sets project owner from Deal Owner (resolved via email → zpuid)
- ✅ Maps 20+ custom fields (text, picklist, date, decimal/double)
- ✅ Supports **multi-user lookup field** for Project Owners (CRM → Projects ZUID resolution)
- ✅ Natively associates the project with the CRM Deal
- ✅ Writes project URL and ID back to the Deal for easy navigation
- ✅ Full error handling with non-fatal per-user resolution fallback
- ✅ Uses both Zoho Projects v1 and v3 APIs where appropriate

---

## 🔧 Prerequisites

| Requirement | Detail |
|---|---|
| Zoho CRM | Enterprise plan or above (for functions/workflows) |
| Zoho Projects | Active portal with custom fields configured |
| Deluge | Available in Zoho CRM Workflow Functions |
| Portal ID | Your Zoho Projects Portal ID (see Setup) |

---

## 🔌 Connections Required

You must configure two OAuth connections in **Zoho CRM → Setup → Connections**:

| Connection Name | Service | Scopes Needed |
|---|---|---|
| `zohocrm` | Zoho CRM | `ZohoCRM.modules.deals.READ`, `ZohoCRM.users.READ`, `ZohoCRM.modules.deals.UPDATE` |
| `zoho_projects` | Zoho Projects | `ZohoProjects.projects.ALL`, `ZohoProjects.portals.READ`, `ZohoProjects.users.READ` |

> ⚠️ The connection names in the script (`zohocrm` and `zoho_projects`) must exactly match the names you create in CRM Connections.

---

## 🗂️ CRM Field Mapping

These CRM Deal fields are read by the function:

| CRM API Name | Description |
|---|---|
| `ASEM_Project_Ref` | Used as the project name |
| `Owner` | Deal owner — resolved to Zoho Projects zpuid |
| `Project_Owners` | Multi-user lookup — resolved to Zoho Projects ZUIDs |
| `Account_Name` | Linked account (client) |
| `Stage` | Deal stage (informational check) |
| `Project_Status` | Project status picklist |
| `Client_Po_No` | Client PO number |
| `Client_PO_Date` | Client PO date |
| `Remining_Amount` | Remaining amount |
| `Delivery_Date` | Delivery date |
| `Expected_Shipment` | Expected shipment date |
| `Arrival_at_Warehouse` | Arrival at warehouse date |
| `Arrival_at_Port` | Arrival at port date |
| `Arrival_at_Site` | Arrival at site date |
| `Clients_First_Payment` | Client first payment value |
| `Clients_Second_Payment` | Client second payment value |
| `Clients_Third_Payment` | Client third payment value |
| `Client_Forth_Payment` | Client fourth payment value |
| `Client_order_value_incld_VAT` | Client order value including VAT |
| `Vendor_Amount` | Vendor amount |
| `Vendor_Order_Value_AED` | Vendor order value in AED |
| `Vendor_Order_Value_Euro` | Vendor order value in EUR |
| `Vendor_Order_Value_USD` | Vendor order value in USD |
| `Vendor_Remining_Amount` | Vendor remaining amount |
| `First_Partial_Payment` | First partial payment |
| `Second_Partial_Payment` | Second partial payment |
| `Third_Partial_Payment` | Third partial payment |
| `Forth_Partial_Payment` | Fourth partial payment |

---

## 📐 UDF Field Mapping Reference

Custom fields in Zoho Projects use internal UDF keys. Here is the full mapping:

| UDF Key | Field Type | Business Field |
|---|---|---|
| `UDF_TEXT2` | Text (Integration) | Client Name (Account ID) |
| `UDF_CHAR8` | Picklist | Project Status |
| `UDF_CHAR3` | Text | Client PO Number |
| `UDF_CHAR4` | Text | Remaining Amount |
| `UDF_DATE1` | Date | Client PO Date |
| `UDF_DATE2` | Date | Delivery Date |
| `UDF_DATE3` | Date | Expected Shipment Date |
| `UDF_DATE4` | Date | Arrival at Warehouse |
| `UDF_DATE5` | Date | Arrival at Port |
| `UDF_DATE6` | Date | Arrival at Site |
| `UDF_DOUBLE1` | Decimal | Client Second Payment |
| `UDF_DOUBLE2` | Decimal | Client Fourth Payment |
| `UDF_DOUBLE3` | Decimal | Client Third Payment |
| `UDF_DOUBLE6` | Decimal | Vendor Order Value (AED) |
| `UDF_DOUBLE7` | Decimal | Vendor Order Value (EUR) |
| `UDF_DOUBLE8` | Decimal | Vendor Order Value (USD) |
| `UDF_DOUBLE11` | Decimal | Client First Payment |
| `UDF_DOUBLE12` | Decimal | Client Order Value incl. VAT |
| `UDF_DOUBLE13` | Decimal | Vendor Amount |
| `UDF_DOUBLE14` | Decimal | First Partial Payment |
| `UDF_DOUBLE15` | Decimal | Second Partial Payment |
| `UDF_DOUBLE16` | Decimal | Third Partial Payment |
| `UDF_DOUBLE17` | Fourth Partial Payment |
| `UDF_DOUBLE18` | Decimal | Vendor Remaining Amount |
| `emp` (UDF_MULTIUSER1) | Multi-User | Employee Name / Project Owners |

> 💡 To find your own UDF keys: Zoho Projects → Settings → Custom Fields → hover over a field name.

---

## ⚙️ How It Works

```
CRM Deal (trigger)
        │
        ▼
Fetch Deal Details (zoho.crm.getRecordById)
        │
        ▼
Resolve Deal Owner Email → zpuid (Projects v3 /users/{email})
        │
        ▼
Create Project (Projects v3 POST /projects)
        │
        ├──▶ Update UDF custom fields (Projects v1 POST /projects/{id}/)
        │
        ├──▶ Resolve Project Owners (multi-user):
        │         CRM User ID → email (CRM /users/{id})
        │                     → ZUID (Projects v3 /users/{email})
        │         PATCH emp field (Projects v3 PATCH /projects/{id})
        │
        ├──▶ Associate project with CRM Deal (CRM /Deals/{id}/Zoho_Projects/{projectId})
        │
        └──▶ Update Deal: Zoho_Projects_Link + Zoho_Project_ID
```

### Why Two API Versions?

| API | Used For | Reason |
|---|---|---|
| Zoho Projects **v3** | Create project, user lookup, PATCH multi-user field | v3 supports `owner` (zpuid) on creation and multi-user PATCH |
| Zoho Projects **v1** | Update UDF custom fields (text, date, decimal) | v3 does not yet support UDF field updates |

---

## 🛠️ Setup Instructions

### 1. Get Your Portal ID

Go to **Zoho Projects → Settings → Portal Settings** and copy your Portal ID. Update line 14 of the script:

```deluge
portalId = "YOUR_PORTAL_ID_HERE";
```

### 2. Create OAuth Connections

In **Zoho CRM → Setup → Developer Space → Connections**, create:

- `zohocrm` — Service: Zoho CRM, with read/write scopes on Deals and Users
- `zoho_projects` — Service: Zoho Projects, with project and user scopes

### 3. Configure Custom Fields

Ensure all UDF fields exist in your Zoho Projects portal. Map your own UDF keys by checking **Zoho Projects → Settings → Custom Fields** and update the `updateData.put(...)` lines accordingly.

### 4. Configure CRM Fields

Ensure your CRM Deals module has all the fields listed in the [CRM Field Mapping](#crm-field-mapping) table. Update the `deal.get("field_api_name")` calls to match your own field API names.

### 5. Create the Function in CRM

Go to **Zoho CRM → Setup → Developer Space → Functions → New Function**:

- Category: `Automation`
- Function name: `create_a_project_into_zoho_projects`
- Argument: `dealId` (type: Int)
- Paste the contents of `create_project_zoho_projects.dg`

### 6. Attach to a Workflow / Button

Create a **Workflow Rule** or **Custom Button** on the Deals module that calls this function, passing `${Deals.id}` as the `dealId` argument.

---

## 🚨 Error Handling

The script uses a multi-layer error handling approach:

| Layer | Behavior |
|---|---|
| Global `try/catch` | Catches any unhandled exception and logs it via `info` |
| Project creation check | Validates `projectResponse.get("id")` before proceeding |
| Per-owner `try/catch` | If one Project Owner can't be resolved, logs a warning and continues — other owners are still processed |
| Deal update check | Validates `updateDealResponse.get("id")` and logs success or failure |

All errors are written to the **Zoho CRM Function Logs** (`info` statements). To view logs: **CRM → Setup → Developer Space → Functions → Logs**.

---

## ⚠️ Known Limitations

- **Zoho Projects v3 UDF support**: As of this writing, UDF text/date/decimal fields must still be updated via the v1 API. If Zoho adds v3 UDF support, the Step 7 call can be consolidated.
- **Multi-user field payload**: The `emp` field PATCH requires ZUIDs as `Long` values. If `toLong()` fails, check that the ZUID returned by the Projects user lookup is a valid numeric string.
- **Portal ID is hardcoded**: For multi-portal setups, consider passing `portalId` as a function parameter or storing it in a CRM variable.
- **Deluge loop type**: Deluge only supports `for each` loops — no `while` or `for (int i...)` loops.

---

## 👤 Author

**Rafiullah Nikzad**
Senior Zoho Developer @ CloudZ Technologies

- 🌐 Portfolio: [rafiullahnikzad.netlify.app](https://rafiullahnikzad.netlify.app)
- 💼 LinkedIn Community: [Zoho Afghanistan](https://www.linkedin.com/groups/zoho-afghanistan)
- 📦 Specialization: Zoho CRM · Zoho Creator · Zoho Projects · Deluge Scripting · API Integrations

---

## 📄 License

MIT License — feel free to use, fork, and adapt with attribution.
