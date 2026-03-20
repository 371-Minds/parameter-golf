Comparison plotter for comparison.json

Excellent. This gives you the “see the theory” layer.

This script will:

load the orchestrator’s comparison.json
plot macro leaderboard
plot per-benchmark model comparison
plot load curves per model from by_condition
support the four benchmark families:
constraint stacking
extraneous load
chunking
element interactivity

Because the orchestrator stores by_condition, we can reconstruct the curves for each model.

Python script
import json
import argparse
from collections import defaultdict

import matplotlib.pyplot as plt


# ============================================================
# IO
# ============================================================

def load_json(path):
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


# ============================================================
# Helpers
# ============================================================

def parse_condition_key(cond_key: str):
    out = {}
    if not cond_key:
        return out
    for part in cond_key.split("|"):
        if "=" in part:
            k, v = part.split("=", 1)
            out[k] = v
    return out

def try_num(x):
    try:
        if "." in str(x):
            return float(x)
        return int(x)
    except Exception:
        return x

def leaderboard_rows(summary):
    return summary.get("leaderboard", [])

def collect_benchmark_names(summary):
    names = set()
    for row in leaderboard_rows(summary):
        for bench_name in row.get("benchmarks", {}).keys():
            names.add(bench_name)
    return sorted(names)

def model_to_benchmark(summary):
    out = {}
    for row in leaderboard_rows(summary):
        model = row["model"]
        out[model] = row.get("benchmarks", {})
    return out

def infer_benchmark_family(bench_name: str):
    name = bench_name.lower()
    if "constraint" in name:
        return "constraint_stacking"
    if "extraneous" in name:
        return "extraneous_load"
    if "chunk" in name:
        return "chunking"
    if "interactivity" in name:
        return "element_interactivity"
    return "unknown"


# ============================================================
# Plot 1: Macro leaderboard
# ============================================================

def plot_macro_leaderboard(summary):
    rows = leaderboard_rows(summary)
    if not rows:
        print("No leaderboard rows found.")
        return

    models = [r["model"] for r in rows]
    macro_exact = [r.get("macro_exact_accuracy", 0.0) for r in rows]
    macro_css = [
        r["macro_constraint_satisfaction"] if r.get("macro_constraint_satisfaction") is not None else 0.0
        for r in rows
    ]

    fig, axes = plt.subplots(1, 2, figsize=(12, 5))

    # Macro exact
    ax = axes[0]
    bars = ax.bar(models, macro_exact)
    ax.set_title("Macro Exact Accuracy by Model")
    ax.set_ylabel("Macro Exact Accuracy")
    ax.set_ylim(0, 1.05)
    ax.grid(True, axis="y", alpha=0.3)
    ax.tick_params(axis="x", rotation=25)
    for b, v in zip(bars, macro_exact):
        ax.text(b.get_x() + b.get_width() / 2, v + 0.02, f"{v:.2f}", ha="center")

    # Macro css
    ax = axes[1]
    bars = ax.bar(models, macro_css)
    ax.set_title("Macro Constraint Satisfaction by Model")
    ax.set_ylabel("Macro Constraint Satisfaction")
    ax.set_ylim(0, 1.05)
    ax.grid(True, axis="y", alpha=0.3)
    ax.tick_params(axis="x", rotation=25)
    for b, v in zip(bars, macro_css):
        ax.text(b.get_x() + b.get_width() / 2, v + 0.02, f"{v:.2f}", ha="center")

    plt.tight_layout()
    plt.show()


# ============================================================
# Plot 2: Per-benchmark bar comparison
# ============================================================

def plot_per_benchmark_bars(summary):
    rows = leaderboard_rows(summary)
    bench_names = collect_benchmark_names(summary)

    if not rows or not bench_names:
        print("Missing leaderboard or benchmark data.")
        return

    models = [r["model"] for r in rows]
    n_bench = len(bench_names)

    fig, axes = plt.subplots(n_bench, 1, figsize=(10, 4 * n_bench))
    if n_bench == 1:
        axes = [axes]

    for ax, bench_name in zip(axes, bench_names):
        vals = []
        for row in rows:
            bench = row.get("benchmarks", {}).get(bench_name, {})
            vals.append(bench.get("exact_accuracy", 0.0))

        bars = ax.bar(models, vals)
        ax.set_title(f"{bench_name} - Exact Accuracy")
        ax.set_ylabel("Exact Accuracy")
        ax.set_ylim(0, 1.05)
        ax.grid(True, axis="y", alpha=0.3)
        ax.tick_params(axis="x", rotation=25)

        for b, v in zip(bars, vals):
            ax.text(b.get_x() + b.get_width() / 2, v + 0.02, f"{v:.2f}", ha="center")

    plt.tight_layout()
    plt.show()


