Plotting script for overload curves

Here’s a Python script that reads the evaluator summary JSON and plots:

exact accuracy by condition
constraint satisfaction by condition
distractor sensitivity
chunked vs flat
low vs high interactivity

It assumes you’ve already run the evaluator and saved files like:

cs_summary.json
el_summary.json
ch_summary.json
ei_summary.json
Python script
import json
import argparse
import math
from collections import defaultdict

import matplotlib.pyplot as plt

# ============================================================
# Helpers
# ============================================================

def load_json(path):
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)

def parse_condition_key(cond_key: str):
    out = {}
    if not cond_key:
        return out
    parts = cond_key.split("|")
    for p in parts:
        if "=" in p:
            k, v = p.split("=", 1)
            out[k] = v
    return out

def try_num(x):
    try:
        if "." in str(x):
            return float(x)
        return int(x)
    except Exception:
        return x

def aggregate_by(parsed_items, x_field, group_field=None, y_field="exact_accuracy"):
    grouped = defaultdict(list)

    for cond, stats in parsed_items:
        x = try_num(cond.get(x_field))
        if x is None:
            continue
        g = cond.get(group_field) if group_field else "all"
        y = stats.get(y_field, 0.0)
        grouped[g].append((x, y, stats))

    result = {}
    for g, vals in grouped.items():
        vals.sort(key=lambda t: t[0])
        result[g] = vals
    return result

def make_parsed_items(summary):
    return [
        (parse_condition_key(cond_key), stats)
        for cond_key, stats in summary.get("by_condition", {}).items()
    ]

# ============================================================
# Plot functions
# ============================================================

def plot_constraint_stacking(summary, title="Constraint Stacking"):
    parsed = make_parsed_items(summary)

    acc = aggregate_by(parsed, x_field="num_constraints", y_field="exact_accuracy")
    css = aggregate_by(parsed, x_field="num_constraints", y_field="mean_constraint_satisfaction")

    fig, axes = plt.subplots(1, 2, figsize=(12, 5))

    # Exact accuracy
    ax = axes[0]
    for group, vals in acc.items():
        xs = [x for x, _, _ in vals]
        ys = [y for _, y, _ in vals]
        ax.plot(xs, ys, marker="o", label=group)
    ax.set_title(f"{title} - Exact Accuracy")
    ax.set_xlabel("Number of Constraints")
    ax.set_ylabel("Exact Accuracy")
    ax.set_ylim(0, 1.05)
    ax.grid(True, alpha=0.3)

    # Constraint satisfaction
    ax = axes[1]
    for group, vals in css.items():
        xs = [x for x, _, _ in vals]
        ys = [y for _, y, _ in vals]
        ax.plot(xs, ys, marker="o", label=group)
    ax.set_title(f"{title} - Mean Constraint Satisfaction")
    ax.set_xlabel("Number of Constraints")
    ax.set_ylabel("Constraint Satisfaction")
    ax.set_ylim(0, 1.05)
    ax.grid(True, alpha=0.3)

    plt.tight_layout()
    plt.show()

def plot_extraneous_load(summary, title="Extraneous Load"):
    parsed = make_parsed_items(summary)
    grouped = aggregate_by(parsed, x_field="distractor_count", group_field="distractor_type", y_field="exact_accuracy")

    plt.figure(figsize=(8, 5))
    for distractor_type, vals in grouped.items():
        xs = [x for x, _, _ in vals]
        ys = [y for _, y, _ in vals]
        plt.plot(xs, ys, marker="o", label=distractor_type)

    plt.title(f"{title} - Accuracy vs Distractor Count")
    plt.xlabel("Distractor Count")
    plt.ylabel("Exact Accuracy")
    plt.ylim(0, 1.05)
    plt.grid(True, alpha=0.3)
    plt.legend()
    plt.tight_layout()
    plt.show()

def plot_chunking(summary, title="Chunking"):
    parsed = make_parsed_items(summary)

    by_format = defaultdict(list)
    for cond, stats in parsed:
        fmt = cond.get("format", "unknown")
        query_type = cond.get("query_type", "unknown")
        by_format[(fmt, query_type)].append(stats["exact_accuracy"])

    labels = []
    values = []
    for (fmt, query_type), vals in sorted(by_format.items()):
        labels.append(f"{fmt}\n{query_type}")
        values.append(sum(vals) / len(vals))

    plt.figure(figsize=(8, 5))
    bars = plt.bar(labels, values)
    plt.title(f"{title} - Accuracy by Format")
    plt.ylabel("Exact Accuracy")
    plt.ylim(0, 1.05)
    plt.grid(True, axis="y", alpha=0.3)

    for b, v in zip(bars, values):
        plt.text(b.get_x() + b.get_width() / 2, v + 0.02, f"{v:.2f}", ha="center")

    plt.tight_layout()
    plt.show()

