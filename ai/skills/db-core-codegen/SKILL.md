---
name: db-core-codegen
description: "End-to-end workflow for TypeScript db-core modules: discover schema/model inputs, run DB/document/DAO generation, scaffold DB/document converters, and scaffold procedures with capability-aware validation and safe persistence boundaries. Use when the user asks to regenerate db-core artifacts, generate beans/DAOs from schemas, scaffold or update converters, or scaffold procedure CRUD methods. Trigger keywords: db-core, Db beans, DAO, converter, procedure."
---

# db-core Code Generation

Use this skill when you need to generate or update any db-core layer in a module:
1. DB beans
2. Document beans
3. DAOs
4. DB/document converters
5. Procedures that call DAOs

This skill is designed to keep generation and scaffolding deterministic, schema-driven, and safe across module variants.

## What this skill guarantees

1. Schema-first decisions: generation inputs and converter context fields are inferred from discovered schema/model files, not script aliases alone.
2. Capability-aware planning: generation resolves from module capabilities (or `generate` script), with DAO `--schemas` validation in both `--plan auto` and `--plan chain`.
3. Predictable converter scaffolds: defaults to `generated/beans/*` imports, enforces `convertFrom`/`convertTo` mapping rules, and preserves stable context parameter order.
4. Safe optional wrappers: validates and uses `src/beans/*` and `src/dao/*` only when those wrappers actually exist.
5. Procedure-safe boundaries: procedure APIs stay DB-oriented while DAO APIs stay document-oriented, with convention-aware scaffolding defaults.
6. Default audit ownership: `created_at`/`updated_at` ownership stays in DAO unless explicitly overridden.
7. Preflight pause/resume: pause only when `node_modules` is missing; ask the user to run `npm run install-snapshot` (override script name via `--install-script`), then resume after user confirmation.
8. Self-contained execution: command usage, contracts, and post-change checks are defined in this skill without requiring external definition docs.
9. Module-local context only: inference, examples, conventions, and edits are restricted to the selected `--module` root.

## NEVER do these (anti-patterns)

1. NEVER manually edit generated artifacts under `generated/beans/*` or `generated/dao/*`.
   1. Why: generators overwrite these files, so manual edits drift and are lost on regeneration.
   2. Correct action: change schema/model inputs and rerun generation.
2. NEVER patch generated TypeScript to fix field/type/model mismatches.
   1. Why: generated output is derivative; schema/datamodel files are the source of truth.
   2. Correct action: update the schema/datamodel inputs first, then regenerate.
3. NEVER write or modify code outside the target `db-core` scope during this workflow.
   1. Why: cross-scope edits create hidden coupling and unnecessary blast radius.
   2. Exception: update code outside `db-core` only when the user explicitly asks for it.
4. NEVER use sibling/other `db-core` modules for context inference, convention inference, or copy patterns.
   1. Why: this introduces cross-module coupling and wrong conventions.
   2. Correct action: infer only from files under the selected module root.

## Inputs

Collect these before making changes:

1. `module selector`: exact module path or module hint.
2. `scope`: `generation-only`, `converter-only`, `generation-and-converter`, `procedure-methods`, or `full-procedure`.
3. `target entities`: DB bean + document + DAO + converter names.
4. `persistence contract`: whether module uses direct `generated/dao` or optional `src/dao` wrappers.
5. `schema locations`: discover from generation scripts when not explicitly provided.

## Module boundary protocol (mandatory)

1. Treat the selected `--module` directory as the only allowed context root for module files.
2. Read/inspect only paths that are descendants of that module root when inferring schemas, conventions, converters, procedures, and DAO usage.
3. Reject any inferred/discovered path that resolves outside module root; do not use it for decisions.
4. Do not sample procedures/converters from other modules for style inference.
5. If required files are missing inside module root, report and stop or continue with explicit warning; do not substitute from another module.

## Reference loading protocol (mandatory)

Before any implementation, select `scope` and load only the required references.

