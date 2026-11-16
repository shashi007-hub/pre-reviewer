# Problem statement: structured PR analyzer

**Track:** Phase 1, Day 3 (boss level) · **Effort:** about 6 hours over 2–3 sessions · **Dataset:** `pr-dataset/` (13 merged PRs)

---

## 1. Context

Maintainers of popular libraries review hundreds of pull requests. Before reading any code, they want quick answers to a few questions: what changed, does it break the public API, what's risky, and what deserves a comment? You will build a component that answers these as **validated, structured data** that a bot could post on a PR or use to label it.

This is the first building block of Project 2 (the code-review agent). Day 4 (tool calling) and Day 9 (evals) build on it.

## 2. Objective

Build a Python program that takes a unified diff of a pull request and returns a `PRReview` object (schema in section 4) that is:
1. **Valid:** it always passes Pydantic validation.
2. **Grounded:** everything it cites (files, line numbers, signatures) exists in the diff.
3. **Measured:** its accuracy is scored against ground truth computed by code, across all 13 PRs and 4 pipeline configurations.

The goal isn't a perfect reviewer. It's an **honest measurement** of how well an LLM does this task, and which design choices change the result.

## 3. Inputs

- `pr-dataset/diffs/*.diff`: raw unified diffs (`git diff <commit>^ <commit>`)
- `pr-dataset/manifest.json`: PR metadata, including `title` and `description`. You may pass these to the model.
- `pr-dataset/answer_key/api_changes_ast.json`: AST-derived reference API changes. **Don't open it until milestone M2 is done.**

## 4. Output schema (fixed, use as-is)

```python
from typing import Literal
from pydantic import BaseModel, Field

class ApiChange(BaseModel):
    file: str
    symbol: str = Field(description="Function/method name; use Class.method for methods.")
    kind: Literal["added", "removed", "signature_changed"]
    before: str | None = Field(description="Old signature line copied verbatim from removed lines (without the leading '-'), else null.")
    after: str | None = Field(description="New signature line copied verbatim from added lines (without the leading '+'), else null.")
    breaking: bool = Field(description="True if existing callers could break: removed public function, "
                                       "removed/renamed/reordered parameter, new required parameter.")

class ReviewComment(BaseModel):
    file: str
    line: int = Field(description="Line number in the NEW file. Must be an added line.")
    severity: Literal["blocker", "major", "minor", "nit"]
    comment: str = Field(description="Specific and actionable. Reference the code. No praise.")

class FileReview(BaseModel):
    path: str
    change_type: Literal["added", "removed", "modified", "renamed"]
    purpose: str = Field(description="One sentence: what this file's change does.")
    api_changes: list[ApiChange]
    comments: list[ReviewComment] = Field(description="At most 5 per file. Only real issues.")

class PRReview(BaseModel):
    summary: str = Field(description="2-3 sentences for a maintainer.")
    risk: Literal["low", "medium", "high"]
    tests_changed: bool
    files: list[FileReview]
```

You may **add** `Field(description=...)` text to improve first-attempt success. Don't change field names or types. Your metrics depend on them.

## 5. Functional requirements

**R1. Ground truth (no LLM).** `ground_truth(diff_text) -> dict` using `unidiff`, returning:
- `files`: `{path: change_type}`
- `commentable_lines`: `{path: set[int]}`, the new-file line numbers of added lines
- `tests_changed`: `bool`, true if any changed path contains `test`
- `api_changes`: your **own** heuristic (regex over `def` lines) giving `(file, symbol, kind)` tuples

**R2. File filtering.** Skip files that aren't worth reviewing: lock files, `*.min.js`, binaries, and anything you justify in your report. Apply the same rule in every pipeline and in the ground truth.

**R3. Input formats.** Support two ways of presenting a diff to the model:
- **raw:** the diff text unchanged
- **annotated:** every line prefixed with its new-file line number, for example:
  ```
  L41   context line
  L42 + added line
        - removed line          (removed lines have no new-file number)
  ```

**R4. Extraction with semantic checks.** Extend your Day 3 `extract()` with a `checks: Callable[[BaseModel], list[str]]` parameter. After Pydantic validation passes, run the checks. If any problems are returned, send them back to the model (with its previous output) and retry. Default `max_attempts = 3`.

**R5. Checks.** `make_checks(gt)` returns a checker that reports:
- a `file` / `path` not in the diff
- a comment `line` that isn't in `commentable_lines[file]`, **listing the nearest valid lines** in the message
- `before` / `after` text that doesn't appear in the diff (compare after collapsing whitespace)
- a `change_type` that disagrees with `gt["files"]`

**R6. Two pipelines.**
- **single:** one call → `PRReview`
- **map-reduce:** one `extract(..., FileReview)` call per file (map), then one call that receives **only the list of FileReview JSON objects** and returns `summary`, `risk` and `tests_changed` (reduce). Assemble the `PRReview` in code.

**R7. Failure handling.** If a PR still fails after `max_attempts`, record it as failed with the reason, and continue with the next PR. One bad PR must never crash the run.

**R8. Instrumentation.** For every PR × configuration, record: number of API calls, attempts per call, prompt and completion tokens, cost (₹), and wall-clock latency.

## 6. Constraints