# ============================================================
# Plot 3: Load curves per benchmark family
# ============================================================

def aggregate_curve_points(by_condition, x_field, y_field="exact_accuracy", group_field=None):
    grouped = defaultdict(list)

    for cond_key, stats in by_condition.items():
        cond = parse_condition_key(cond_key)
        x = cond.get(x_field)
        if x is None:
            continue
        x = try_num(x)

        group = cond.get(group_field, "all") if group_field else "all"
        y = stats.get(y_field, 0.0)
        grouped[group].append((x, y))

    for g in grouped:
        grouped[g] = sorted(grouped[g], key=lambda t: t[0])

    return grouped

def plot_constraint_family(summary, bench_name):
    rows = leaderboard_rows(summary)

    plt.figure(figsize=(9, 5))
    for row in rows:
        model = row["model"]
        bench = row.get("benchmarks", {}).get(bench_name, {})
        by_condition = bench.get("by_condition", {})
        grouped = aggregate_curve_points(by_condition, x_field="num_constraints", y_field="exact_accuracy")

        vals = grouped.get("all", [])
        if not vals:
            continue
        xs = [x for x, _ in vals]
        ys = [y for _, y in vals]
        plt.plot(xs, ys, marker="o", label=model)

    plt.title(f"{bench_name} - Accuracy vs Number of Constraints")
    plt.xlabel("Number of Constraints")
    plt.ylabel("Exact Accuracy")
    plt.ylim(0, 1.05)
    plt.grid(True, alpha=0.3)
    plt.legend()
    plt.tight_layout()
    plt.show()

def plot_extraneous_family(summary, bench_name):
    rows = leaderboard_rows(summary)

    fig, axes = plt.subplots(1, len(rows), figsize=(5 * len(rows), 5), sharey=True)
    if len(rows) == 1:
        axes = [axes]

    for ax, row in zip(axes, rows):
        model = row["model"]
        bench = row.get("benchmarks", {}).get(bench_name, {})
        by_condition = bench.get("by_condition", {})
        grouped = aggregate_curve_points(
            by_condition,
            x_field="distractor_count",
            y_field="exact_accuracy",
            group_field="distractor_type",
        )

        for distractor_type, vals in grouped.items():
            xs = [x for x, _ in vals]
            ys = [y for _, y in vals]
            ax.plot(xs, ys, marker="o", label=distractor_type)

        ax.set_title(model)
        ax.set_xlabel("Distractor Count")
        ax.set_ylim(0, 1.05)
        ax.grid(True, alpha=0.3)

    axes[0].set_ylabel("Exact Accuracy")
    handles, labels = axes[0].get_legend_handles_labels()
    if handles:
        fig.legend(handles, labels, loc="upper center", ncol=3)

    fig.suptitle(f"{bench_name} - Distractor Sensitivity", y=1.03)
    plt.tight_layout()
    plt.show()

def plot_chunking_family(summary, bench_name):
    rows = leaderboard_rows(summary)

    formats = []
    values_by_model = defaultdict(dict)

    for row in rows:
        model = row["model"]
        bench = row.get("benchmarks", {}).get(bench_name, {})
        by_condition = bench.get("by_condition", {})

        grouped_values = defaultdict(list)
        for cond_key, stats in by_condition.items():
            cond = parse_condition_key(cond_key)
            fmt = cond.get("format", "unknown")
            grouped_values[fmt].append(stats.get("exact_accuracy", 0.0))

        for fmt, vals in grouped_values.items():
            values_by_model[model][fmt] = sum(vals) / len(vals)
            formats.append(fmt)

    formats = sorted(set(formats))
    models = [r["model"] for r in rows]

    x = range(len(models))
    width = 0.35 if len(formats) <= 2 else 0.8 / max(1, len(formats))

    plt.figure(figsize=(10, 5))
    for i, fmt in enumerate(formats):
        offsets = [xi + (i - (len(formats)-1)/2) * width for xi in x]
        vals = [values_by_model[m].get(fmt, 0.0) for m in models]
        plt.bar(offsets, vals, width=width, label=fmt)

    plt.xticks(list(x), models, rotation=25)
    plt.ylabel("Exact Accuracy")
    plt.ylim(0, 1.05)
    plt.title(f"{bench_name} - Chunked vs Flat")
    plt.grid(True, axis="y", alpha=0.3)
    plt.legend()
    plt.tight_layout()
    plt.show()

