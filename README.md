# Pool People Apps Script Architecture

This directory contains the Pool People Google Apps Script applications and shared data-platform projects. Project-level README files define local responsibilities; this file records cross-project architecture that must remain consistent across applications.

## Sales-Tax Reconciliation Architecture

The production sales-tax design separates source acquisition, immutable evidence, Level 1 reconciliation, and Level 2 variance explanation. The retired App 05 Phase 9 three-source bridge is diagnostic/research history and is not the production reconciliation pipeline.

```text
QBO
│
├── TaxableSalesDetail API
│        │
│        ▼
│   50 QBO Import Hub Standalone — permanent TaxableSalesDetail source
│        │
│        ├── 00_Controls
│        ├── 01_Current
│        ├── 02_Detail_Snapshots
│        ├── 03_Snapshot_Changes
│        └── 90_Ingestion_Log
│        │
│        └── QBO's current cash-basis reconstruction of the requested
│            reporting period, with immutable Snapshot Records preserving the state observed at each pull ASOF
│        │
│        ▼
│   21 Bookkeeping Audit — Level 2 reconciliation / explanation
│
└── General Ledger API
         │
         ▼
    50 QBO Import Hub Standalone — existing governed GL export
    QBO_Export_General_Ledger_Report
         │
         └── carries its own capture / refresh ASOF
         │
         ▼
    50 QBO Import Hub Standalone — sales-tax recognition derivation
         │
         └── derives recognition evidence from the captured GL dataset;
             it is not assumed to be real-time
         │
         ▼
    21 Bookkeeping Audit — Level 2 reconciliation / explanation
```

The liability evidence path is:

```text
QBO Sales Tax Liability
        │
        ▼
21 Bookkeeping Audit — QBO_Liability_Data
        │
        ├── 00_Controls
        ├── 01_Current
        ├── 02_Liability_Snapshots
        ├── 03_Snapshot_Changes
        └── 90_Ingestion_Log
        │
        ▼
LEVEL 1
Filed / reconstructed filing basis
        vs
Frozen QBO liability snapshot
        │
        ├── no variance -> PASS
        └── variance -> Level 2
```

Level 2 combines TaxableSalesDetail snapshots/changes, GL-derived recognition evidence, liability snapshots/changes, preserved filing-version evidence, and where needed source-transaction and payment/application state. Its acceptance standard is that transaction-level explained dollars equal the Level 1 variance with no unexplained dollar difference remaining.

## Application Ownership

- **50 QBO Import Hub Standalone** owns direct QBO acquisition and governed source datasets. TaxableSalesDetail is a direct current QBO report pull with immutable Snapshot Records and Captured-State CDC evidence. Sales-tax recognition is derived from the existing captured GL export and therefore preserves the GL source ASOF rather than being labeled real-time.
- **21 Bookkeeping Audit** owns Level 1 reconciliation conclusions and Level 2 transaction-level variance explanation. It is the primary consumer of permanent TaxableSalesDetail and GL-recognition evidence.
- **05 Integration Hub** may provide selected reusable normalization, parsing, matching, and transformation logic where needed, including support for the existing normalized sales-tax dataset contract. It does not own the production Level 1/Level 2 reconciliation pipeline. The old Phase 9 three-source bridge and audit-output orchestration must not be treated as production architecture.
- **40 Data Platform Shared Library** remains the reusable controls/framework layer and is not the owner of sales-tax-specific accounting rules.

## Data Storage Architecture

Storage location is determined by the **role of the asset**, not by the application that creates it.

- `Data Platform/Applications/<App>/...` contains application-owned artifacts such as configuration, operating workbooks, reports, test repositories, and application-specific outputs.
- `Data Platform/Data Exchange/<Source or Domain>/...` contains governed datasets intended to be consumed across applications. The producer application does not determine the dataset's semantic home.

For shared QuickBooks datasets, the governed current-workbook location is:

```text
Data Platform/
└── Data Exchange/
    └── QuickBooks/
        ├── Current/
        ├── Master Backups/
        └── Run Snapshots/
```

`Current` identifies the location of the current persistent governed workbook. A workbook stored there may itself contain immutable internal history tables such as `02_<Dataset>_Snapshots` and `03_Snapshot_Changes`. Those immutable row/state records are the audit-history authority and are distinct from Drive-level preservation structures. Master Backups are full-file recovery copies. Run Snapshots contain immutable evidence associated with a run or ASOF and do not inherently require duplicating an unchanged full dataset or workbook.

Examples of governed shared QBO datasets include `QBO_Taxable_Sales_Detail_Data`, `QBO_Liability_Data`, and the governed General Ledger dataset. New shared datasets must not be placed under `Applications/<Producer App>/Workbook` merely because that application provisions them.

Master Backups are full-file recovery copies. Run Snapshots contain immutable evidence associated with a run or ASOF and do not inherently require duplicating an unchanged full dataset or workbook.

## Snapshot and Change Terminology

Use these terms consistently across Apps Script applications:

- **Current** — replaceable latest operational state.
- **Master Backup** — full dataset/workbook duplication for recovery.
- **Run Snapshot / Snapshot Record** — immutable evidence of a state observed at a specific run or ASOF. A Snapshot Record does not inherently require duplicating unchanged rows or the entire dataset.
- **QBO Native CDC** — the source-provided QBO change signal. QBO reports that an entity changed within the requested CDC window.
- **QBO Native CDC Event** — preserved evidence of the source-side change signal returned by QBO.
- **Captured-State CDC** — Pool People's independent comparison of states actually captured. It determines whether an observed state differs from the prior captured state.
- **Change Record** — immutable evidence of a change determined by Captured-State CDC, linking the prior captured state to the newly observed state.
- **Change Detail** — the exact field/path-level before-to-after differences calculated from the two captured states.
- **Run Manifest** — acquisition/run metadata including source and ASOF information, Native CDC window and counts where applicable, and snapshot/change counts.

“Snapshot” must not be used as a generic synonym for a file backup. QBO Native CDC and Captured-State CDC are complementary controls and neither replaces the other.

If QBO changes through intermediate states that Pool People does not capture, Captured-State CDC proves only the difference between the states actually observed. Unobserved intermediate states must not be inferred. Likewise, a QBO Native CDC event may prove that QBO reported a change even when the before and after states captured by Pool People are identical.