1. `generation-only`:
   1. MANDATORY - read entire `references/generation-checklist.md`.
   2. Do NOT load `references/converter-cookbook.md` or `references/procedure-cookbook.md`.
2. `converter-only`:
   1. MANDATORY - read entire `references/converter-cookbook.md`.
   2. OPTIONAL - read `references/generation-checklist.md` only if regeneration is also requested.
   3. Do NOT load `references/procedure-cookbook.md`.
3. `generation-and-converter`:
   1. MANDATORY - read entire `references/generation-checklist.md` and `references/converter-cookbook.md`.
   2. Do NOT load `references/procedure-cookbook.md` unless procedure scaffolding is explicitly requested.
4. `procedure-methods` or `full-procedure`:
   1. MANDATORY - read entire `references/procedure-cookbook.md`.
   2. OPTIONAL - read `references/generation-checklist.md` only when generation is in scope.
   3. Do NOT load `references/converter-cookbook.md` unless converter changes are explicitly requested.
5. Always report which references were loaded and why before running commands.

## Discovery-first workflow (required)

Before creating/updating converters or procedures, do all of the following:

1. Apply `Reference loading protocol (mandatory)` and declare loaded files.
2. Parse module generation scripts and resolve schema/model paths:
   1. document generator (for example `json-to-schemas --datamodel ... --out ...`)
   2. DB generator (for example `jsonschema-to-typescript-objects --input ... --beans-path ...`)
   3. DAO generator (for example `pdm-to-daos --datamodel ... --schemas ... --out ...`)
3. Resolve all script-relative paths against module root, verify they exist, and verify each resolved path stays within module root.
4. Read discovered schemas/models to capture per-entity:
   1. required document keys (PK/SK/index keys)
   2. `@entity` discriminator/default
   3. field requiredness and types
5. Infer converter context fields from schema requirements:
   1. include required document keys not sourced from DB bean fields
   2. prioritize partition/sort keys, then required index keys
   3. keep parameter names consistent with existing module converter/procedure patterns
6. If multiple context-field candidates are plausible:
   1. rank candidates with brief reasoning
   2. choose the best candidate and proceed
   3. surface the decision in output
7. Missing discovered model/schema files should fail inference (or be surfaced as warnings when non-blocking).

## Script commands

Run from repository root.

### Generation

```bash
python .agents/skills/db-core-codegen/scripts/run_dbcore_codegen.py \
  --module <module_root> \
  --plan auto
```

`--plan auto`:
1. Uses `generate` script if available.
2. Otherwise resolves chain by capability (DB, document, DAO, optional tx DAO).

`--plan chain`:
1. Always resolves chain by capability.
2. Requires DB + document + DAO capabilities.
3. Includes tx DAO only when available or explicitly requested.

Install preflight:
1. Before running generation scripts, checks `generated/beans/db`, `generated/beans/document`, and `generated/dao`.
2. Also checks whether `node_modules` exists.
3. If `node_modules` is missing, do not run install commands automatically; pause and ask the user to run `npm run install-snapshot`.
4. Missing/empty generated outputs are surfaced in preflight metadata, but generation still proceeds when `node_modules` exists.
5. Override install script name with `--install-script <script-name>`.

Pause/resume protocol:
1. On preflight failure, stop generation and report the exact command and module root to the user.
2. Do not execute install/setup commands on behalf of the user.
3. After user confirms completion, rerun the same generation command to resume.

### Converter scaffolding

Flat converter layout:

```bash
python .agents/skills/db-core-codegen/scripts/generate_db_converter.py \
  --module <module_root> \
  --converter-dir . \
  --db-type Account \
  --document-type Account \
  --entity-id com.example.document.account \
  --with-list
```

Nested converter layout:

```bash
python .agents/skills/db-core-codegen/scripts/generate_db_converter.py \
  --module <module_root> \
  --converter-dir events/event-subscription \
  --db-type EventSubscription \
  --document-type EventSubscription \
  --entity-id com.example.document.event_subscription \
  --context-field user_id:userId
```

