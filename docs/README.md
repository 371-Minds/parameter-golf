# Cognitive Load Benchmarks for LLMs

A benchmark suite and analysis pipeline for testing whether language models exhibit **bounded-capacity failure patterns analogous to cognitive overload**.

This repository studies whether LLM behavior changes systematically when we manipulate:

- **structure vs. flat presentation**
- **intrinsic relational complexity**
- **extraneous distractor interference**
- **constraint density**

The core claim is not that LLMs “feel” cognitive load in a human sense, but that they may behave like **capacity-limited information processors** whose performance degrades in structured, measurable ways when task demands exceed effective representational bandwidth.

## Core idea

Cognitive Load Theory separates load into:

- **Intrinsic load**: complexity inherent to the task
- **Extraneous load**: irrelevant or badly presented information that interferes
- **Germane load**: useful organization that helps schema formation

This repository operationalizes that frame for LLM evaluation.

We test whether models show signatures like:

- better performance when information is **chunked**
- worse performance when relational coupling is **high**
- disproportionate degradation from **confusable** or **contradictory** distractors
- threshold-like collapse under increased **constraint stacking**

## Repository goals

This project is built to support three use cases:

1. **Scientific testing**
   - Do small or weak models exhibit overload-like failure modes?

2. **Model comparison**
   - Which models are more robust to poor structure, distractors, or coupling?

3. **Design insight**
   - What prompt/model/task properties reduce avoidable processing burden?

## Benchmark families

### V1 benchmark families

These are non-paired synthetic benchmarks with condition-level comparisons.

#### 1. Constraint Stacking
Tests whether accuracy degrades as the number of simultaneously active constraints increases.

Expected pattern:
- performance falls as constraint count rises
- small models collapse faster
- failures become structured, not random

#### 2. Extraneous Load
Tests whether extra information harms performance differently depending on distractor type.

Conditions may include:
- irrelevant filler
- confusable distractors
- contradictory distractors

Expected pattern:
- confusable or contradictory distractors hurt more than harmless filler

#### 3. Chunking
Tests whether grouping information into digestible structure improves performance compared with flat presentation.

Expected pattern:
- chunked formatting improves outcomes
- smaller models benefit more

#### 4. Element Interactivity
Tests whether tasks with stronger dependency coupling are harder than tasks with similar surface length but weaker coupling.

Expected pattern:
- high interactivity produces sharper degradation than low interactivity

## V2 paired benchmark families

These are the stronger causal-design benchmarks.

Instead of comparing broad condition averages across unrelated samples, V2 creates **matched item sets** where the same underlying task is presented in multiple variants.

This allows **within-item paired effect estimation**.

### 1. `chunking_v2`
Same underlying task:
- `flat`
- `chunked`

Primary paired effect:

$$
\Delta_{\text{chunk}} = \mathbb{E}[\text{chunked correct} - \text{flat correct}]
$$

Interpretation:
- positive values mean structure helps

### 2. `element_interactivity_v2`
Same underlying task:
- `low`
- `high`

Primary paired effect:

$$
\Delta_{\text{interactivity}} = \mathbb{E}[\text{low correct} - \text{high correct}]
$$

Interpretation:
- positive values mean high relational coupling hurts performance

### 3. `extraneous_load_v2`
Same underlying task with different distractor regimes:
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

Interpretation:
- positive values mean the second condition is more harmful

## Repository structure

Example structure:

```text
.
├── data/
│   ├── constraint_stacking_train.jsonl
│   ├── constraint_stacking_val.jsonl
│   ├── constraint_stacking_test.jsonl
│   ├── extraneous_load_train.jsonl
│   ├── chunking_train.jsonl
│   ├── element_interactivity_train.jsonl
│   ├── chunking_v2_test.jsonl
│   ├── extraneous_load_v2_test.jsonl
│   └── element_interactivity_v2_test.jsonl
├── scripts/
│   ├── generate_benchmarks.py
│   ├── evaluate_predictions.py
│   ├── plot_results.py
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
├── paired_v2_runs/
├── plots/
├── paired_plots/
└── README.md