- Provider and model: Sarvam `gemma4`, through your `llm.py`, with **schema mode** (`json_schema`, `strict`). It was 10/10 in Step 3.
- `temperature=0` for every call.
- No LLM frameworks (no LangChain, LlamaIndex, etc.). Plain SDK plus Pydantic plus `unidiff`.
- Budget: the full evaluation (13 PRs × 4 configs) should cost **under about ₹50**. Run M3 on the small PRs first to estimate before running everything.
- Large diffs may exceed what fits comfortably in one call. If a single call fails for size reasons, that's a result. Record it rather than working around it.

## 7. Experiments

Run all 13 PRs in each of the four configurations:

| | raw | annotated |
|---|---|---|
| **single** | C1 | C2 |
| **map-reduce** | C3 | C4 |

## 8. Metrics (define them exactly like this)

Let *M* = the model's output and *G* = ground truth, per PR.

| Metric | Definition |
|---|---|
| **Success rate** | % of PRs that produced a valid `PRReview` within the attempt limit |
| **File accuracy** | % of files in *G* whose `change_type` in *M* matches |
| **API precision** | matched / API changes reported in *M* |
| **API recall** | matched / API changes in the reference |
| **Match rule** | same `file`, same `kind`, and the same **last component** of `symbol` (`Client.send` matches `send`) |
| **Public-only P/R** | the same, excluding symbols whose name or any path segment starts with `_` (but keep `__init__`) |
| **First-attempt line validity** | % of comments on the **first** attempt (before any retry) whose line is commentable |
| **Grounding rate** | % of non-null `before`/`after` values found in the diff |
| **Avg attempts, calls, tokens, ₹, seconds** | per PR, plus totals |
| **Comment usefulness** (human) | rate 20 randomly sampled comments from your best configuration as 1 = useless, 2 = plausible, 3 = a maintainer would want it |

Report API P/R twice: against **your** heuristic from R1, and against the **AST answer key**. The gap between the two tells you how good your heuristic is.

## 9. Milestones and checks

| | Milestone | You're done when |
|---|---|---|
| **M1** | Ground truth | `python -m pr_analyzer.ground_truth httpx_2716` prints the files, commentable lines and your API changes |
| **M2** | Heuristic vs. AST | A table of your heuristic's P/R against the answer key for all 13 PRs. *Now* you may open the answer key |
| **M3** | Single pipeline + checks | C1 runs on the 5 small PRs, with retries visible in the logs |
| **M4** | Map-reduce + annotation | C2–C4 run on the 5 small PRs |
| **M5** | Full evaluation | `python -m pr_analyzer.run_eval` runs all 4 configs on all 13 PRs and writes `results/results.csv` plus `results/summary.md` |
| **M6** | Report | `REPORT.md` (section 11) |

## 10. Deliverables

```
genai-foundations/
├── pr-dataset/                 (given)
└── pr_analyzer/
    ├── schema.py               # section 4
    ├── ground_truth.py         # R1, R2
    ├── formatting.py           # R3: raw / annotated
    ├── extract.py              # R4 (moved from day03)
    ├── checks.py               # R5
    ├── pipelines.py            # R6
    ├── evaluate.py             # section 8 metrics
    ├── run_eval.py             # section 7 + R7 + R8
    ├── results/                # results.csv, summary.md, raw outputs per run (JSON)
    └── REPORT.md
```
Save every model output as JSON in `results/raw/<config>/<pr_id>.json`, so you can recompute metrics without paying for new API calls.

## 11. Report (`REPORT.md`, 1–2 pages)

1. **Results table:** the metrics per configuration, plus a breakdown by size bucket (small / medium / large).
2. **Recommendation:** which configuration would you ship, and why? Weigh accuracy against cost and latency.
3. **Failure analysis:** the three most common failure types, each with one concrete example (PR id, what the model said, what was true).
4. **Trap cases:** how did each configuration handle the move (httpx_3388), deprecation vs. removal (fastapi_16520 vs. 16518), and public vs. private (rich_3180)?
5. **Ground-truth limits:** where your heuristic and the AST key disagree, and why.
6. **Comment usefulness:** the human rating, and what separates a useful comment from a useless one.
7. **Next step:** one change you'd make next, and the metric you'd expect it to move.

## 12. Stretch goals (optional)

- **Breaking-change accuracy:** hand-label `breaking` for the public API changes in 5 PRs and score the model.
- **Context:** add the PR `title` and `description` to the prompt and measure the effect.
- **Long-diff strategy:** for the largest file diffs, split by hunk inside the map step.
- **Concurrency:** run the map calls in parallel with a thread pool and report the latency drop.

## 13. Hints

- Start with the smallest PR (`httpx_2716`) and make one full path work before generalising.
- Never re-run a paid call just to recompute a metric. Load the saved JSON.
- If retries rarely help, read the feedback messages you're sending. Vague feedback gives vague fixes.
- `unidiff`: `PatchSet(text)`; per file `f.path`, `f.is_added_file`, `f.is_removed_file`, `f.is_rename`, `f.source_file`, `f.target_file`; per line `line.is_added`, `line.target_line_no`, `line.value`.
- Paths in the diff are prefixed `a/` and `b/`; `unidiff`'s `f.path` already removes them.
