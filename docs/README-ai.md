> **Status: Reference — README draft + coaching**
> This document is a raw chat export that contains an early README draft and strategic framing advice. It is not the canonical README. For the current README, see [README.md](README.md). For operational governance, see [AGENTS.md](AGENTS.md).

---

README cleanup + AGENTS.md

Absolutely. And that framing is actually smart.

You’re not trying to pretend you hand-coded everything from scratch. You’re documenting a very real modern workflow:

you designed the research direction
AI helped generate and refine the implementation
you curated, steered, and integrated the system

That is legitimate authorship of a different kind: AI-directed systems design.

Since you want this to be clear, the best way is:

make the repo structure and README consistent
add an AGENTS.md that explicitly explains the AI-assisted development process
frame yourself as the research designer / benchmark architect / AI-directed builder, not as a traditional line-by-line programmer

Below I’ll give you both:

a cleaned-up README
a strong AGENTS.md

You can copy both directly.

Recommended repo structure

Before the files, here’s the naming scheme I recommend so the README and repo stay aligned:

.
├── README.md
├── AGENTS.md
├── data/
│   ├── v1/
│   │   ├── constraint_stacking_train.jsonl
│   │   ├── constraint_stacking_val.jsonl
│   │   ├── constraint_stacking_test.jsonl
│   │   ├── extraneous_load_train.jsonl
│   │   ├── extraneous_load_val.jsonl
│   │   ├── extraneous_load_test.jsonl
│   │   ├── chunking_train.jsonl
│   │   ├── chunking_val.jsonl
│   │   ├── chunking_test.jsonl
│   │   ├── element_interactivity_train.jsonl
│   │   ├── element_interactivity_val.jsonl
│   │   └── element_interactivity_test.jsonl
│   └── v2/
│       ├── chunking_v2_test.jsonl
│       ├── extraneous_load_v2_test.jsonl
│       └── element_interactivity_v2_test.jsonl
├── scripts/
│   ├── generate_v1_benchmarks.py
│   ├── generate_v2_benchmarks.py
│   ├── evaluate_predictions.py
│   ├── plot_v1_results.py
│   ├── run_model_openai_compatible.py
│   ├── multi_model_orchestrator.py
│   ├── comparison_plotter.py
│   ├── paired_orchestrate_v2.py
│   ├── paired_comparison_plotter.py
│   ├── cross_model_significance.py
│   ├── cross_model_permutation.py
│   ├── cross_model_permutation_corrected.py
│   └── forest_plot_corrected.py
├── runs/
│   └── v1/
├── paired_v2_runs/
├── plots/
│   ├── v1/
│   └── v2/
└── forest_plots_corrected/


That structure is simple, legible, and makes the README much cleaner.

Cleaned-up README.md
# Cognitive Load Benchmarks for LLMs

A benchmark and analysis pipeline for testing whether language models exhibit **bounded-capacity failure patterns analogous to cognitive overload**.

This repository explores whether LLM performance changes systematically when we manipulate:

- **presentation structure**
- **relational complexity**
- **distractor interference**
- **constraint density**

The goal is not to claim that LLMs experience human cognition literally.  
The goal is to test whether they behave like **capacity-limited information processors** in ways that are usefully described by cognitive load theory.

## Project thesis

Cognitive Load Theory separates load into:

- **Intrinsic load**: complexity inherent to the task
- **Extraneous load**: irrelevant or badly presented information that interferes
- **Germane load**: useful organization that improves learning or processing

This project operationalizes that frame for LLMs.

We test whether models show signatures like:

- better performance when information is **chunked**
- worse performance when dependency structure is **highly interactive**
- disproportionate degradation from **confusable** or **contradictory** distractors
- collapse as active **constraint count** increases

## What this repository contains

This repo includes:

- synthetic benchmark generators
- model runners for OpenAI-compatible APIs
- evaluators
- multi-model orchestration scripts
- paired-benchmark analysis
- bootstrap and permutation-based cross-model testing
- multiple-comparison correction
- corrected forest plots

## Benchmark families

### V1 benchmark families

These are synthetic benchmark families with condition-level comparisons.

#### 1. Constraint Stacking
Tests whether performance degrades as the number of simultaneous constraints increases.

Expected pattern:
- accuracy falls as constraint count rises
- smaller or weaker models collapse faster
- errors become structured rather than random

#### 2. Extraneous Load
Tests whether extra information harms performance differently depending on distractor type.

