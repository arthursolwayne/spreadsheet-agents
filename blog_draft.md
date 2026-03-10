# Spreadsheet Agents: What 400 Excel Tasks Reveal About AI Model Behavior

## The Setup

We ran three AI systems against SpreadsheetBench-400 — a benchmark of 400 verified spreadsheet tasks spanning formula computation, data filtering, lookups, text processing, and multi-sheet operations. Each task provides an input xlsx, a natural language instruction, and a golden output xlsx. Pass@1: all cells in the answer region must match exactly.

The three systems:
- **Claude Opus 4.6** with a 3-line bare prompt (no tool hints, no constraints)
- **GPT-5.4** via Codex CLI with a strict prompt (explicit openpyxl, no-VBA constraints)
- **Claude Opus 4.6 + Tetra**, a neurosymbolic pipeline that pairs neural code generation with symbolic formula verification

The headline numbers: **Tetra 95.2%, Opus bare 80.2%, GPT-5.4 strict 78.2%.**

But the headline numbers are the least interesting part of this story.

## Finding 1: The Models Are Closer Than You Think (And Farther Than You'd Expect)

On the full 400 tasks, Opus bare scores 321/400 and GPT-5.4 strict scores 313/400. Eight tasks apart. McNemar's test on the paired results: **p = 0.45**. Not significant. Not close to significant.

But those eight tasks hide a much bigger story. The models don't fail on the same tasks. Of the 400 tasks, 47 are passed by Opus but failed by GPT-5.4, and 39 are passed by GPT-5.4 but failed by Opus. The models have substantially different failure profiles — they just happen to fail on roughly the same *number* of tasks.

This is the first lesson: aggregate accuracy is a lossy compression of model capability. Two models can score within noise of each other while behaving fundamentally differently.

## Finding 2: GPT-5.4 Wants to Teach, Opus Wants to Compute

When given a bare prompt — "read this spreadsheet, follow this instruction, save the result" — the two models adopt completely different strategies.

Opus reads the xlsx with openpyxl, computes values in Python, and writes results to cells. No one told it to use openpyxl. No one told it to compute in Python. It just does.

GPT-5.4 writes VBA macros. It creates Excel formulas. It puts explanatory text in cells ("This formula calculates the sum of..."). On a 45-task subset where we isolated this behavior, GPT-5.4 with a bare prompt scored **0/45**. Not because it couldn't solve the problems — because it expressed its answers in a format the evaluator can't read.

This isn't a capability gap. It's a behavioral default. GPT-5.4's instinct is to *teach* — to leave behind a formula or macro that a human could inspect and understand. Opus's instinct is to *execute* — to compute the answer and write the value.

## Finding 3: Prompt Engineering Recovers 78 Percentage Points

We ran a 5-stage prompt ablation on those 45 contaminated tasks:

| Stage | Prompt | Pass Rate |
|-------|--------|-----------|
| 1. Bare (skill active) | No constraints, Codex skill enabled | 0/45 = 0% |
| 2. Bare (skill disabled) | No constraints, skill disabled | 8/45 = 18% |
| 3. + "manipulate xlsx directly" | Generic tool hint | 20/45 = 44% |
| 4. + "use openpyxl" | Specific library hint | 15/44 = 34% |
| 5. + strict (no VBA/formulas/text) | Full behavioral constraints | 8/21 = 38% |
| -- | **Opus bare (same 45)** | **36/45 = 80%** |
| -- | **Tetra (same 45)** | **43/45 = 96%** |

