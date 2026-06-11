# Tamarind Bio — validated examples & output shapes

Every `settings` payload below was confirmed with `validateJob` against the live
API (`valid: true`) using real values. Tool schemas evolve — if one stops
validating, re-fetch with `getJobSchema(<tool>)` / `GET /tools`. Sequences here are
illustrative; swap your own.

**File params (`proteinFile`, `pdbFile`, `ligandFile`, …) need a real file value** —
either an uploaded/prior-job **path** (`{email}/target.pdb`, `JobName/out/x.pdb`) or
**inline PDB/SDF-format text** (multi-line `ATOM`/`HETATM` records). The
`<...>` placeholders below are NOT valid as written — replace them. **Do not put an
amino-acid sequence in a file param** — `validateJob` rejects it with
`File ... must be of types: ["pdb"]`. (A sequence goes in `sequence`, a structure
goes in a file param.)

`BASE = "https://app.tamarind.bio/api"`, `HEADERS = {"x-api-key": <key>}`.

## Self-check (run this first to confirm the skill works for you)

Read-only + dry-run, no submission, no cost. Confirms the discover → schema →
validate loop end-to-end:

```python
import os, requests
BASE, HEADERS = "https://app.tamarind.bio/api", {"x-api-key": os.environ["TAMARIND_API_KEY"]}

# 1. discovery reachable?
tools = requests.get(f"{BASE}/tools", headers=HEADERS).json()
assert isinstance(tools, list) and any(t["name"] == "alphafold" for t in tools), "tools endpoint"

# 2. validate a known-good payload (MCP validateJob; or skip if REST-only)
#    expect {"valid": true, ...}
```

With the MCP server: `validateJob(jobName="selfcheck", type="alphafold",
settings={"sequence": "MKTAYIAKQRQISFVKSHFSRQLEERLGLIE"})` → `valid: true`.

## Validated input payloads

### AlphaFold — monomer
```json
{ "sequence": "MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQAPILSRVGDGTQDNLSGAEKAVQVKVKALPDAQFEVVHSLAKWKR",
  "numModels": "1", "numRecycles": 3 }
```
Only `sequence` is required; everything else has a default. `numModels` is a string
dropdown (`"1"`–`"5"`).

### AlphaFold — multimer (colon-separated chains)
```json
{ "sequence": "MKTAYIAKQRQISFVKSHFSRQLEERLGLIE:DIQMTQSPSSLSASVGDRVTITCRASQSISSYLN" }
```
Join chains with `:`. No separate "multimer" flag — chain count drives it.

### Boltz-2 — sequence mode
```json
{ "inputFormat": "sequence",
  "sequence": "MKTAYIAKQRQISFVKSHFSRQLEERLGLIEVQAPILSRVGDGTQDNLSGAEKAVQVKVKALP" }
```
`inputFormat` is **required** (`"sequence"` / `"list"` / `"molecules"` / `"yaml"`).
Omitting it fails — see "What fails" below.

### DiffDock — protein + SMILES ligand
```json
{ "ligandFormat": "SMILES",
  "ligandSmiles": "CC(=O)Oc1ccccc1C(=O)O",
  "proteinFile": "<uploaded-path-or-inline-PDB-text>" }
```
`ligandFormat` chooses the conditional field: `"SMILES"` → `ligandSmiles`;
`"sdf/mol2 file"` → `ligandFile`. `proteinFile` is a file param — pass an uploaded
path (`{email}/target.pdb` or `JobName/...`) or inline PDB text (see file-input
rules in `api_reference.md`).

### ProteinMPNN — design residues on a backbone
```json
{ "pdbFile": "<uploaded-path-or-inline-PDB-text>",
  "designedResidues": { "A": "1 2 3 4 5" },
  "numSequences": 4, "modelType": "proteinmpnn" }
```
Requires `pdbFile` + `designedResidues` (per-chain, space-separated resnums).
`modelType` ∈ `proteinmpnn`/`ligandmpnn`/`solublempnn`/`hypermpnn`/`abmpnn`.
Note `designedChains` is `exclude:["api"]` — don't send it over the API.

### Batch (same tool, many jobs)
```json
{ "batchName": "screen-1", "type": "alphafold",
  "jobNames": ["s1", "s2"],
  "settings": [ { "sequence": "MKT..." }, { "sequence": "AVF..." } ] }
```

## What fails (and the exact error) — confirmed live

- **Boltz without `inputFormat`** → `valid:false`, `Missing required boltz field "inputFormat"`. Always check required fields with `getJobSchema` first; `sequence` alone is not enough for boltz/chai.
- **Round-tripping `validateJob` normalized into a submit** → `valid:false`, `Unrecognized setting: "submit_method"`. The `normalized` echo carries platform-internal fields (`submit_method`, `msa`); submit YOUR clean settings, not the echo.
- **File param given a bare string that isn't a real path** → treated as INLINE file content (uploaded as `<email>/<jobname>-<param>.<ext>`), not a reference. To point at an existing object use `{email}/{filename}` or `JobName/...`. Referencing a path that doesn't exist → `File ... has not been uploaded`.

## Output shapes (describe, don't expect exact values)

Outputs are non-deterministic (seed/model/MSA) — reason about the *shape*, not
golden numbers.

- **Job row `Score`** (JSON string on completed jobs): tool-family dependent.
  - Folding (alphafold/boltz/chai/esmfold): `plddt`, `ptm`, and for complexes
    `iptm` plus interface metrics (`ipSAE_*`, `pDockQ_*`). Higher pLDDT/pTM = more
    confident; iptm/ipSAE gauge interface quality.
  - Other families carry their own metrics — read the keys, don't assume.
- **Results zip** (`POST /result` → presigned URL → GET): per-tool, typically the
  structure files (`rank_*.pdb` / `*.cif`), a scores CSV, and logs. Use
  `listJobFiles(jobName)` (MCP) to enumerate exact filenames before downloading.
- **`WeightedHours`** on the row is the billing unit (see `usage-statistics`).

To learn a specific tool's exact outputs, run one small job and `listJobFiles` it —
don't hardcode filenames, which vary by tool and version.
