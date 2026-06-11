---
name: tamarind
description: Run protein and small-molecule modeling jobs on the Tamarind Bio platform via its REST API or MCP server. Use when the user mentions Tamarind, tamarind.bio, or wants to run structure prediction (AlphaFold, Boltz, Chai, ESMFold), protein/binder design (RFdiffusion, ProteinMPNN, BoltzGen), antibody design and developability, protein-ligand docking (DiffDock), binding-affinity prediction, MSA generation, or molecular dynamics in the cloud without local GPUs. Also trigger when code references app.tamarind.bio/api or the x-api-key header for Tamarind, or when a workflow needs to submit batches of sequences for structural or biophysical characterization.
license: MIT
compatibility: Requires Python 3.10+, a Tamarind Bio account, and an API key from app.tamarind.bio. Uses the `requests` library against the public REST API (no dedicated Python SDK exists). Network access required. Optional MCP server at mcp.tamarind.bio/mcp for agent hosts.
metadata:
  version: "1.0"
  skill-author: Tamarind Bio
  trigger-keywords: "protein structure prediction, AlphaFold, Boltz, Chai, ESMFold, protein design, binder design, antibody design, nanobody, protein-ligand docking, DiffDock, binding affinity, MSA generation, inverse folding, ProteinMPNN, RFdiffusion, BoltzGen, cloud GPU biology, structure prediction API, x-api-key"
---

# Tamarind Bio

Tamarind Bio is a cloud platform that runs computational biology tools — structure prediction, protein and antibody design, docking, binding-affinity, MSA generation, and molecular dynamics — on managed GPUs. Users submit sequences or structures and get back predicted structures, designs, and biophysical scores, without provisioning their own hardware. It exposes hundreds of tools (AlphaFold, Boltz-2, Chai-1, RFdiffusion, ProteinMPNN, BoltzGen, ESMFold2, DiffDock, and many more) through one uniform job API.

