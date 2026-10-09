# IDMC CDI Projects

Self-initiated data integration projects built on **Informatica Intelligent Data Management Cloud (IDMC) – Cloud Data Integration (CDI)**.

> These are personal learning projects built with synthetic data. They are not client or production work.

## What these projects demonstrate

- **Transformations:** Source, Target, Expression, Filter, Joiner, Lookup, Router, Aggregator, Sorter, Sequence Generator
- **Patterns:** SCD Type 2, validation with reject routing, multi-stage pipelines where one mapping's output feeds the next, derived flags and KPIs
- **Orchestration:** taskflow with mapping tasks, Decision steps based on task status, and failure email notifications
- **Connectivity:** flat-file (CSV) sources and targets on AWS S3 through an IDMC connection

## Tech stack

IDMC CDI · Mapping Tasks · Taskflows · AWS S3 · CSV

## Repository layout

```
idmc-cdi-projects/
├── healthcare_project/    README + screenshots
├── banking_project/       README + screenshots
└── shipping_project/      README + screenshots
```

Each project folder has its own README covering the business problem, data flow, mapping logic and screenshots of the mapping canvases.