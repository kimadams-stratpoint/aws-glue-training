Executive Summary

This document outlines the architectural decisions for an automated, Medallion Data Architecture (Landing → Bronze → Silver → Gold) built on AWS Glue, Amazon S3, Glue Data Catalog, and Amazon Athena. The system is designed to be incrementally self-updating, supporting bookmark-driven ingestion, partition-scoped writes, dynamic catalog metastore updates, database-level security segregation, and strict fault tolerance via a dead-letter quarantine mechanism.

Key Architectural Decisions & Justifications

Multi-Hop Medallion Storage Model:

Landing: Acts as an un-mutated, append-only staging area for incoming raw CSV payloads, partitioned by year/month in Hive-style S3 prefixes.
Bronze: Incrementally ingests only newly landed CSVs, re-encoding them into compressed, partitioned Parquet files while stamping native batch/run IDs and ingest timestamps for lineage audits.
Silver: Conformed, cleaned, row-level data with explicit data-quality validation, optimized for analytical processing.
Gold: Pre-aggregated metrics and dimensional models tailored specifically for business consumption and BI reporting.
Quarantine: Dead-letter target that isolates records failing structural or business-rule validation from operational data models.

Incremental, Bookmark-Driven Bronze Ingestion
Decision: Ingest Landing data using AWS Glue's create_dynamic_frame.from_options reader with Job Bookmarks enabled (transformation_ctx-tracked), rather than a full re-read of the Landing zone on every run.

Justification:

Cost & Runtime Efficiency: Each run processes only files not previously seen by Glue's bookmark state, avoiding a full reprocess of historical data as the Landing zone grows.
Idempotent Partition Writes: Combined with dynamic partition overwrite mode, each run writes only the year/month partitions actually touched by new files, leaving all other Bronze partitions untouched.
Path-Derived Partitioning: Because bookmark-mode ingestion lists source files individually rather than performing directory-level Hive partition discovery, year/month are derived explicitly per record from each file's S3 path (via input_file_name()), preserving the same partitioning contract Silver and Gold depend on.
Downstream Schema Stability: A _corrupt_record column is preserved (populated as NULL when not natively supported by the reader) so Silver's existing quarantine logic continues to evaluate row structure without a breaking schema change.

Strategic Storage Segregation & Governance

Decision: Store Landing, Bronze, Silver, Quarantine, and Gold data as dedicated folders/prefixes within a single S3 bucket (s3-kim-capstone-training), rather than splitting business-facing and operational data across separate buckets.

Justification:

Role-Based Access Control (RBAC): Folder-level (prefix-based) IAM/Lake Formation policies allow business users and BI dashboards to be scoped strictly to the gold/ prefix, without granting access to raw, intermediate, or quarantined data — access boundaries are enforced by prefix policy rather than physical bucket separation.
Operational Simplicity: Consolidating all layers under one bucket simplifies lifecycle management (versioning, logging, replication, cost allocation tags) since a single bucket-level configuration applies end-to-end, rather than needing to keep multiple bucket policies in sync.
Quarantine Proximity: Placing quarantine/ alongside silver/ keeps failed records close to the validation logic that produced them, simplifying reprocessing and root-cause review.
Clear Consumption Boundary: Although physically co-located, the gold/ prefix is treated as the only sanctioned access point for downstream BI tools and non-technical consumers — Athena/Glue Catalog table grants are scoped accordingly even though storage isn't physically split.

Fault Tolerance via Silver-Layer Quarantine
Decision: Apply row-level validation in Silver (event time parseability, allowed event types, required product/session identifiers, non-negative pricing) and route failing records to a dedicated Quarantine path rather than dropping or blocking the pipeline.

Justification:

Non-Blocking Ingestion: Malformed or incomplete records don't halt the pipeline; they're isolated with an explicit quarantine_reason for later inspection or reprocessing.
Auditability: Every quarantined record retains its reason(s) for exclusion, supporting root-cause analysis on upstream data quality.

Storage Optimization & Partitioning
Decision: Enforce Parquet format partitioned by year and month, with mergeSchema enabled to tolerate source column evolution over time.

Justification:

Columnar Efficiency: Lowers Amazon Athena costs (billed per byte scanned) via column projection pushdowns.
Partition Pruning: Prevents expensive full-table scans on time-bounded queries.
Schema Evolution Tolerance: New source columns introduced in later batches are absorbed without breaking existing Parquet reads.
