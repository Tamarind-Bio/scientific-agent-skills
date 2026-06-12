# Tamarind Bio tool catalog

Tamarind exposes hundreds of tools through one uniform job API. The catalog changes frequently — **always enumerate at runtime** with `GET /tools` (or MCP `getAvailableTools`) rather than hardcoding names. This file is a map for interpreting what you get back.

## How to discover

**REST** `GET /tools` returns the **full list** (it does not filter server-side). Filter client-side:

```python
tools = requests.get(f"{BASE}/tools", headers=HEADERS).json()      # a list
docking = [t for t in tools if "diffdock" in t["name"].lower()]
```

Each REST tool entry carries: `name` (the `type` you submit), `displayName`, `description`, `github`, `paper`, and `settings` (the inline parameter schema). REST entries do **not** include `categories`/`tags`.

**MCP** `getAvailableTools(search=..., category=..., tag=..., custom=...)` filters server-side and returns entries with `categories` and `tags`. Set `custom=true` to list only your account's custom tools.

## Categories

| Category | What it covers |
|---|---|
| `protein` | General protein structure/design/analysis (largest category) |
| `antibody` | Antibody/nanobody design, humanization, developability, immunogenicity |
| `enzyme` | Enzyme design and engineering |
| `small-molecule` | Ligands, docking, small-molecule property prediction/generation |
| `peptide` | Peptide design and structure |
| `nucleic-acid` | DNA/RNA design, RNA language models |
| `cryoem` | Cryo-EM density-guided modeling |
| `finetuning` | Finetuning workflows for supported models |

## Common tags

`structure-prediction`, `protein-design`, `binder-design`, `antibody-design`, `protein-ligand-docking`, `protein-protein-docking`, `binding-affinity`, `inverse-folding`, `developability`, `humanization`, `immunogenicity`, `molecular-dynamics`, `protein-language-models`, `rna-language-models`, `small-molecule-property-prediction`, `generate-small-mols`, `mutation-scoring`, `solubility`, `aggregation`, `utilities`, `experimental-data`, `finetuning`.

## Representative tool families

Verify exact names and availability with `/tools` — these are common anchors, not an exhaustive or guaranteed list.

**Structure prediction / folding**
- `alphafold` — AlphaFold; monomer + multimer, MSA + templates, recycles, relaxation.
- `boltz` — Boltz-2; structure + affinity, biomolecular complexes incl. ligands.
- `chai` — Chai-1; complex structure prediction with optional MSA.
- `esmfold` / `esmfold2` — fast single-sequence folding.

**Protein / binder design**
- `rfdiffusion` — motif scaffolding.
- `boltzgen` — generative design.
- `bindcraft` — binder design.
- `proteinmpnn` / `ligandmpnn` — inverse folding (sequence given backbone; ligand-aware variant).

**Docking / affinity**
- `diffdock` — protein-small-molecule docking.
- Boltz/affinity tools — binding-affinity prediction.

**Antibody**
- Antibody language models and generators, humanization, developability, immunogenicity scoring.

**MSA / utilities**
- MSA generation tools feed downstream folding; utilities cover format conversion, scoring, and analysis.

## Reading a tool schema

`getJobSchema(jobType)` (MCP) or the `/tools` entry returns a `parameters` list. Each parameter has:

- `name`, `type` (`sequence`, `number`, `boolean`, `dropdown`, file types like `pdb`/`cif`/`sdf`, …)
- `descr`, `displayName`
- `required`, `default`
- `options` / `optionsDescr` (for dropdowns), `lowerBound` / `upperBound` / `lengthLimit`
- `conditionals` — applies only when another field has a given value
- `exclude` (`["api"]` / `["batch"]`) — omit on that surface
- `restrictOrgs` — gated to specific orgs/domains
- `list: true` — accepts multiple values/files
- `example` — a sample value

Top-level tool metadata also includes a `hint` such as: *"For file parameters, you can use s3Path values from listJobFiles() to chain jobs without downloading/uploading."*

Always read the schema before constructing `settings`, and run `validateJob` to confirm before `submitJob`.
