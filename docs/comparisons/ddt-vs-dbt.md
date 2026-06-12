# DDT vs dbt — complement, not competitor

dbt and DDT manage **different layers** of the lakehouse. Most teams that adopt DDT keep dbt exactly as it is.

## TL;DR

- Use **dbt** for: transformations — models, tests, materializations, docs for the modeled layer.
- Use **DDT** for: everything dbt deliberately doesn't model — raw/landing tables, volumes, external locations, functions, grants-bearing DDL, streaming tables created outside pipelines — with compare, drift detection, and catalog-aware safety classification.
- They meet at the boundary: DDT imports your dbt `manifest.json` so the modeled layer and the DDL layer live in one compared, classified view.

## The layer split

Ask a dbt-on-Databricks shop "how do you ship a new landing table, the volume it loads from, and the grants on both?" and the answer is hand-run DDL, a `run-operation` macro, or a notebook someone executes — the ad-hoc workflow dbt eliminated for models, alive and well one layer down.

| Layer | Owner |
|---|---|
| Raw / landing tables, volumes, external locations | **DDT** (dbt *declares* sources; it doesn't create or alter them) |
| Models (views/tables from `SELECT`) | **dbt** |
| Functions, procedures | **DDT** |
| Streaming tables outside Lakeflow pipelines | **DDT** |
| Tests on data values | **dbt** |
| Safety classification of DDL (managed-drop = file deletion, streaming-replace = checkpoint loss) | **DDT** |
| Schema drift vs git | **DDT** (`ddt drift`) — dbt notices drift only when a run breaks |

dbt's Fusion engine (real SQL comprehension, column-level lineage) is a genuine leap for the modeled layer — and doesn't change this boundary: Fusion models *models*. Non-model DDL remains outside dbt's design scope.

## Honest limits of the complement story

- If your lakehouse is **100% dbt-modeled**, DDT adds little — you don't have the layer it manages.
- DDT does **no transformation work** — no Jinja, no incremental materialization, no data tests. It will never replace dbt and doesn't try.
- Two tools is two tools; the importer reduces the seam but doesn't remove it.

## Using them together

```sh
# Seed a DDT project from your dbt manifest — models and sources arrive as declared objects:
ddt import --from dbt --source-path ./target/manifest.json --output ./Lakehouse

# dbt keeps owning the models; DDT compares + classifies the full DDL surface:
ddt compare --project ./Lakehouse --target prod
```

CI pattern: `dbt build` gates the transformation layer, `ddt drift --target prod` gates the DDL layer — both fail the pipeline, each for its own layer. See [CI/CD integration](../ci-cd.md).

---

*Independent comparison written by the DDT maintainer — corrections welcome via [issues](https://github.com/GVOrganization/ddt-tools/issues). "dbt" is a trademark of dbt Labs, Inc.*
