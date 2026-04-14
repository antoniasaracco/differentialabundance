# Issue #551: Unify DE column parameters (`lfc`, `pval`, `qval`)

This note explains what issue [#551](https://github.com/nf-core/differentialabundance/issues/551) is asking for and how to implement it in this repository.

## What the issue is about

Today, the pipeline carries differential result column metadata in multiple places:

- Global defaults (`nextflow.config`)
- Profile overrides (`conf/profile/*`)
- Runtime schema parameters (`nextflow_schema.json`)
- Method-specific hardcoding in `abundance_differential_filter`

The issue proposes to **stop exposing DE result-column names as user parameters** and to define them from the chosen differential method (DESeq2 / limma / dream / propr) in one coordinated place.

## Current duplicated parameters

The issue and linked discussion target these parameters:

- `differential_fc_column`
- `differential_pval_column`
- `differential_qval_column`
- `differential_foldchanges_logged` (coordinated with issue #648)

## Recommended implementation direction

Based on issue discussion and follow-up comments, the preferred direction is:

1. Put DE-column metadata in `meta.params` as early as possible (at paramset creation time).
2. In `ABUNDANCE_DIFFERENTIAL_FILTER`, use method-aware defaults and only inject values when missing.
3. Ensure all downstream consumers (`functional`, `shinyngs`, reports) read from `meta.params` and no longer require user-facing overrides for these fields.

This gives one metadata contract for downstream modules and reduces fragile per-profile/per-module duplication.

## Files you need to touch

### 1) Paramset creation / normalization

**File:** `subworkflows/local/utils_nfcore_differentialabundance_pipeline/main.nf`

- Add/update logic during paramset generation so each paramset has method-aware defaults for DE columns and fold-change logging in `meta.params`.
- Keep behavior compatible with paramsheet overrides where needed (or explicitly deprecate).

### 2) Differential subworkflow metadata handoff

**File:** `subworkflows/nf-core/abundance_differential_filter/main.nf`

- Reuse centralized DE column metadata instead of hardcoding multiple mappings.
- Keep method-specific filtering/cardinality logic explicit, but source column names from `meta.params` (with safe fallback).
- Ensure outputs preserve this metadata for downstream usage.

### 3) Remove user-facing config/profile duplication

**Files:**

- `nextflow.config`
- `conf/profile/**/*.config` (all profiles setting `differential_*_column`)

- Remove or deprecate direct user/profile exposure of DE column name parameters.
- Keep user-facing filtering thresholds (`differential_min_fold_change`, `differential_max_qval`) unless separately changed.

### 4) Schema cleanup

**File:** `nextflow_schema.json`

- Remove or deprecate:
  - `differential_fc_column`
  - `differential_pval_column`
  - `differential_qval_column`
  - `differential_foldchanges_logged` (if resolved with #648)
- Update descriptions/help text so users are not asked to provide method-internal column names.

### 5) Downstream consumers using DE columns

**File:** `conf/modules.config`

- Audit modules that currently read `meta.params.differential_*_column` and `meta.params.differential_foldchanges_logged`.
- Confirm they consume the new unified metadata keys consistently.

### 6) Test fixtures

**Files:**

- `conf/test_paramsheet.yaml`
- Relevant snapshot tests under `tests/`

- Update tests that still define explicit per-paramset column names.
- Regenerate snapshots if output metadata structure changes.

## Practical migration notes

- Keep one method mapping table for:
  - DE column names (`fc`, `pval`, `qval`)
  - Whether fold changes are already logged
- Treat this as internal metadata, not a user-facing API.
- Prefer adding metadata once and passing it through channels rather than remapping repeatedly.

## Expected outcome

After implementing #551 (and aligned #648 changes), users should only choose the DE method and thresholds, while the pipeline internally knows which DE table columns to use for each method.
