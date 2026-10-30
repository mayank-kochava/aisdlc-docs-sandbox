# TG1: Run tests & validate build

## In short

Run the full build and tests for Kochava/aim-data-ingestion with every task above merged.

🟢 Clear

## Overview

| Field | Value |
|---|---|
| Task ID | TG1 |
| Repo | Kochava/aim-data-ingestion |
| Phase | Phase 9 |
| User Story | N/A |
| Parallel | No |
| Status | New |
| Owner | Test/Change Agent |

## Context

Run the full build and tests for Kochava/aim-data-ingestion with every task above merged. Do not edit source.

This task is part of Kochava Connector — New ETL (Sharawmer Migration).

### Source Code — `aim-data-ingestion`
```
Aim.DataIngestion.Core/
├── DataImport/
│   ├── DataSuppliers/
│   │   └── IKochavaRedshiftReport.cs                    [DONE]
│   ├── Models/Configuration/KochavaRedshift/
│   │   ├── KochavaRedshiftConfigModel.cs                [DONE]
│   │   └── KochavaRedshiftFeatureStoreTableDefinitions.cs [DONE]
│   ├── Models/Report/KochavaRedshift/
│   │   └── KochavaRedshiftReportModel.cs                [DONE]
│   └── Services/KochavaRedshift/
│       └── KochavaRedshiftDataImportService.cs          [DONE]
└── DataConsolidation/Services/KochavaRedshift/
    ├── KochavaRedshiftDataAggregator.cs              [TODO]
    └── KochavaRedshiftDataConsolidator.cs            [TODO]
Aim.DataIngestion.DataSuppliers/
└── KochavaRedshift/
    └── KochavaRedshiftReport.cs                      [IN PROGRESS]
Aim.DataIngestion.Command/
├── IngestionTaskFactory.cs                          [DONE — dataimport case; TODO — consolidation wiring]
└── ServiceContainer.cs                              [DONE]
Aim.DataIngestion.Shared/
└── IngestionTask/Tasks.cs                            [DONE]
Aim.DataIngestion.Tests/
└── DataSuppliers/KochavaRedshift/
    ├── KochavaRedshiftReportTests.cs                  [TODO]
    └── KochavaRedshiftReportBuildQueryTests.cs        [TODO]
Aim.DataIngestion.Tests/
└── Core/DataImport/Services/KochavaRedshift/
    └── KochavaRedshiftDataImportServiceTests.cs       [TODO]
```

### `aim-data-ingestion`
**Technology:** C# 12 / .NET 8, Autofac IoC, Npgsql, xUnit  
| Change | File | Type | Complexity | Notes |
| :--- | :--- | :---: | :---: | :--- |
| Kochava RS interface | `IKochavaRedshiftReport.cs` | Done | — | Complete |
| Config model | `KochavaRedshiftConfigModel.cs` | Done | — | Complete |
| Table definitions | `KochavaRedshiftFeatureStoreTableDefinitions.cs` | Done | — | Complete |
...

## Implementation Guide

1. Implement: Run tests & validate build
2. Add or update the tests listed under Files to Modify
3. Run `dotnet test`

## Files to Modify

| File | Create/Modify/Test | Note |
|---|---|---|
| (none captured, check the plan) | - | - |

## Acceptance Criteria

- Build and existing tests still pass.
- Existing tests pass
- New functionality has test coverage
- Code follows repository conventions (see CLAUDE.md)

## Dependencies

**Blocked by:**
- T010 - Create Aim.DataIngestion.Tests/DataSuppliers/KochavaRedshift/KochavaRedshiftReportBuildQueryTests.cs with two tests…
- T011 - Add SwapKochavaAppIds and Dispose tests to Aim.DataIngestion.Tests/DataSuppliers/KochavaRedshift/KochavaRedshiftReportT…
- T012 - Create Aim.DataIngestion.Tests/Core/DataImport/Services/KochavaRedshift/KochavaRedshiftDataImportServiceTests.cs with…
- T018 - Deploy check: run the featurecreation and featurecreation.postprocessor jobs for Sharawmer on dev

**Blocks:**
- none

## Testing Notes

```bash
dotnet test
```

## Reviewer Notes

Add suggestions here or as Asana comments; tell the agent in the Slack thread and it will apply them.

Coding agent: one PR per repo — include all `Kochava/aim-data-ingestion` tasks in one PR on branch `impl/prd059-backend`.

---

**Parent Task:** Kochava Connector — New ETL (Sharawmer Migration) — Kochava/aim-data-ingestion (20 tasks · 1 PR)
**PRD:** docs/16-PRDs/059-kochava-connector-new-etl