Conditions include:
- irrelevant filler
- confusable distractors
- contradictory distractors

Expected pattern:
- confusable and contradictory distractors hurt more than harmless filler

#### 3. Chunking
Tests whether grouping information into digestible structure improves performance relative to flat presentation.

Expected pattern:
- chunked formatting improves outcomes
- smaller models benefit more

#### 4. Element Interactivity
Tests whether tasks with stronger dependency coupling are harder than tasks with similar surface length but weaker coupling.

Expected pattern:
- high interactivity produces sharper degradation than low interactivity

### V2 paired benchmark families

These are stronger matched-item benchmark designs.

Instead of comparing unrelated examples across conditions, V2 benchmarks create **matched variants of the same underlying item**.

This allows **within-item paired effect estimation**.

#### 1. `chunking_v2`
Same underlying task:
- `flat`
- `chunked`

Primary paired effect:

$$
\Delta_{\text{chunk}} = \mathbb{E}[\text{chunked correct} - \text{flat correct}]
$$

#### 2. `element_interactivity_v2`
Same underlying task:
- `low`
- `high`

Primary paired effect:

$$
\Delta_{\text{interactivity}} = \mathbb{E}[\text{low correct} - \text{high correct}]
$$

#### 3. `extraneous_load_v2`
Same underlying task:
- `irrelevant`
- `confusable`
- `contradictory_adjacent`

Primary paired effects:

$$
\Delta_{ir-conf} = \mathbb{E}[\text{irrelevant correct} - \text{confusable correct}]
$$

$$
\Delta_{ir-contra} = \mathbb{E}[\text{irrelevant correct} - \text{contradictory\_adjacent correct}]
$$

$$
\Delta_{conf-contra} = \mathbb{E}[\text{confusable correct} - \text{contradictory\_adjacent correct}]
$$

## Recommended repository structure

