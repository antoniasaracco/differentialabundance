# Issue #551 — unification of differential expression column parameters

This note explains the intent of issue #551 and where pipeline code must be updated.

## Problem being addressed

The pipeline currently exposes and redefines differential-result column semantics in multiple places:

- Global defaults in `nextflow.config` (`differential_fc_column`, `differential_pval_column`, `differential_qval_column`, `differential_foldchanges_logged`).
- Method-specific values in `conf/paramsheet.yaml`.
- Hardcoded method-specific filtering logic in `subworkflows/nf-core/abundance_differential_filter/main.nf`.
- Downstream module arguments in `conf/modules.config` that assume these parameters exist in `meta.params`.

This creates duplication and user confusion, because users can see/override parameters that should be internal method metadata.

## Agreed direction from ticket discussion

Use method-owned metadata instead of user-facing parameters:

1. Determine DE columns/cardinality from the selected method in the differential subworkflow.
2. Add method-derived values into `meta.params` (or another internal meta map) on differential output channels.
3. Consume that internal metadata downstream for filtering/reporting/shiny outputs.
4. Remove user-facing schema/CLI exposure of these method-dependent parameters.

The same approach also applies to `differential_foldchanges_logged` (see issue #648 reference in #551).

## Concrete repo updates to implement

1. **Subworkflow (`abundance_differential_filter`)**
   - Build a method->column mapping once.
   - Attach mapping values to output `meta` (e.g. `differential_fc_column`, `differential_pval_column`, `differential_qval_column`, `differential_foldchanges_logged`, and filter cardinalities).
   - Reuse these fields in filtering instead of local hardcoded map.

2. **Pipeline/module wiring (`conf/modules.config`)**
   - Keep downstream module args reading `meta.params.*`, but ensure values now originate from subworkflow metadata injection.
   - Remove any remaining dependency on user-provided values for DE column names.

3. **Defaults and schema**
   - Remove deprecated user params from `nextflow.config` defaults.
   - Remove/deprecate entries from `nextflow_schema.json`:
     - `differential_fc_column`
     - `differential_pval_column`
     - `differential_qval_column`
     - `differential_foldchanges_logged`
   - Keep threshold params (`differential_max_pval`, `differential_max_qval`) if still intended to be user-configurable.

4. **Paramsheet examples/tests**
   - Remove DE column-name keys from `conf/paramsheet.yaml` and any tests that set them.
   - Update module/subworkflow tests expecting old params.

5. **Docs/changelog**
   - Document that DE column names and fold-change logging state are now method-internal.
   - Mention migration impact for users with old parameter sets.

## Suggested implementation order

1. Refactor subworkflow metadata propagation.
2. Update downstream consumers to read the propagated metadata.
3. Remove schema/config exposure.
4. Update tests and docs.
