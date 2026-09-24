# Intelligent Document Processing (IDP) on Databricks

An end-to-end pipeline that turns a folder of unstructured PDF documents (invoices, purchase orders, and receipts) into clean, queryable, governed tables, entirely in Databricks SQL, using Unity Catalog and Databricks' built-in AI functions.

No custom ML models, no training, no external services. Just SQL.

---

## What it does

Point it at a volume full of mixed PDFs and it will:

1. **Parse** each PDF into structured text.
2. **Classify** every document as an Invoice, Purchase Order, Receipt, or Other.
3. **Extract** the fields relevant to each document type.
4. **Flatten** the results into clean columns and save them as governed Unity Catalog tables.

The result is three tidy tables : `idp.finance.invoices`, `idp.finance.purchase_order`, and `idp.finance.receipt` : that anyone with the right permissions can query directly.

---

## Architecture

```
PDFs in a Volume  (/Volumes/idp/default/final_project)
        │
        ▼
[ ai_parse_document ]  →  parsed_data      (raw structured parse)
        │
        ▼
[ flatten elements  ]  →  pretty_data      (one clean doc_text per file)
        │
        ▼
[ ai_classify       ]  →  classified_data  (adds doc_classification label)
        │
        ├── WHERE 'Invoice'        → ai_extract → invoice_data        → idp.finance.invoices
        ├── WHERE 'Purchase Order' → ai_extract → purchase_order_data → idp.finance.purchase_order
        └── WHERE 'Receipt'        → ai_extract → receipt             → idp.finance.receipt
```

---

## Prerequisites

- A Databricks workspace with **Unity Catalog** enabled.
- A running **serverless SQL warehouse** (the pipeline is SQL; a SQL warehouse is the right compute).
- A Unity Catalog **catalog** named `idp` with a **volume** at `/Volumes/idp/default/final_project` containing your source PDFs.
- Access to Databricks **AI Functions** (`ai_parse_document`, `ai_classify`, `ai_extract`). These call Databricks-hosted models behind the scenes — no GPU compute required on your side.

---

## Unity Catalog layout

```
idp                              (catalog)
├── default                      (schema)
│   └── Volumes
│       └── final_project        ← source PDFs live here
├── finance                      (schema)
│   └── Tables
│       ├── invoices             ← final invoice output
│       ├── purchase_order       ← final PO output
│       └── receipt              ← final receipt output
└── information_schema           (auto-created metadata)
```

---

## Pipeline stages

**1. Read the source files.** Lists the PDFs in the volume to confirm they're visible.

**2. Parse the documents.** `ai_parse_document` converts each PDF into a structured representation (a VARIANT holding the document's elements) and stores it in `parsed_data`.

**3. Flatten into clean text.** Walks every parsed element, pulls out its text, and joins them with newlines into a single readable `doc_text` per document (`pretty_data`). Empty elements such as images are safely handled so they don't break the join.

**4. Classify each document.** `ai_classify` reads the text and tags each document as Invoice, Purchase Order, Receipt, or Other (`classified_data`). The Other bucket catches anything that doesn't fit the three target types.

**5. Extract fields per document type.** Each type gets its own extract, filtered by classification and asking for type-appropriate fields:
- **Invoices** → vendor name, invoice number, invoice date, due date, payment method, total
- **Purchase Orders** → merchant name, PO number, PO date, total
- **Receipts** → merchant name, receipt number, transaction date, total

`ai_extract` returns each requested field as a sub-field inside an `extracted` struct. Field names act as semantic instructions — the model maps them to the right content even when the document uses different wording.

**6. Flatten and save to governed tables.** The `extracted` struct is unpacked into real, named columns and written to `idp.finance.invoices`, `idp.finance.purchase_order`, and `idp.finance.receipt`. Every table is created with a replace-if-exists pattern, so the whole notebook can be rerun safely as more documents are added.

---

## How to run

1. Upload PDFs to the volume `/Volumes/idp/default/final_project`.
2. Attach the notebook to a **serverless SQL warehouse**.
3. Run the cells top to bottom.
4. Query the final tables in `idp.finance` to see the structured results.

Because every table is rebuilt on each run, the pipeline is **idempotent** — add more PDFs to the volume and rerun, and the tables rebuild cleanly from the full set. (This project was validated exactly that way: run on an initial batch, then rerun after adding more files.)

---

## AI functions used

| Function | Role |
|---|---|
| `ai_parse_document` | Converts a PDF into structured, machine-readable elements |
| `ai_classify` | Assigns each document to one label from a fixed list |
| `ai_extract` | Pulls named fields out of free-form text into a struct |

---

## Notes & possible improvements

- **Naming consistency:** the final tables mix plural and singular (`invoices` vs `purchase_order` / `receipt`). Pick one convention if it matters to you.
- **The Other bucket** is intentionally not extracted or saved — documents that don't fit the three types stay in `classified_data` only.
- **Intermediate tables** (`parsed_data`, `pretty_data`, `classified_data`, and the per-type `*_data` tables) are staging tables; only the `idp.finance.*` tables are the final deliverables. You could drop the staging tables afterward, or convert the flow into a view-based / DLT pipeline for automatic refresh.
- **Data quality:** consider adding validation (e.g. flag invoices with a null total or an unparseable date) before the data is consumed downstream.
- **Governance:** grant read access on the `idp.finance` schema to your finance team, and check the **Lineage** tab in Catalog Explorer to see the full parse → classify → extract chain visualized automatically.

---

## Tech stack

- **Databricks** (Free Edition / workspace)
- **Unity Catalog** for governance and the three-level namespace
- **Serverless SQL Warehouse** for compute
- **Databricks AI Functions** for parsing, classification, and extraction
- **Delta tables** as the storage format