```text
.
├── README.md
├── AGENTS.md
├── data/
│   ├── v1/
│   └── v2/
├── scripts/
├── runs/
│   └── v1/
├── paired_v2_runs/
├── plots/
│   ├── v1/
│   └── v2/
└── forest_plots_corrected/

Script index
Data generation
scripts/generate_v1_benchmarks.py
scripts/generate_v2_benchmarks.py
Evaluation and plotting
scripts/evaluate_predictions.py
scripts/plot_v1_results.py
Model running
scripts/run_model_openai_compatible.py
V1 multi-model orchestration
scripts/multi_model_orchestrator.py
scripts/comparison_plotter.py
V2 paired orchestration
scripts/paired_orchestrate_v2.py
scripts/paired_comparison_plotter.py
Cross-model statistics
scripts/cross_model_significance.py
scripts/cross_model_permutation.py
scripts/cross_model_permutation_corrected.py
Corrected forest plots
scripts/forest_plot_corrected.py
Installation

Python 3.10+ recommended.

Install dependencies:

pip install matplotlib numpy openai

OpenAI-compatible API support

The runner and orchestration scripts use OpenAI-compatible APIs.

This means they can work with:

OpenAI
compatible routers
RouteLLM-compatible endpoints
RouteLLM

Base URL: RouteLLM API

Docs: API docs

Sample page: RouteLLM app

Example model:

route-llm
Quickstart
1. Generate datasets
python scripts/generate_v1_benchmarks.py
python scripts/generate_v2_benchmarks.py

2. Run one model on one dataset
python scripts/run_model_openai_compatible.py \
  --dataset data/v2/chunking_v2_test.jsonl \
  --output predictions_chunking_v2.jsonl \
  --model route-llm \
  --base-url https://routellm.abacus.ai/v1 \
  --api-key $OPENAI_API_KEY

3. Evaluate predictions
python scripts/evaluate_predictions.py \
  --dataset data/v2/chunking_v2_test.jsonl \
  --predictions predictions_chunking_v2.jsonl \
  --output summary_chunking_v2.json

V1 multi-model workflow
Run multi-model v1 benchmark sweep
python scripts/multi_model_orchestrator.py \
  --datasets \
    data/v1/constraint_stacking_test.jsonl \
    data/v1/extraneous_load_test.jsonl \
    data/v1/chunking_test.jsonl \
    data/v1/element_interactivity_test.jsonl \
  --models route-llm gpt-4o-mini \
  --base-url https://routellm.abacus.ai/v1 \
  --api-key $OPENAI_API_KEY \
  --output-dir runs/v1

Plot v1 comparison results
python scripts/comparison_plotter.py \
  --input runs/v1/comparison.json \
  --output-dir plots/v1

V2 paired workflow
Run paired multi-model benchmark sweep
python scripts/paired_orchestrate_v2.py \
  --datasets \
    data/v2/chunking_v2_test.jsonl \
    data/v2/extraneous_load_v2_test.jsonl \
    data/v2/element_interactivity_v2_test.jsonl \
  --models route-llm gpt-4o-mini gpt-4.1-mini \
  --base-url https://routellm.abacus.ai/v1 \
  --api-key $OPENAI_API_KEY \
  --output-dir paired_v2_runs

Plot paired comparison results
python scripts/paired_comparison_plotter.py \
  --input paired_v2_runs/paired_comparison.json \
  --output-dir plots/v2

Cross-model significance testing
Bootstrap-based cross-model comparison
python scripts/cross_model_significance.py \
  --inputs \
    paired_v2_runs/gpt_4o_mini__chunking_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__chunking_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__chunking_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4o_mini__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4o_mini__extraneous_load_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__extraneous_load_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__extraneous_load_v2__paired_results.jsonl \
  --json-out cross_model_significance.json \
  --md-out cross_model_significance.md

Permutation-based cross-model comparison
python scripts/cross_model_permutation.py \
  --inputs \
    paired_v2_runs/gpt_4o_mini__chunking_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__chunking_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__chunking_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4o_mini__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4o_mini__extraneous_load_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__extraneous_load_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__extraneous_load_v2__paired_results.jsonl \
  --json-out cross_model_permutation.json \
  --md-out cross_model_permutation.md \
  --n-perm 20000 \
  --seed 42

Multiple-comparison corrected permutation analysis
python scripts/cross_model_permutation_corrected.py \
  --inputs \
    paired_v2_runs/gpt_4o_mini__chunking_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__chunking_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__chunking_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4o_mini__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__element_interactivity_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4o_mini__extraneous_load_v2__paired_results.jsonl \
    paired_v2_runs/gpt_4_1_mini__extraneous_load_v2__paired_results.jsonl \
    paired_v2_runs/route_llm__extraneous_load_v2__paired_results.jsonl \
  --json-out cross_model_permutation_corrected.json \
  --md-out cross_model_permutation_corrected.md \
  --n-perm 20000 \
  --alpha 0.05

Corrected forest plots
BH-FDR
python scripts/forest_plot_corrected.py \
  --input cross_model_permutation_corrected.json \
  --output-dir forest_plots_corrected \
  --correction bh

Holm
python scripts/forest_plot_corrected.py \
  --input cross_model_permutation_corrected.json \
  --output-dir forest_plots_corrected \
  --correction holm

Bonferroni
python scripts/forest_plot_corrected.py \
  --input cross_model_permutation_corrected.json \
  --output-dir forest_plots_corrected \
  --correction bonferroni

Interpretation

Evidence supporting overload-like behavior includes:

performance drops as dependency load increases
structure improves performance on matched tasks
confusable or contradictory distractors are more damaging than irrelevant filler
high interactivity is harder than low interactivity
weaker models show larger penalties or larger structure gains
cross-model differences survive correction
Limitations

This project does not claim that LLMs possess human cognition.

It tests whether they exhibit:

bounded-capacity processing signatures
overload-like degradation curves
structured interference effects

Other limitations:

synthetic tasks may not capture all real workloads
exact-match grading can undercount partial reasoning
API models may change over time
multiple-testing family definition affects corrected results
Authorship and AI use

This repository was built through an AI-assisted development workflow.

The project direction, benchmark framing, hypothesis design, and integration decisions were directed by the repository author.
Much of the code, analysis scaffolding, and documentation were generated and refined with the help of AI systems.

See AGENTS.md for a full description of the AI-assisted workflow.

Future additions

Planned or recommended extensions:

paired constraint_stacking_v2
exact small-(n) permutation enumeration
within-family correction modes
paper-style results memo generator
publication-ready multi-panel figure compositor
local model runner support
richer free-form graders
Philosophy

This repository treats “cognitive load in LLMs” as a measurement and systems-design hypothesis.

The question is not:

“Are LLMs human?”

The question is:

“Do they show reproducible bounded-capacity failure patterns that cognitive load theory helps explain?”

