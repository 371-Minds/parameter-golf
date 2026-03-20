# Tiny LLM Cognitive Load Benchmarks

A benchmark suite for testing whether tiny language models exhibit structured overload effects analogous to cognitive load phenomena in humans.

## Overview

This project investigates the idea that small language models fail not only because they know less, but because they can coordinate fewer active dependencies, are more vulnerable to interference, and benefit more from structured presentation.

We evaluate four dimensions:

- **Constraint Stacking**  
  Can the model satisfy multiple simultaneous rules at once?

- **Extraneous Load**  
  How much does irrelevant, confusable, or contradictory information impair performance?

- **Chunking Benefit**  
  Does equivalent information become easier when grouped or structured?

- **Element Interactivity**  
  Are tightly coupled facts harder than equally numerous independent facts?

The core hypothesis is that tiny models behave like bounded-capacity processors with structured failure modes under load.

## Why this matters

This benchmark is designed for:

- evaluating tiny or compressed language models
- testing hypotheses about capacity limits
- studying interference and prompt organization effects
- comparing model robustness under increasing dependency burden
- informing architecture choices for parameter-constrained systems

This is especially relevant for:
- edge models
- tiny transformers
- distilled systems
- parameter-golf style challenges
- highly compressed reasoning systems

## Benchmark families

### 1. Constraint Stacking

The model must produce an answer satisfying several simultaneous constraints, such as:

- a character at a specific position
- inclusion of a required symbol
- exact digit count
- ordering constraints
- distinctness constraints

This probes coordination under concurrent rule pressure.

### 2. Extraneous Load

The model answers a simple question from a context that also contains distractors. Distractors vary by:

- count
- similarity to target facts
- contradiction level
- placement near relevant content

This probes interference sensitivity.

### 3. Chunking

The same latent information is presented in two forms:

- flat/unstructured
- grouped/chunked

The task stays constant. This probes whether structure reduces effective load.

### 4. Element Interactivity

The model answers questions based on either:

- independent facts
- interdependent fact chains

This probes whether coupling is harder than raw quantity alone.

## Core hypotheses

We test the following predictions:

1. **Constraint overload:** exact accuracy falls sharply as simultaneous constraints increase.
2. **Interference-sensitive distraction:** confusable distractors hurt more than irrelevant ones.
3. **Chunking benefit:** grouped information improves performance without changing content.
4. **Element interactivity penalty:** tightly coupled facts are harder than equally numerous independent facts.
5. **Capacity-scaling effect:** smaller models collapse earlier and more steeply.

## Repository structure

```text
.
├── generate_datasets.py
├── run_benchmark.py
├── evaluate.py
├── plot_overload.py
├── orchestrate_benchmarks.py
├── plot_comparison.py
├── README.md
└── data/
