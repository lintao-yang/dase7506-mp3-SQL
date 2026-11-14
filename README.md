# DASE7506 MP3 - SQL Repair Agent

Individual submission for **DASE7506 Mini-project 3: SQL Repair Agent**.

| Metric | Result |
|---|---|
| Official test score | **183 / 240 = 76.25%** |
| Validation | 96 / 120 = 80.00% |
| Starting baseline (submit draft) | 88 / 240 = 36.67% |
| Mean charged tool calls | 3.47 per task (budget 8) |
| CPU prediction time | 2327 s (budget 2 h) |
| Peak memory | 2722 MB (budget 16 GiB) |

The official score document is `outputs/test-score.json` inside the source archive
(`protocol = dase7506-sql-repair-v1`, `n = 240`, `correct = 183`,
`accuracy = 76.25`).

## Submission artifacts

The authoritative, immutable artifacts are the assets of the GitHub Release:

- **Source archive**: `DASE7506_MP3_submission_v2.zip`
  - `submission/submission.py`, `config.json`, `report.pdf`,
    `requirements.txt`, `requirements-model.txt`, `reproduce.md`
  - `outputs/test.json`, `outputs/test-score.json`, `outputs/llm_trace.jsonl`
- **Adapter archive**: `DASE7506_MP3_adapter.zip`
  - exactly `adapter_model.safetensors` and `adapter_config.json`

Direct download links (replace `v1.0` with the published tag if different):

```
https://github.com/lintao-yang/dase7506-mp3-SQL/releases/download/v1.0/DASE7506_MP3_submission_v2.zip
https://github.com/lintao-yang/dase7506-mp3-SQL/releases/download/v1.0/DASE7506_MP3_adapter.zip
```

## Approach

The agent is a deterministic perceive-plan-act loop over the official tools,
followed by a local LoRA repair model for tasks that the fixed shared hints
cannot cover.

1. Check the exact original draft (`check`).
2. Read the fixed shared hints (`repair_hint`), deduplicate them, and check them
   newest-first. Submit the first suggestion that reproduces the public example
   (163 / 240 test tasks).
3. If no candidate matches, read the schema (`schema`) and ask a locally loaded
   Qwen3-0.6B LoRA adapter for up to four corrected queries. Submit the first
   generated query that the real executor confirms against the public example.
4. Otherwise submit the draft.

Every tool decision is derived from the observation history only, so the
official scorer can replay each trajectory exactly. Model sampling is seeded per
task and never enters the trace; only the generated SQL is verified by `check`.

### Model

- Base: `Qwen/Qwen3-0.6B`, revision
  `c1899de289a04d12100db370d81485cdf75e47ca` (unchanged, not included).
- Adapter: LoRA A/B matrices only, rank 16, alpha 32, dropout 0.05, applied to
  `q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj, down_proj`.
- Trainable parameters: 10,092,544 / 606,142,464 (1.665%).
- Adapter size: 40.4 MB.
- Inference: CPU, four threads, bfloat16, four completions per task, at most 128
  new tokens, `--workers 1`.

### Training

- 480 supervised repair pairs from the **training split only**.
- Prompt: schema as `CREATE TABLE` statements with declared foreign keys,
  question, draft, real execution feedback, public example (columns, row count,
  up to five rows).
- Target: training reference SQL from `train_memory.jsonl`.
- 3 epochs, 180 optimizer steps, batch 1 x gradient accumulation 8,
  learning rate 2e-4 cosine, max length 1536, bfloat16, gradient checkpointing,
  one RTX 3060 Laptop GPU, 683 s. Mean training loss 0.267, final logged loss
  0.10.
- Offline hard-task repair rate: 13/25 validation, 23/48 test.

No validation or test reference, grading fingerprint or extra evaluation
database was used to train any component.

## Reproduction

Requirements: Python 3.11 or 3.12 on Linux, CPU only, no network, no GPU.

```bash
python3 -m venv .venv
. .venv/bin/activate

# official package root
python -m pip install -r submission/requirements.txt
python -m pip install -r submission/requirements-model.txt
python -m mp3_sql.check_data

# the base model must be provisioned locally at the declared revision
export MP3_BASE_MODEL=/path/to/Qwen3-0.6B
export MP3_ADAPTER_DIR=/path/to/adapter
export MP3_LLM_LOG=outputs/llm_trace.jsonl
export PYTHONPATH="$PWD/submission"

python -m mp3_sql.predict --split test --workers 1 \
  --method submission/submission.py --checkpoint submission/config.json \
  --output outputs/test.json

python -m mp3_sql.score --split test \
  --predictions outputs/test.json --output outputs/test-score.json
```

Expected score document: `n = 240`, `correct = 183`,
`accuracy = 76.25`. The agent is deterministic; a repeated CPU run on the same
package should reproduce the same predictions.

Unsetting `MP3_BASE_MODEL` or `MP3_ADAPTER_DIR` disables the model stage
gracefully and reproduces the 163/240 hint-verifier result.

## Repository layout

This repository only hosts the release assets and this description. The
submission source, configuration, report, predictions and LLM trace are inside
the source archive; the adapter is inside the adapter archive.

## Data provenance, licenses and attribution

- Benchmark derived from **Spider** (Yu et al., EMNLP 2018),
  <https://yale-lily.github.io/spider>, distributed under **CC BY-SA 4.0**.
- Database mirror: `HAL-9001/spider-databases`, revision
  `4a01bbac6520cd35b216db9e1724e5e1ada60aa4`.
- Draft SQL: `Qwen/Qwen3-0.6B`, revision
  `c1899de289a04d12100db370d81485cdf75e47ca`.
- Shared repair hints: course-provided fixed cache generated by
  `Qwen/Qwen3-8B`, revision
  `b968826d9c46dd6066d109eabc6255188de91218`.
- Execution framework: SQLite and SQLGlot (MIT).
- Course subsets, schema-disjoint sampling, public outputs and the repair
  packaging are modifications under the same CC BY-SA 4.0 data license. This is
  not an official Spider/BIRD leaderboard.

Qwen model outputs and base models retain their applicable model terms; no base
model weights are distributed here.

## External assistance and integrity

- The Trae AI coding assistant was used for code drafting, offline data
  analysis, report editing and experiment orchestration. No external model or
  API is called during grading.
- The agent never reads `test_references.json`, the grading databases or the
  expected-result fingerprints; those are used only by the official scorer.
- Hints are treated as opaque SQL and are only accepted after independent
  execution and fingerprint comparison.
- All model calls, prompts, generated text and token counts are recorded in
  `outputs/llm_trace.jsonl` (48 tasks, 192 generations; prompts average 308
  tokens, generations average 29 tokens).