Every stage transition is statistically significant (McNemar's exact test):
- Bare → xlsx hint: **p = 0.004**
- Bare → openpyxl hint: **p = 0.016**
- xlsx hint vs openpyxl hint: p = 0.39 (no difference — the model doesn't need to be told *which* library, just *that* it should use one)

And on the full 400 tasks with the strict prompt, GPT-5.4 reaches 78.2% — close to Opus's 80.2%. The gap between the models largely collapses once you fix the behavioral default. The "capability" was always there; it was masked by a formatting problem.

This is a 78-percentage-point swing from prompt engineering alone. Same model, same tasks, same evaluation — the only variable is the instruction text.

## Finding 4: The Codex Skill File Does Nothing

Codex ships with a built-in spreadsheets skill file (`SKILL.md`) that instructs the model to use `@oai/artifact-tool`, a proprietary JavaScript runtime for spreadsheet manipulation. We found that 294 of 400 tasks triggered reading this file — even when we renamed the skill directory to `spreadsheets.disabled/`.

The natural question: does this skill help or hurt?

Pass rate with skill read: 231/294 = **78.6%**
Pass rate without skill read: 82/106 = **77.4%**

Delta: 1.2 percentage points. Within noise. **The skill file has null effect.** The strict prompt's behavioral constraints override whatever the skill file says, making it irrelevant.

This matters for benchmarking: you don't need to worry about the skill contaminating your results, as long as your task prompt is specific enough about output format.

## Finding 5: Symbolic Verification Buys 15 Points

Tetra pairs Opus with a symbolic formula engine. The model writes check rows — essentially test assertions expressed as spreadsheet formulas — and the engine evaluates them. If the checks fail, the model iterates. It's TDD for spreadsheets: the formula engine doesn't hallucinate, so if `=SUM(A1:A10)` returns 42 and your answer says 43, you get an objective signal that you're wrong.

On the full 400: Tetra scores **381/400 = 95.2%** vs Opus bare at **321/400 = 80.2%**. McNemar's test: **p < 0.000001**.

On the 45 hardest tasks (the VBA-contaminated subset): Tetra scores **43/45 = 95.6%** vs Opus bare at **36/45 = 80.0%**. Even here, where the tasks specifically trip up models with formatting issues, the symbolic verification loop catches errors that pure neural generation misses.

The extra accuracy costs compute: Tetra averages 110 seconds per task vs 41 seconds for bare Opus (2.7x). But 15 percentage points of accuracy for 2.7x compute is a trade most production systems would take.

## Finding 6: Task Taxonomy Reveals Where Models Diverge

We classified all 400 tasks into 7 categories using LLM-based qualitative coding (open coding → axial coding):

| Category | N | Tetra | Opus | GPT-5.4 |
|---|---|---|---|---|
| Conditional Aggregation | 84 | 94% | 77% | 81% |
| Lookup & Cross-Reference | 77 | 94% | 83% | 79% |
| Data Filtering & Sorting | 74 | 97% | 74% | 68% |
| Formula & Specialized Calc | 54 | 94% | 80% | 81% |
| Classification & Conditional | 43 | 98% | 91% | 91% |
| Text & String Processing | 39 | 95% | 82% | 72% |
| Data Restructuring | 29 | 97% | 79% | 79% |

Tetra dominates every category (94-98%). The bare models show more interesting variation:

- **Data Filtering & Sorting** is the hardest category for both bare models (Opus 74%, GPT-5.4 68%) — and the one where Tetra's uplift is largest (97%). The symbolic verification loop is most valuable when the model needs to get sort orders, filter conditions, and row counts exactly right.

- **Classification & Conditional** is easiest for both (91% each). These tasks have clear rules and binary outputs — less room for formatting or precision errors.

- **Text & String Processing** shows the widest bare-model gap: Opus 82% vs GPT-5.4 72%. These tasks involve exact whitespace preservation, concatenation, and string formatting — areas where GPT-5.4's tendency to write explanatory text is most damaging.

## The Real Takeaway

The interesting comparison isn't model A vs model B on aggregate accuracy. It's this:

1. **Behavioral defaults matter more than raw capability.** GPT-5.4 and Opus have similar solve rates when prompted correctly. But GPT-5.4 needs to be told what *not* to do (no VBA, no formulas, no text), while Opus just does the right thing. For production deployment, the model that requires less prompt engineering is the one you want — not because it's "smarter," but because it's less fragile.

2. **Symbolic verification is the real multiplier.** The 15pp gap between bare Opus and Tetra is larger than the gap between the two bare models. And it's the only gap that's unambiguously significant. Pairing neural generation with a deterministic verifier catches errors that no amount of prompt engineering can prevent.

3. **Benchmark scores are not model scores.** They're system scores — model + prompt + evaluation pipeline + random seed. Changing the prompt swings GPT-5.4 by 78 percentage points on a 45-task subset. Changing the evaluation pipeline (adding formula resolution) recovers tasks that would otherwise read as failures. A single Pass@1 number, without the full system specification, is not reproducible and not comparable.

## Methodology Notes

**Evaluation**: Cell-level comparison using `openpyxl.load_workbook(data_only=True)`. Value normalization via `transform_value()` (type coercion, exact equality, `"" == None`). All cells in the designated answer position must match.

**Formula resolution**: Both models' outputs processed through an identical pipeline — Python `formulas` library + LibreOffice headless, best-of-both. This resolves formulas that openpyxl can't evaluate. Zero regressions verified (strictly additive).

**Statistical tests**: McNemar's exact test (two-sided) for all paired comparisons. Appropriate because the same 400 tasks are shared across models — samples are paired, not independent.

**Run conditions**: All runs single-pass (Pass@1), no retries, no cherry-picking. Identical input files and instructions (modulo prompt mode). Session traces provided for full auditability (400 JSONL files per model).

**Timing**: Opus bare avg 41s, GPT-5.4 strict avg ~60s, Tetra avg 110s.

All evidence — session logs, evaluation results, ablation data, and per-task classifications — is available at [github.com/arthursolwayne/spreadsheet-agents](https://github.com/arthursolwayne/spreadsheet-agents).

## Conclusion

The number everyone wants is 95.2% vs 80.2% vs 78.2%. And the number is the wrong thing to look at.

What this experiment actually measured is that benchmark scores are system scores, not model scores. Each of those three numbers is the product of a specific model, a specific prompt, a specific evaluation pipeline, and whatever stochastic state the model happened to land in on that particular run. Change the prompt and GPT-5.4 swings 78 percentage points — from 0% to 78% — on the same tasks with the same model weights. That is not a rounding error. That is the prompt doing more work than the model. Anyone who publishes a single Pass@1 number without specifying the full system — exact prompt text, tool configuration, output processing pipeline, formula resolution method — has published a number that no one else can reproduce, and that no one should compare to any other number.

The model-vs-model comparison is the least interesting finding in this entire study. Opus bare beats GPT-5.4 strict by 8 tasks out of 400. McNemar's test gives p = 0.45. That is not a real difference. You could flip a coin and get a more significant result. The models fail on different tasks — 47 passed by Opus alone, 39 passed by GPT-5.4 alone — but they fail on the same *number* of tasks. If you showed someone only the aggregate scores, they would conclude the models are interchangeable. If you showed someone only the per-task disagreements, they would conclude the models are fundamentally different. Both conclusions are correct, which is why aggregate accuracy is such a poor summary statistic.

What *is* statistically significant — overwhelmingly, unambiguously significant — is the effect of symbolic verification. Tetra's 15-percentage-point advantage over bare Opus carries a p-value below 0.000001. This is the one result in the study that you can take to the bank. The formula engine does not hallucinate. When `=SUM(A1:A10)` evaluates to 42 and the model's answer says 43, that is an objective, deterministic signal that the answer is wrong. No amount of prompt engineering provides the same guarantee, because prompt engineering operates on the model's *intentions* while symbolic verification operates on the model's *outputs*. One is a suggestion; the other is a proof.

The mechanism is worth being specific about. Consider task 59129. The instruction asks the model to count how many items were "open at end of month" for each of 12 months. Opus bare gets 11 of the 12 months right. It computes January through November correctly. For one month — the one with a boundary condition where an item opens and closes on the same day — it returns 21 instead of 20. One cell wrong out of 12. Accuracy: 91.7%. Result: fail. Pass@1 requires perfection.

Now imagine reviewing that output by hand. You are looking at a 22-row spreadsheet with 12 answer cells. The numbers look reasonable. They are in the right ballpark. You would have to manually recount the items that were open on the last day of that specific month, cross-referencing open dates and close dates, to catch a single off-by-one. No human reviewer does that. The check row does. Tetra writes `=COUNTIFS(open_date,"<="&EOMONTH(month,0),close_date,">="&EOMONTH(month,0))` — or something equivalent — and the formula engine evaluates it against the data. The model gets immediate, cell-level feedback: this one is wrong. It iterates. It fixes the boundary condition. It passes.

This is TDD for spreadsheets, and it works for the same reason TDD works in software engineering: it converts vague correctness ("the numbers look right") into specific, falsifiable assertions ("this cell equals this value"). The formula engine is the test runner. The model is the developer. The check rows are the test suite. And unlike a human reviewer, the test runner checks every cell, every time, with zero fatigue and zero probability of overlooking a single off-by-one in row 14.

The behavioral default finding is subtler but arguably more important for practitioners. GPT-5.4 and Opus do not differ primarily in what they *can* do — they differ in what they *choose* to do when you don't tell them what to do. Opus reads a spreadsheet task and writes Python that computes values and saves them to cells. GPT-5.4 reads the same task and writes VBA macros, Excel formulas, and explanatory annotations. Both approaches reflect reasonable interpretations of the instruction. But only one produces output that a cell-value evaluator can score. The model that defaults to the right behavior for your evaluation pipeline will always outperform the model that defaults to the wrong behavior, regardless of which model is "smarter" in some abstract sense. This is not a capability gap. It is a compatibility gap. And it vanishes entirely with three lines of prompt engineering — which is precisely why aggregate benchmark scores without prompt specifications are meaningless.

So what should you actually take away from this?

If you are building a production system that uses LLMs to manipulate spreadsheets, the single highest-leverage investment is not model selection. It is symbolic verification. Pair your model — whichever model — with a formula engine that can evaluate check assertions against the model's output. The 15 percentage points you gain from this are real, significant, and robust across task categories. The 2 percentage points you gain from picking the "right" model are noise.

If you are publishing benchmark results, specify your system. The prompt matters. The tool configuration matters. The output processing pipeline matters. A number without a system specification is not a measurement — it is an anecdote.

And if you are evaluating whether an AI system "can do spreadsheets," do not look at the aggregate score. Look at task 59129. Look at the model that gets 11 out of 12 months right and misses one boundary condition that no human reviewer would catch in a 22-row spreadsheet. Ask yourself whether 91.7% accuracy is good enough for your use case. Then ask yourself whether you would have caught the error. The formula engine catches it every time. That is the entire argument for neurosymbolic systems, in one cell.

## Future Work

This evaluation answered some questions and raised more. Here's what we'd run next.

### More Models, Same Benchmark

We tested two frontier models. That's not enough to draw general conclusions about behavioral defaults. Gemini 2.5 Pro is the obvious next candidate — Google has invested heavily in structured data reasoning, and Gemini's function-calling behavior may diverge from both Opus's compute-first instinct and GPT-5.4's teach-first instinct in ways we can't predict. Does Gemini default to writing Google Sheets formulas? Does it hallucinate cell references at a different rate? We don't know.

Open-source models are equally interesting but for different reasons. Llama 4, Qwen 3, and DeepSeek-R1 lack the RLHF-shaped behavioral defaults that commercial models carry. A model without a strong prior toward "be helpful by explaining your work" might score better on SpreadsheetBench out of the box — or it might fail in entirely new ways (wrong library imports, malformed xlsx writes, silent data truncation). The failure taxonomy from Finding 6 gives us a framework to classify those failures without re-doing the qualitative coding from scratch.

The practical question: does the VBA behavioral default we observed in GPT-5.4 exist in other models, or is it specific to OpenAI's training pipeline? If Gemini also defaults to formula-writing, that's evidence of a systematic training bias in RLHF'd models toward "show your work" outputs. If it doesn't, the bias is OpenAI-specific and the 78pp prompt sensitivity finding is less generalizable than it appears.

### Multi-Run Variance Estimation

Every number in this post is from a single Pass@1 run. That's the standard benchmark protocol and it's also the protocol's biggest weakness. We have no variance estimates. When we say Opus scores 80.2%, is that 80.2% +/- 0.5% or 80.2% +/- 3%?

Our experience with the Tetra pipeline suggests the variance is non-trivial. During development, we tracked per-task reliability across multiple runs of the same code and prompt. Some tasks are deterministic — they pass or fail 100% of the time. Others are stochastic: a task might pass 2 out of 3 runs, or 1 out of 5. We identified at least 10 tasks (out of 400) where the pass/fail outcome varied across runs with identical prompts and code. At the aggregate level, this means the true Pass@1 rate for any system has a confidence interval that single-run evaluation doesn't capture.

The right experiment: run each system 5 times (2,000 tasks total per system, ~$9,000 per model at current API pricing) and report both the mean and the per-task pass rate distribution. This would let us distinguish three categories of tasks: (1) tasks the model always solves, (2) tasks the model never solves, and (3) tasks where the outcome depends on the sample. Category 3 is where the interesting engineering questions live — those are the tasks where prompt changes, symbolic verification, or retry strategies have the highest marginal value.

This also has implications for model comparison. Our McNemar's test on Opus vs GPT-5.4 returned p = 0.45. But that test assumed each task's outcome is fixed. If 10% of tasks are stochastic, the paired test is comparing noisy observations and may be underpowered or miscalibrated. Multi-run data would let us use a proper mixed-effects model with task-level random intercepts, which is the right statistical framework for this kind of benchmark.

### Harder Benchmarks and Larger Spreadsheets

SpreadsheetBench-400 tasks are, by design, solvable by a competent Excel user in minutes. The spreadsheets are small (typically under 1,000 rows, a handful of sheets). The instructions are unambiguous. The answer regions are well-defined.

Real-world spreadsheet tasks are messier. Enterprise spreadsheets routinely exceed 100,000 rows. They span dozens of sheets with cross-references that form dependency graphs. Instructions come as email threads, not clean single-sentence prompts. The answer isn't a contiguous cell range — it's "update the Q3 forecast tab to reflect the revised assumptions in the email from Tuesday."

We'd want to test along three axes of difficulty:

**Scale.** How does accuracy degrade as spreadsheet size increases? Our current tasks fit comfortably in a model's context window. At 100K rows, the model either needs to stream the data, summarize it, or operate on it in chunks. The symbolic verification loop in Tetra becomes more valuable here — the formula engine can validate results on data the model never fully "saw."

**Ambiguity.** SpreadsheetBench instructions are precise. What happens when the instruction is "fill in the missing values" with no further specification? The model has to infer the pattern from context. This is closer to how people actually use spreadsheets, and it's where behavioral defaults (GPT-5.4's tendency to explain, Opus's tendency to compute) might matter even more.

**Multi-step reasoning.** Some SpreadsheetBench tasks require chaining 3-4 operations (filter, group, aggregate, format). Real financial models require dozens of dependent steps. Error propagation through a long chain is qualitatively different from error on a single step — and symbolic verification's value should increase with chain length, since each check row catches errors before they compound.

There's also the question of spreadsheet-adjacent formats. CSV files, Google Sheets (via API), database exports with xlsx wrappers. Each format introduces its own failure modes. A model that handles openpyxl xlsx flawlessly might fail on a CSV with mixed encodings or a Google Sheet with conditional formatting that doesn't survive the export.

### Symbolic Verification Beyond Spreadsheets

The Tetra pipeline's core insight is domain-general: pair a neural generator with a deterministic verifier, and let the verifier's feedback drive iteration. Spreadsheets happen to be a clean testbed because the verifier (a formula engine) is well-defined, fast, and unambiguous. But the same architecture applies anywhere you have a formal specification language.

**SQL.** The model writes a query; the verifier runs it against a test database and checks the result set. This is already done informally in text-to-SQL benchmarks, but not as an in-loop feedback mechanism. The model should be able to write its own test queries ("this JOIN should produce exactly 47 rows") and iterate against a real database engine.

**Unit tests.** The model writes code; the verifier runs a test suite. This is the most obvious application and the one closest to existing practice (SWE-bench, HumanEval). But most implementations treat test execution as a one-shot evaluation signal, not as an iterative feedback loop. The Tetra pattern — write check assertions, execute, iterate until checks pass, then submit — could improve Pass@1 rates on code generation benchmarks the same way it improved spreadsheet accuracy by 15pp.

**Formal proofs.** The model writes a proof step; a proof assistant (Lean, Coq, Isabelle) checks it. This is the highest-fidelity version of the pattern — the verifier is provably correct, not just empirically reliable. The challenge is that proof assistants give cryptic error messages, so the feedback loop requires the model to interpret type errors and unification failures. But that's a prompt engineering problem, not an architectural one.

**Data pipelines.** The model writes a transformation (pandas, dbt, Spark); the verifier runs it against sample data and checks invariants (row counts preserved, no nulls in required columns, referential integrity maintained). This is closest to the spreadsheet case and would be the most natural next step.

The key question in each domain: how much does the symbolic verification loop buy over a single-pass neural generation? Our spreadsheet result is 15pp. If the uplift is consistent across domains, that's evidence for a general principle. If it varies widely, we need to understand why — is it the quality of the verifier's feedback? The difficulty of the tasks? The model's baseline accuracy? Mapping this curve across domains would be a significant contribution.

### Prompt Sensitivity as a First-Class Measurement

Finding 3 — the 78pp swing from prompt engineering — is, in some ways, the most important result in this post, and we only measured it on one model and one task subset. The natural question is whether this level of prompt sensitivity is specific to GPT-5.4's VBA default, or whether it's a general property of frontier models on structured output tasks.

We'd want to run a systematic prompt ablation on every model we test, not just the one that happens to have a problematic default. The ablation dimensions:

**Output format constraints.** "Write values, not formulas." "Use openpyxl, not xlsxwriter." "Don't include explanatory text." Each constraint is a binary toggle. With 4-5 constraints, a full factorial design is 16-32 runs per model — expensive but informative. The goal isn't to find the best prompt for each model; it's to measure how much each constraint changes the outcome. If constraint X swings accuracy by 20pp on model A and 0pp on model B, that tells us something about how those models were trained.

**Instruction specificity.** SpreadsheetBench instructions range from very specific ("compute the sum of column B for rows where column A equals 'Yes'") to moderately ambiguous ("fill in the summary table"). Does prompt sensitivity correlate with instruction ambiguity? Our intuition says yes — the more ambiguous the instruction, the more the model's behavioral default determines the output format, and the more a format-constraining prompt matters.

**Cross-task-type generalization.** Does the VBA default hurt GPT-5.4 on other structured output tasks? Code generation (writing to files, not explaining in comments)? Data analysis (producing numbers, not narratives)? Report generation (filling templates, not writing prose)? If the "teach vs compute" behavioral split we observed generalizes beyond spreadsheets, it has implications for how models should be trained and how benchmarks should be designed.

### GPT-5.4 VBA Default Stability Across Model Versions

We observed the VBA behavioral default in GPT-5.4 as tested in early 2026. But model behavior drifts. OpenAI ships model updates, adjusts RLHF weights, and fine-tunes on new data. The natural question: is the VBA default a stable property of the GPT-5.4 model family, or will it silently disappear in a future checkpoint?

This matters for reproducibility. If someone runs our evaluation in six months and gets different results, is it because the model changed or because they did something different? Model versioning is opaque — "GPT-5.4" might refer to different weights at different points in time. We'd want to re-run the 45-task VBA ablation quarterly against whatever "GPT-5.4" resolves to, tracking whether the bare-prompt pass rate drifts. If OpenAI fixes the VBA default through training, the 78pp sensitivity finding becomes a historical artifact rather than a current concern. If it persists across checkpoints, it's a structural property of their RLHF pipeline — something competitors and researchers need to account for.

More broadly, the question applies to all behavioral defaults we observed. Does Opus's compute-first instinct persist across Opus versions? If Anthropic releases Opus 5 and it starts writing formulas instead of computing values, our entire baseline shifts. Longitudinal tracking of behavioral defaults across model versions — not just accuracy numbers, but the *strategies* models choose — would be a useful contribution to the evaluation literature. Nobody currently tracks this. Everyone should.

There's a deeper question here about benchmark validity over time. SpreadsheetBench-400 is a fixed dataset. Models improve. At some point, all three systems will score 95%+ and the benchmark will stop discriminating between them. The useful life of a benchmark is bounded by the pace of model improvement. We don't know where that ceiling is for SpreadsheetBench-400, but the Tetra result (95.2%) suggests we're approaching it. The remaining 19 failures include 3 benchmark defects (the "correct" answer is wrong or date-dependent) and several that require capabilities no current model has (e.g., exact numerical agreement on 480-value arrays). The benchmark's discriminative power is concentrated in maybe 50-80 tasks — the rest are "always pass" for frontier models.

Building harder benchmarks is one response. But the more productive response might be to shift from "how many tasks does the model pass?" to "how does the model fail?" — which is what this post tried to do. The failure taxonomy, the behavioral default analysis, and the symbolic verification uplift are all findings that survive benchmark saturation. They're about the structure of model behavior, not the score.