def plot_element_interactivity(summary, title="Element Interactivity"):
    parsed = make_parsed_items(summary)
    grouped = aggregate_by(parsed, x_field="num_facts", group_field="interactivity", y_field="exact_accuracy")

    plt.figure(figsize=(8, 5))
    for interactivity, vals in grouped.items():
        xs = [x for x, _, _ in vals]
        ys = [y for _, y, _ in vals]
        plt.plot(xs, ys, marker="o", label=interactivity)

    plt.title(f"{title} - Accuracy vs Number of Facts")
    plt.xlabel("Number of Facts")
    plt.ylabel("Exact Accuracy")
    plt.ylim(0, 1.05)
    plt.grid(True, alpha=0.3)
    plt.legend()
    plt.tight_layout()
    plt.show()

def plot_failure_counts(summary, title="Failure Types"):
    failure_counts = summary.get("failure_counts", {})
    if not failure_counts:
        print("No failure counts available in this summary.")
        return

    items = sorted(failure_counts.items(), key=lambda x: (-x[1], x[0]))
    labels = [k for k, _ in items]
    values = [v for _, v in items]

    plt.figure(figsize=(10, 5))
    bars = plt.bar(labels, values)
    plt.title(title)
    plt.ylabel("Count")
    plt.xticks(rotation=30, ha="right")
    plt.grid(True, axis="y", alpha=0.3)

    for b, v in zip(bars, values):
        plt.text(b.get_x() + b.get_width() / 2, v + max(values) * 0.01, str(v), ha="center")

    plt.tight_layout()
    plt.show()

# ============================================================
# CLI
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--cs", help="Constraint stacking summary JSON")
    parser.add_argument("--el", help="Extraneous load summary JSON")
    parser.add_argument("--ch", help="Chunking summary JSON")
    parser.add_argument("--ei", help="Element interactivity summary JSON")
    parser.add_argument("--failures", action="store_true", help="Plot failure counts for constraint stacking if available")
    args = parser.parse_args()

    if args.cs:
        cs = load_json(args.cs)
        plot_constraint_stacking(cs)
        if args.failures:
            plot_failure_counts(cs, title="Constraint Stacking Failure Types")

    if args.el:
        el = load_json(args.el)
        plot_extraneous_load(el)

    if args.ch:
        ch = load_json(args.ch)
        plot_chunking(ch)

    if args.ei:
        ei = load_json(args.ei)
        plot_element_interactivity(ei)

if __name__ == "__main__":
    main()

Example usage

Plot all four:

python plot_overload.py --cs cs_summary.json --el el_summary.json --ch ch_summary.json --ei ei_summary.json --failures


Or just one:

python plot_overload.py --cs cs_summary.json --failures

What each graph should show
Constraint stacking

You want to see:

exact accuracy dropping as number of constraints increases
constraint satisfaction maybe degrading more gradually
this gap is important because it shows:
local rule handling may survive
global coordination collapses first

That’s a classic overload signature.

Extraneous load

You want:

irrelevant distractors hurt somewhat
confusable distractors hurt more
contradictory_adjacent hurts most, ideally

That supports the idea that load is about interference, not just length.

Chunking

You want:

chunked > flat
especially on lookup/comparison tasks

That supports the idea that organization reduces effective load.

Element interactivity

You want:

high interactivity underperforms low
the gap widens as number of facts increases

That supports the claim that dependency coupling matters more than raw information amount.

Best interpretation pattern

If your plots come out like this:

constraint curves bend sharply downward
confusable distractors create steeper drops than irrelevant ones
chunked consistently beats flat
high-interactivity underperforms low-interactivity at same fact count

then your theory has real teeth.

Because then the system is not merely:

running out of tokens
failing randomly
showing generic noise

It’s showing structured overload behavior.

What I’d add next if you continue

The strongest next upgrades would be:

1. Multi-model overlay plots

Plot curves from:

3M model
15M model
60M model

on the same axes.

That would make the “capacity-sensitive overload” story much stronger.

2. Paired-difference plots

For chunking and distractors:

same latent item
different condition
plot per-item delta

This is much stronger statistically.

3. Regression summary tables

Fit simple models like:

accuracy ~ num_constraints
accuracy ~ distractor_count * distractor_type
accuracy ~ format
accuracy ~ interactivity * num_facts

Then you have both visual and statistical support.

If you want the most useful next thing

I’d suggest one of these:

a model runner script that takes a benchmark JSONL and calls an OpenAI-compatible endpoint
a multi-model comparison plotter
a paper-style methodology/results template

Given your trajectory, I think 1 is the best next move, especially if you want to test this quickly.
I can write a runner that works with OpenAI-compatible APIs, including something like RouteLLM-style endpoints, if you want.
