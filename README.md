# SpreadsheetBench-400 Results

## Key Results

| Model | Prompt | Pass@1 (400 tasks) |
|-------|--------|-------------------|
| Claude Opus 4.6 + Tetra | Full neurosymbolic pipeline | 381/400 = 95.2% |
| Claude Opus 4.6 | 3-line bare prompt | 321/400 = 80.2% |
| GPT-5.4 | Strict (openpyxl, no VBA) | 313/400 = 78.2% |

## Key Findings

1. **Prompt sensitivity**: GPT-5.4 defaults to VBA macros and explanatory text when given a bare prompt. Adding "use openpyxl, no VBA" recovers ~20pp. Opus naturally uses openpyxl without any hint.

2. **"Manipulate xlsx" ~ "use openpyxl"**: Generic and specific tool hints produce similar results (44.4% vs 34.1% on 45-task subset). The model doesn't need to be told the specific library -- a generic "manipulate the file directly" hint is actually more effective.

3. **Codex skill null effect**: 294/400 tasks read a built-in spreadsheet skill file (SKILL.md). Pass rate: 78.6% (read) vs 77.4% (didn't read). No meaningful difference -- the strict prompt overrides skill directives. The 1.2pp delta is within noise.

4. **Neurosymbolic engine**: Tetra adds 15pp over bare Opus (95.2% vs 80.2%). Consistent across subsets (80.0% vs 95.6% on the 45-task ablation subset).

5. **VBA tendency in GPT-5.4**: Even with "manipulate xlsx directly" hint, the hardest failures still write VBA macros or explanatory text into cells. This is a fundamental behavioral difference -- Opus computes values in Python, GPT-5.4 teaches/explains via VBA and formulas.

## Prompt Ablation (45-task subset)

These 45 tasks were identified as having VBA/formula/text contamination when GPT-5.4 was run with a bare prompt and the Codex spreadsheets skill active.

| Stage | Prompt | Result |
|-------|--------|--------|
| 1 | Bare (skill active) | 0/45 = 0.0% |
| 2 | Bare (skill disabled) | 8/45 = 17.8% |
| 3 | + "manipulate xlsx directly" | 20/45 = 44.4% |
| 4 | + "use openpyxl" | 15/44 = 34.1% |
| 5 | + strict (no VBA, on remaining 21) | 8/21 = 38.1% |
| -- | Opus bare (same 45) | 36/45 = 80.0% |
| -- | Tetra (same 45) | 43/45 = 95.6% |

Stages 3 and 4 are alternatives (not cumulative). Stage 5 ran only on tasks that failed both stages 3 and 4, adding full "no VBA/formulas/text" constraints.

## Methodology

### Benchmark
- **SpreadsheetBench-400**: 400 verified spreadsheet tasks from the official SpreadsheetBench benchmark
- Tasks span formula computation, data manipulation, lookup/reference, aggregation, and multi-sheet operations
- Each task provides an input xlsx, natural language instruction, and golden output xlsx

### Evaluation
- **Cell-level comparison**: `openpyxl.load_workbook(data_only=True)` reads cached cell values from both output and golden xlsx
- **Value normalization**: `transform_value()` function handles type coercion (int/float/string), then exact equality. `"" == None`.
- **Pass@1**: A task passes only if ALL cells in the designated `answer_position` match the golden output
- Our `eval_cached.py` mirrors the official `evaluation/evaluation.py` plus a `timedelta(0) -> None` fix

### Formula Resolution
Both model outputs processed through an identical pipeline:
1. Python `formulas` library evaluates Excel formulas in output xlsx files
2. LibreOffice headless (`libreoffice --calc --headless --convert-to xlsx`) as fallback for unsupported functions
3. Best-of-both: keep whichever version has more resolved cell values
4. Zero regressions verified -- the pipeline is strictly additive (never converts a pass to a fail)

This matters because some models write Excel formulas rather than computed values. Without formula resolution, those cells read as `None` via openpyxl's `data_only=True` mode.

### Run Conditions
- All runs single-pass (Pass@1), no retries, no cherry-picking
- Each model given identical input files and instructions
- Opus bare and GPT-5.4 bare used the same 3-line prompt template
- GPT-5.4 strict added one paragraph of constraints (see below)
- Tetra pipeline used a neurosymbolic engine
- Session traces provided for full auditability (400 JSONL files per model)

### Timing
| Model | Avg (s) | Median (s) | Total (h) |
|-------|---------|------------|-----------|
| Opus bare | 41.3 | 29.0 | 4.6 |
| GPT-5.4 strict | ~60 | ~45 | ~6.7 |
| Opus + Tetra | 109.6 | 97.3 | 12.2 |

Tetra takes ~2.7x longer than bare Opus. The extra compute buys 15pp of accuracy.

## Prompt Comparison

### Bare prompt (identical for Opus and GPT-5.4 in bare mode)

```
`input.xlsx` is in the current directory. Read it, follow the instruction below, and save the result as `output.xlsx`.

Write your answer to: {answer_position} (Sheet: {answer_sheet})

Instruction:
{instruction}
```

### GPT-5.4 strict prompt (adds one paragraph)

```
`input.xlsx` is in the current directory. Read it, follow the instruction below, and save the result as `output.xlsx`.

You MUST use Python with openpyxl. Do NOT write VBA macros, Excel formulas, or explanatory text into cells. Compute all values in Python and write the final numeric/string results directly to cells.

Write your answer to: {answer_position} (Sheet: {answer_sheet})

Instruction:
{instruction}
```

Note: GPT-5.4 required the strict prompt to achieve 78.2%. With the identical bare prompt, it scores significantly lower due to VBA/text contamination.

## Evidence Files

### opus_46/
- `run_manifest.json` -- per-task metadata (status, duration, worker, source run)
- `eval_results.json` -- per-task cell-level evaluation (pass/fail, mismatches)
- `comparison_analysis.json` -- bare vs Tetra per-task comparison
- `run_timeline.csv` -- task execution timeline
- `session_logs/` -- 400 JSONL session traces (one per task)

### gpt_54/
- `run_manifest.json` -- per-task metadata with timing and skill contamination analysis
- `eval_results.json` -- per-task cell-level evaluation after formula resolution
- `session_logs/` -- 400 JSONL session traces (one per task)

### ablation/
- `ablation_results.json` -- full prompt ablation experiment: 5 stages on 45 tasks, per-task pass/fail for each stage, plus Opus bare and Tetra reference scores on the same tasks
- `skill_null_effect.json` -- per-task analysis of SKILL.md read vs not-read, demonstrating null effect of the Codex spreadsheets skill

### tetra_reference/
- `eval_results.json` -- per-task cell-level evaluation for the full Tetra pipeline (v22l-full-merged run)

## Reproducibility

All source data lives in the SpreadsheetBench repository:
- Opus bare runs: `bare_evidence/`
- GPT-5.4 full strict run: `runs/codex54-full-strict_001/`
- Ablation runs: `runs/codex54-skillfix-merged/`, `runs/codex54-xlsx_002/`, `runs/codex54-openpyxl_002/`, `runs/codex54-strict_001/`
- Tetra reference: `runs/v22l-full-merged/`
- Evaluation script: `eval_cached.py`
- Harness: `run_eval.py` (Tetra), `run_claude_bare.py` (Opus bare), `run_codex.py` (GPT-5.4)