def plot_interactivity_family(summary, bench_name):
    rows = leaderboard_rows(summary)

    plt.figure(figsize=(9, 5))
    for row in rows:
        model = row["model"]
        bench = row.get("benchmarks", {}).get(bench_name, {})
        by_condition = bench.get("by_condition", {})

        grouped = aggregate_curve_points(
            by_condition,
            x_field="num_facts",
            y_field="exact_accuracy",
            group_field="interactivity",
        )

        for interactivity, vals in grouped.items():
            xs = [x for x, _ in vals]
            ys = [y for _, y in vals]
            linestyle = "-" if interactivity == "low" else "--"
            plt.plot(xs, ys, marker="o", linestyle=linestyle, label=f"{model} | {interactivity}")

    plt.title(f"{bench_name} - Accuracy vs Number of Facts")
    plt.xlabel("Number of Facts")
    plt.ylabel("Exact Accuracy")
    plt.ylim(0, 1.05)
    plt.grid(True, alpha=0.3)
    plt.legend()
    plt.tight_layout()
    plt.show()

def plot_family_curves(summary):
    bench_names = collect_benchmark_names(summary)

    for bench_name in bench_names:
        family = infer_benchmark_family(bench_name)
        if family == "constraint_stacking":
            plot_constraint_family(summary, bench_name)
        elif family == "extraneous_load":
            plot_extraneous_family(summary, bench_name)
        elif family == "chunking":
            plot_chunking_family(summary, bench_name)
        elif family == "element_interactivity":
            plot_interactivity_family(summary, bench_name)
        else:
            print(f"Skipping unsupported benchmark family for: {bench_name}")


# ============================================================
# Main
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True, help="comparison.json from orchestrator")
    parser.add_argument("--macro", action="store_true", help="Plot macro leaderboard")
    parser.add_argument("--bars", action="store_true", help="Plot per-benchmark comparison bars")
    parser.add_argument("--curves", action="store_true", help="Plot load curves by benchmark family")
    args = parser.parse_args()

    summary = load_json(args.input)

    if not (args.macro or args.bars or args.curves):
        # default: show everything
        args.macro = True
        args.bars = True
        args.curves = True

    if args.macro:
        plot_macro_leaderboard(summary)

    if args.bars:
        plot_per_benchmark_bars(summary)

    if args.curves:
        plot_family_curves(summary)


if __name__ == "__main__":
    main()

Example usage

Plot everything:

python plot_comparison.py --input comparison.json


Only macro leaderboard:

python plot_comparison.py --input comparison.json --macro


Only curves:

python plot_comparison.py --input comparison.json --curves

What this gives you conceptually
1. Macro leaderboard

This shows:

which model is strongest overall
whether overall strength correlates with overload robustness
2. Per-benchmark bars

This shows:

whether a model is uniformly good
or specifically weak on one load dimension

Example:

good on chunking
weak on interactivity
catastrophic on constraint stacking

That would be theoretically interesting.

3. Family-specific load curves

This is the real payoff.

You can directly inspect whether:

accuracy decays smoothly
accuracy shows threshold collapse
chunking shifts the curve upward
distractor type changes slope
interactivity widens the gap as fact count increases

That’s the signature you’re actually looking for.

What patterns would support your thesis most strongly

The cleanest evidence would be something like:

smaller / weaker models show earlier, steeper collapse
chunking improves performance at the same raw information level
confusable distractors damage performance much more than irrelevant ones
high interactivity hurts more than simply adding independent facts
models preserve partial local competence while losing global exactness

That combination is very hard to explain as “just randomness.”

Best next upgrade from here

Now that you have:

generator
evaluator
runner
orchestrator
comparison plotter

the next best step is one of these:

Option 1: Statistical testing layer

Add:

paired bootstrap CIs
McNemar tests
slope comparisons
effect sizes

This turns the visual story into a publishable one.

Option 2: Paired benchmark v2

Rebuild datasets so each latent example appears in:

flat vs chunked
low vs high distraction
low vs high interactivity

This gives much stronger causal inference.

Option 3: Research memo / paper skeleton

Turn all of this into:

hypothesis
benchmark design
predictions
evaluation
expected signatures
implications for tiny model architecture

If you want the most strategically useful next artifact, I’d recommend Option 3 now:
a concise research memo that frames this as a serious scientific claim rather than just a tooling exercise.