If `--context-field` is omitted, context fields are inferred from discovered schema/model metadata.

### Procedure scaffolding

CRUD method scaffolding with convention-aware defaults:

```bash
python .agents/skills/db-core-codegen/scripts/generate_db_procedure.py \
  --module <module_root> \
  --db-type Link \
  --document-type Link \
  --methods get,post,update
```

Common overrides:
1. `--timestamp-strategy procedure|dao` to override timestamp ownership (default: `dao`).
2. `--post-generate-id always|never` to force UUID generation behavior.
3. `--beans-import-style index|entity` and `--dao-import-style index|entity`.
4. `--dao-class <EntityDaoClass>` and `--dao-method-*` when DAO naming differs.

## Contracts

### Generation contract

1. Validate selected DAO/tx DAO scripts have correct `--schemas` usage.
2. Run install preflight checks before generation; pause for manual user action only when `node_modules` is missing.
3. Validate generated outputs:
   1. `generated/beans/db`
   2. `generated/beans/document`
   3. `generated/dao`
4. Validate wrappers only if wrapper files exist.
5. Validate package export/type contracts when present.

### Converter contract

1. `convertFrom(document)` maps `Document -> DB`.
2. `convertTo(db, ...context)` maps `DB -> Document`.
3. `convertTo` must set `@entity`.
4. Context keys must come from schema-first inference and be explicit typed parameters.
5. Context parameter order should be stable:
   1. partition key context
   2. sort key context not available on DB bean
   3. required index context
6. Failures must throw `ConverterException`.

### Procedure contract

1. Procedure boundary accepts/returns DB objects.
2. DAO boundary accepts/returns document objects.
3. Procedures should use DAOs via module capability:
   1. `generated/dao` by default.
   2. `src/dao` wrappers only when that capability exists in the module.
4. DAO selection must be explicit from method intent:
   1. Use `TableDao` for query/list/index pagination and non-atomic batch access.
   2. Use `TransactionDao` only when atomic all-or-nothing behavior or cross-item invariant enforcement is required.
   3. If atomicity is not explicitly required, default to `TableDao` (or `EntityDao` for single-item key CRUD).
5. Capture and report DAO choice inputs when scaffolding procedure methods:
   1. operation type (`query/list`, `single-item CRUD`, `multi-item mutation`)
   2. atomicity required (`true/false`)
   3. invariant scope (single item vs cross-item/cross-entity)
6. Audit timestamps (`created_at`/`updated_at` and camel variants) are DAO-managed by default.
   1. Use `--timestamp-strategy procedure` only when a module intentionally follows that non-default pattern.
   2. Surface the chosen strategy in output.

## Output checklist

After changes:

1. Generation plan is explicit and valid for detected module capabilities.
2. Converter files compile and exports are updated idempotently.
3. No hard failures due to absent optional wrapper files.
4. Discovered schema/model paths are reported.
5. Inferred context fields are reported per entity with brief reasoning.
6. Dry-run output is available (human + `--json` modes).
7. `--json` output includes:
   1. `discovered_schema_paths`
   2. `selected_script_paths`
   3. `validated_schema_targets`
   4. `inferred_context_fields`
   5. `context_decisions`
   6. `entity_metadata`
   7. `install_preflight` (`script`, `script_exists`, `command`, `executed`, `manual_action_required`, `missing_outputs`, `missing_node_modules`)
8. Procedure scaffold output (when used) reports:
   1. resolved conventions
   2. resolved import styles
   3. selected mutation strategy
9. Report `loaded_references` and `reference_selection_reason`.
10. Confirm no manual edits to generated files and no out-of-scope (`db-core`) changes unless explicitly requested by the user.
11. Report `module_root` and confirm all module-file context paths used for inference were under that root.

## References

Load these only through the `Reference loading protocol (mandatory)` above.

1. `references/generation-checklist.md`
2. `references/converter-cookbook.md`
3. `references/procedure-cookbook.md`