**Official docs:** [app.tamarind.bio/api-docs](https://app.tamarind.bio/api-docs) · platform UI at [app.tamarind.bio](https://app.tamarind.bio)

## Canonical sources — fetch these, don't rely on a stale copy

Tamarind publishes live, machine-readable sources. Prefer fetching them at runtime over trusting any hardcoded list — tool names, schemas, and endpoints change frequently:

- **`https://app.tamarind.bio/llms.txt`** — LLM index: links to the spec, API docs, and MCP guide.
- **`https://app.tamarind.bio/openapi.yaml`** — OpenAPI 3.0 spec for the 8 core job endpoints (submit-job/-batch, jobs, result, upload, files, delete-job/-file; auth `ApiKeyAuth`). Fetch it for those exact shapes. Discovery/management endpoints (`/tools`, `/usage-statistics`, pipelines, …) aren't in it — use the MCP/REST discovery tools for those.
- **`https://docs.tamarind.bio/llms.txt`** — documentation index; every page has a `.md` form (e.g. `docs.tamarind.bio/tamarind/batch.md`, `/tamarind/api.md`, `/tamarind/pipelines.md`).
- **Live tool discovery** — `GET /tools` (REST) or MCP `getAvailableTools` + `getJobSchema(jobType)` are the source of truth for what tools exist and their parameters.

This skill teaches the surface + the non-obvious behaviors those sources don't spell out (see the reference files). When in doubt about a shape, fetch `openapi.yaml`.

## When to use this skill

Use Tamarind when the user wants to:

- **Predict structure** of a protein, complex, or protein-ligand system (AlphaFold, Boltz-2, Chai-1, ESMFold2, Chai/Boltz cofolding)
- **Design proteins or binders** (RFdiffusion, BoltzGen, BindCraft, ProteinMPNN/LigandMPNN inverse folding)
- **Design or characterize antibodies/nanobodies** (sequence generation, humanization, developability, immunogenicity)
- **Dock small molecules** to a protein (DiffDock) or predict **binding affinity**
- **Generate MSAs** for downstream folding
- **Run molecular dynamics** or other biophysical workflows on managed GPUs
- **Batch-screen** many sequences or designs through the same tool
- **Chain tools** into pipelines (e.g. design → fold → score) using the output of one job as the input of the next

This skill is the right fit when the work should run on Tamarind's managed cloud rather than on a local install. For purely local cheminformatics or one-off sequence I/O, use a local library (RDKit, BioPython) instead.

## Access and authentication

1. Sign in at [app.tamarind.bio](https://app.tamarind.bio) and create an API key from the account/API settings.
2. Authenticate every REST request with the `x-api-key` header.
3. **Never hardcode the key.** Read it from the `TAMARIND_API_KEY` environment variable or a `.env` file (use `python-dotenv`). Never commit keys to source control.

```bash
export TAMARIND_API_KEY="your_api_key"
# List available tools
curl https://app.tamarind.bio/api/tools \
  -H "x-api-key: $TAMARIND_API_KEY"
```

**Base URL:** `https://app.tamarind.bio/api/`

There is **no official Python SDK** — the PyPI package named `tamarind` is an unrelated Neo4j tool. Do not `pip install tamarind`. Write plain `requests` calls against the REST API (the endpoint shapes are in `openapi.yaml`), or use the MCP server for agent hosts.

## Two ways to call Tamarind

### MCP server (best for AI agents)

Tamarind hosts an MCP server at `https://mcp.tamarind.bio/mcp` (API-key auth via the `X-API-Key` header). When your agent host supports MCP, prefer it — the tools mirror the REST API with agent-friendly schemas:

- `getAvailableTools(category?, tag?, search?, custom?)` — discover tools
- `getJobSchema(jobType)` — exact parameter schema for a tool
- `validateJob(jobName, type, settings)` — dry-run validation before submitting
- `submitJob(jobName, type, settings)` / `submitBatch(batchName, type, settings[], jobNames[])`
- `getJobs(jobName?, batch?, limit?)` — list/inspect jobs and statuses
- `getJobLogs(jobName)` — fetch output logs for debugging
- `listJobFiles(jobName)` — list output files (returns `s3Path` for chaining)
- `getResult(jobName, fileName?)` — download results
- `uploadFile(filename)` — get a presigned upload URL

Scope note: MCP query tools (`getJobs`, `getResult`, `listJobFiles`, …) are scoped to the authenticated account.

### REST API (universal)

Use plain HTTP with `requests` — the endpoint shapes are in `openapi.yaml`. The core loop is below; `references/workflows.md` has full recipes.

## Core workflow

Always follow discover → schema → validate → submit → poll → results. Do not hardcode tool names or settings — the catalog changes frequently.

```python
import os, time, requests

BASE = "https://app.tamarind.bio/api"
HEADERS = {"x-api-key": os.environ["TAMARIND_API_KEY"]}

# 1. Discover tools. REST /tools returns the full list; filter client-side.
tools = requests.get(f"{BASE}/tools", headers=HEADERS).json()
alphafold = next(t for t in tools if t["name"] == "alphafold")

# 2. Get the exact schema for the chosen tool.
#    REST: each /tools entry already includes its inline `settings` schema
#          (parameter list) — find the entry whose name == your job type.
#    MCP:  getJobSchema(jobType) returns the same per-tool detail.

# 3. Submit a job. `settings` is tool-specific — match the schema exactly.
payload = {
    "jobName": "my-alphafold-run",          # ^[a-zA-Z0-9_-]+$, <=100 chars, unique
    "type": "alphafold",
    "settings": {
        "sequence": "MKTVRQERLKSIVRILERSKEPVSGAQLAEELSVSRQVIVQDIAYLRSLGYNIVATPRGYVLAGG",
        "numRecycles": 3,
    },
}
resp = requests.post(f"{BASE}/submit-job", headers=HEADERS, json=payload)
resp.raise_for_status()   # 200 ok; 400 bad request; 403 budget exceeded; 401 unauthorized

# 4. Poll for completion.
#    NOTE the response shape: GET /jobs?jobName=<name> returns the job ROW
#    directly (no "jobs" wrapper); the list query (no jobName) returns
#    {"jobs": [...]}. Don't index ["jobs"][0] on the by-name response.
while True:
    job = requests.get(f"{BASE}/jobs", headers=HEADERS,
                       params={"jobName": "my-alphafold-run"}).json()
    if job["JobStatus"] in ("Complete", "Stopped", "Deleted"):
        break
    time.sleep(30)

# 5. Retrieve results. POST /result returns a presigned URL *string*;
#    GET that URL to download the actual results zip (two-step).
url = requests.post(f"{BASE}/result", headers=HEADERS,
                    json={"jobName": "my-alphafold-run"}).text.strip('"')
open("my-alphafold-run.zip", "wb").write(requests.get(url).content)
```

For the agentic version of this loop using MCP tools, and for richer examples, see `references/workflows.md`.

## Discovering tools

The catalog has hundreds of tools. Always enumerate at runtime — never rely on a hardcoded list.

**REST** `GET /tools` returns the **full list** (it does not filter server-side); each item is `{name, displayName, github, paper, description, settings}` where `settings` is that tool's inline parameter schema. Filter client-side:

```python
tools = requests.get(f"{BASE}/tools", headers=HEADERS).json()   # a list
boltz = [t for t in tools if "boltz" in t["name"].lower()]
```

Note: the **MCP** `getAvailableTools` can list the same tool name more than once (per-region/variant rows); the **REST** `/tools` list is deduplicated. Either way, match by `name` and take the first (`next(t for t in tools if t["name"] == "alphafold")`) rather than assuming a single row.

**MCP** `getAvailableTools(search=..., category=..., tag=...)` filters server-side and adds `categories`/`tags` per tool. Categories: `protein`, `antibody`, `enzyme`, `small-molecule`, `peptide`, `nucleic-acid`, `cryoem`, `finetuning`. Common tags: `structure-prediction`, `protein-design`, `binder-design`, `antibody-design`, `protein-ligand-docking`, `binding-affinity`, `inverse-folding`, `developability`, `molecular-dynamics`, `protein-language-models`.

A representative set of widely-used tools (verify with `/tools`): `alphafold`, `boltz` (Boltz-2), `chai` (Chai-1), `esmfold` / `esmfold2`, `rfdiffusion`, `proteinmpnn`, `ligandmpnn`, `boltzgen`, `bindcraft`, `diffdock`. See `references/tool_catalog.md` for the full category/tag map and how to read tool metadata.

## Choosing the right tool

The catalog has many tools per task; **don't hardcode a favorite — filter by `tag`, then read each candidate's `description` and match it to the user's actual goal** (input you have, output you need, constraints like speed or "no MSA"). The `description` and `tags` fields are the public "what it's for" signal; let them, plus `validateJob`, drive the pick. Quick orientation by task:

- **Fold a single protein / complex** (`tag=structure-prediction`): `alphafold` is the accurate default for monomers + multimers (join chains with `:`); `esmfold` is single-sequence (no MSA) and fast — reach for it when you want speed and have no MSA; `esmfold2` is newer and conditions on an MSA by default (with a faster single-sequence `esmfold2-fast` variant); `boltz`/`chai`/`openfold`/`protenix`/`intfold` are AlphaFold3-class for **protein + nucleic-acid + small-molecule complexes** (use these when a ligand/RNA/DNA is part of the system, not just protein — and `boltz` adds binding-affinity). Specialized folders exist for antibodies (`abodybuilder`, `immunebuilder`), cyclic peptides (`highfold`), and conformational ensembles (`afcluster`, `alphaflow`) — filter and read descriptions.
- **Design a binder** (`tag=binder-design`): `rfdiffusion` (backbone generation, de novo binders, motif scaffolding), `bindcraft` (de novo miniprotein binders), `boltzgen` (binders for protein **and** small-molecule targets, incl. nanobodies/antibodies/peptides). Antibody-specific generators live under `tag=antibody-design`.
- **Design sequence for a known backbone** (`tag=inverse-folding`): `proteinmpnn` (general), `ligandmpnn` (ligand-aware), plus thermostable/soluble/antibody MPNN variants. Inverse folding takes a **structure** and emits **sequences** — fold them back to verify (see chaining).
- **Dock a small molecule** (`tag=protein-ligand-docking`): `diffdock` (blind docking, no known pocket); `boltz`/`chai` co-fold the ligand into the complex when you'd rather predict the bound structure than dock into a fixed receptor.
- **Predict binding affinity** (`tag=binding-affinity`) or **generate an MSA** (search `msa`) — filter and read.

When the user names a specific tool, evaluate that one **and** sanity-check the alternatives in its `tag` group — a faster or more appropriate sibling often exists. When unsure, `getJobSchema`/`validateJob` to confirm a candidate actually accepts the input you have before committing.

## Job settings, schemas, and validation

Each tool has its own `settings` schema. Fetch it before submitting:

- **REST** `/tools` entry: each `settings` param is a **trimmed** dict. Only `name` and `required` are always present; `type`, `default`, `description`, `options` appear only when relevant (≈60% have `type`) — so use `param.get("type")`, not `param["type"]`. The advanced gating keys (`exclude`, `restrictOrgs`, `conditionals`) are **NOT in the REST response** at all.
- **MCP** `getJobSchema(jobType)`: the **full** schema, including `exclude`, `restrictOrgs`, `conditionals`, bounds. Use MCP when you need to reason about those gating keys.

**Always `validateJob` (MCP) before submitting** — it's the reliable guard. It runs the same validation as `/submit-job` without submitting, and surfaces the first missing/invalid field. Don't try to hand-derive which fields to strip from the schema keys (over REST you can't see them anyway) — let `validateJob` tell you. (The response may include a `source` field, e.g. `"static-fallback"` — an internal note on which schema source validated; `valid: true/false` is the signal you act on.)

⚠️ **Do NOT round-trip `validateJob`'s `normalized` output back into a submit.** The `normalized` blob echoes platform-internal fields (`submit_method`, `msa`, and UI-only fields like `chooseBest`); resubmitting it verbatim **400s** with `Unrecognized setting`. Build YOUR clean `settings`, validate them, then submit **the same clean settings** — not the normalized echo.

**Sequences:** amino-acid string; separate chains of a multimer with a colon (`:`), e.g. `"MVLS...:EVQL..."`. Note that some tools (e.g. `boltz`, `chai`) require more than `sequence` — `boltz` also requires `inputFormat` (and accepts `yamlFile`/`molecules`). Always `getJobSchema`/`validateJob` to learn a tool's required fields; don't assume `sequence` alone suffices.

**Platform-internal fields** — never set these yourself; the platform owns them: `submit_method`, `monomer_msa`, `msa`. See `references/api_reference.md` for the full field-handling rules.

## File inputs (PDB, CIF, SDF, …)

Tools with file parameters accept input three ways:

1. **Upload first, then reference.** `PUT /upload/{filename}` (or MCP `uploadFile` → presigned URL → `curl -X PUT -T file "<url>"`). After upload the file lives at `{email}/{filename}`.
2. **Reference a prior job's output** by its path: `JobName/path/to/file.ext` (this is how you chain jobs — see below).
3. **Inline content.** Send the file's text content directly as the field value.

**Foot-gun:** for a file-typed parameter, a **plain string value is treated as inline file content**, not as a path to an existing object. To point at an already-uploaded file, use the `{email}/{filename}` or `JobName/...` path form, not a bare string you expect to resolve.

## Chaining jobs into pipelines

A finished job's output becomes the next job's input — no download/re-upload. **Match the input type the next tool actually wants:** a sequence-design tool (ProteinMPNN) emits *sequences*, so you fold them by passing each as a `sequence`; a tool that takes a *file* parameter takes a path.

The cleanest design→fold chain is the MCP `submitBatch(fromJob=...)`, which reads a completed design job's generated sequences and folds each as one job:

```
# ProteinMPNN designs sequences -> fold every one with AlphaFold, one call:
submitBatch(batchName="verify-designs", type="alphafold", fromJob="my-proteinmpnn-job")
```

For a **file** input (e.g. a tool that takes a `.pdb`/`.cif`), reference a prior job's output by the path form `JobName/path/to/file.ext` in that file parameter. Two cautions, both confirmed by validation: (1) match the parameter's required **file type** — e.g. AlphaFold's `templateFiles` accepts only `.cif` and is a list, and is gated behind `templateMode: "custom"`; (2) `templateFiles` is for *structural templates*, not for "fold this designed sequence" — to fold a sequence, pass `sequence`. Always `getJobSchema`/`validateJob` to confirm a file param's type/conditions before chaining into it.

To discover a job's exact output paths, use MCP `listJobFiles(job1)` — it returns each file's `s3Path`, usable directly in the next `submitJob`. (The REST `GET /files` lists your account's *uploaded* files as a flat name list; it does not enumerate a job's outputs.) Tamarind also supports saved **pipelines**: build one in the UI, then drive it with `/run-pipeline` (`{pipelineName, initialInputs, inputs}`) or define `stages[]` inline via `/submit-pipeline` (each stage names a `task` + `toolSettings`, using `"pdbFile": "pipe"` to thread one stage's output into the next). See `references/workflows.md`.

## Finetuned and custom models

- **Finetuned models:** `GET /finetuned-models` lists your finetuned models — `{personalModels: [{name, type, inferenceType, baseModel, status}]}`. To run inference on one, submit its `inferenceType` as the job `type` and add `"modelName": "<name>"` to `settings`. Filter with `?type=plm-finetune` (etc.).
- **Custom models:** `POST /deploy-model` deploys your own container (`{modelName, dockerImage, inputSchema, outputSchema}`); `GET /models` lists deployed custom models. Custom tools also appear in `getAvailableTools(custom=true)`.

## Batch submission

Submit many jobs of the **same tool** in one call. The Python form uses parallel `settings[]` and `jobNames[]` arrays (same length, up to 100):

```python
requests.post(f"{BASE}/submit-batch", headers=HEADERS, json={
    "batchName": "egfr-binder-screen",
    "type": "alphafold",
    "jobNames": ["seq1", "seq2", "seq3"],
    "settings": [{"sequence": "..."}, {"sequence": "..."}, {"sequence": "..."}],
    # optional: "maxRuntimeSeconds": 3600, "weightedHoursBudget": 100,
    # (some accounts also accept an optional "gpuType" — confirm with support)
})
```

**Poll the batch *parent* on `batchStatus`, not subjob `JobStatus`.** A batch creates a parent job (`Type: "batch"`) plus subjobs. Subjobs flip to `Complete` as soon as they finish computing, but the batch then spends a few minutes **aggregating** results into the final downloadable output. Fetch the parent by name and watch `batchStatus`:

```python
import time
while True:
    # ?jobName= returns the parent ROW directly (no "jobs" wrapper)
    parent = requests.get(f"{BASE}/jobs", headers=HEADERS,
                          params={"jobName": "egfr-binder-screen"}).json()
    bs = parent.get("batchStatus")
    if bs == "Complete":
        break
    if bs in ("Stopped", "AggregationFailed"):
        raise RuntimeError(parent.get("AggregationError", bs))
    time.sleep(15)   # Running / Aggregating -> keep waiting
# When Complete, the parent carries a presigned `resultUrl` and a `statuses`
# subjob tally ({Complete, Running, In Queue, Stopped}).
open("batch.zip", "wb").write(requests.get(parent["resultUrl"]).content)
```

Add `includeSubjobs=true` to `GET /jobs?batch=<name>` to list per-subjob rows.

## Job status lifecycle

Single jobs report `JobStatus`; batch parents report `batchStatus` (poll that for batches — see above).

| Status | Meaning |
|---|---|
| `In Queue` | Accepted, waiting for capacity |
| `Running` | Executing on a worker |
| `Complete` | Finished successfully — results available |
| `Stopped` | Stopped (failure, timeout, manual stop, or budget) |
| `Deleted` | Job was deleted out-of-band |
| `Aggregating` | (batch parent only) subjobs done; building the final output |
| `AggregationFailed` | (batch parent only) aggregation step failed |

Completed jobs carry a `Score` (tool-specific metrics, e.g. pLDDT/pTM/ipTM for folding) and `WeightedHours`. Treat `Complete`/`Stopped`/`Deleted` (and `AggregationFailed` for batches) as terminal; poll on a 15-30s interval. **Break your poll loop on any terminal status, not just `Complete`/`Stopped`** — a job that goes `Deleted` mid-poll would otherwise loop forever. For a `Stopped` job, fetch `getJobLogs(jobName)` to see why.

## Billing — weighted hours

Usage is measured in **weighted hours** (`weighted_hours = raw_hours × GPU_multiplier`), reported per job in the `WeightedHours` field and bounded per batch by `weightedHoursBudget`. A `403` on submit means an org/team budget was exceeded. Don't quote raw cloud $/hour — weighted hours are the unit.

Query usage directly via the `usage-statistics` endpoint:

```python
requests.get(f"{BASE}/usage-statistics", headers=HEADERS,
             params={"statistic": "weighted_hours", "scope": "user"})
# -> {"users": [{"email": ..., "total": <weighted_hrs>, "tools": {<tool>: <hrs>}}]}
# statistic also accepts "jobs"; scope can be "user" or org-wide.
```

## Error handling

| Code | Meaning | Action |
|---|---|---|
| 400 | Bad request / invalid settings | Re-check against the schema; run `validateJob` first |
| 401 | Unauthorized | Check `x-api-key` |
| 403 | Budget exceeded (org/team) | Lower scope or raise the budget |
| 429 | Rate limited | Back off and retry |
| 500 | Server error | Retry; if persistent, contact support |

## Reference files

The `openapi.yaml` spec is the source of truth for endpoint shapes; these files add the behaviors and gotchas the spec doesn't spell out:

- `references/examples.md` — **validated** `settings` payloads per common tool (alphafold/boltz/diffdock/proteinmpnn/batch), a copy-paste self-check, the "what fails and the exact error" list, and output-shape notes. Start here for a working payload.
- `references/api_reference.md` — endpoint quick-reference + the non-obvious shapes: `/jobs` by-name returns a bare row (not `{jobs:[...]}`), `/result` is a two-step download, batch parents poll on `batchStatus`, `/files` is a flat name list, the `settings` field-handling rules.
- `references/tool_catalog.md` — category/tag map, how to read tool + parameter metadata, common tool families.
- `references/workflows.md` — end-to-end recipes: fold a sequence, validate-before-submit, upload + reference a file, design→fold chaining, batch screen with aggregation polling, finetuned-model inference, usage stats, pagination, and the non-blocking submit-now/check-later pattern for long jobs.
