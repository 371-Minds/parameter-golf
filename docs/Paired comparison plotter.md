Paired comparison plotter

Yes — this is the natural next step.

This script reads paired_comparison.json from the v2 orchestrator and generates clean comparison figures for:

chunking gain by model
interactivity penalty by model
extraneous load gaps by model
optional accuracy bars for each paired benchmark

It is designed to turn your paired statistics into presentation-ready visuals quickly.

What it expects

Input file from the orchestrator:

paired_comparison.json

Expected structure:

models
datasets
results[model][dataset] = analysis
What it plots
1. Chunking gain

For each model:

Δchunk​=E[chunked correct−flat correct]

with bootstrap CI error bars.

2. Interactivity penalty

For each model:

Δinteractivity​=E[low correct−high correct]

with bootstrap CI.

3. Extraneous load sensitivity

For each model:

irrelevant - confusable
irrelevant - contradictory_adjacent
confusable - contradictory_adjacent

with bootstrap CI.

4. Accuracy profile bars

Per model:

chunking: flat vs chunked
interactivity: low vs high
extraneous: irrelevant vs confusable vs contradictory-adjacent
Python script
import json
import argparse
import os
import numpy as np
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

def ensure_dir(path):
    os.makedirs(path, exist_ok=True)

def fmt(x):
    if x is None:
        return None
    return float(x)

def ci_to_err(est, ci_low, ci_high):
    if est is None or ci_low is None or ci_high is None:
        return None, None
    lower = est - ci_low
    upper = ci_high - est
    return lower, upper

def get_result(comparison, model, dataset):
    return comparison.get("results", {}).get(model, {}).get(dataset, None)

def available_models(comparison):
    return comparison.get("models", [])

def available_datasets(comparison):
    return comparison.get("datasets", [])


# ============================================================
# Plotting utilities
# ============================================================

def savefig(path):
    plt.tight_layout()
    plt.savefig(path, dpi=180, bbox_inches="tight")
    plt.close()

def barplot_with_ci(labels, estimates, ci_lows, ci_highs, title, ylabel, out_path, color="#4C78A8"):
    x = np.arange(len(labels))
    y = np.array([0.0 if v is None else v for v in estimates], dtype=float)

    lower_err = []
    upper_err = []
    mask = []

    for est, lo, hi in zip(estimates, ci_lows, ci_highs):
        if est is None or lo is None or hi is None:
            lower_err.append(0.0)
            upper_err.append(0.0)
            mask.append(False)
        else:
            le, ue = ci_to_err(est, lo, hi)
            lower_err.append(le)
            upper_err.append(ue)
            mask.append(True)

    plt.figure(figsize=(max(8, len(labels) * 1.1), 5))
    bars = plt.bar(x, y, color=color, alpha=0.85)

    # error bars
    yerr = np.array([lower_err, upper_err])
    plt.errorbar(x, y, yerr=yerr, fmt="none", ecolor="black", capsize=4, lw=1.2)

    plt.xticks(x, labels, rotation=25, ha="right")
    plt.axhline(0, color="gray", linewidth=1)
    plt.title(title)
    plt.ylabel(ylabel)

    for rect, val in zip(bars, estimates):
        if val is not None:
            plt.text(
                rect.get_x() + rect.get_width() / 2,
                rect.get_height(),
                f"{val:.3f}",
                ha="center",
                va="bottom" if val >= 0 else "top",
                fontsize=9,
            )

    savefig(out_path)

def grouped_barplot(series_names, labels, values_2d, title, ylabel, out_path, colors=None):
    n_series = len(series_names)
    n_labels = len(labels)

    x = np.arange(n_labels)
    width = 0.8 / max(n_series, 1)

    if colors is None:
        colors = ["#4C78A8", "#F58518", "#54A24B", "#E45756", "#72B7B2"]

    plt.figure(figsize=(max(8, n_labels * 1.2), 5))

    for i, series_name in enumerate(series_names):
        vals = values_2d[i]
        vals = [0.0 if v is None else v for v in vals]
        plt.bar(
            x + (i - (n_series - 1) / 2) * width,
            vals,
            width=width,
            label=series_name,
            color=colors[i % len(colors)],
            alpha=0.85,
        )

    plt.xticks(x, labels, rotation=25, ha="right")
    plt.title(title)
    plt.ylabel(ylabel)
    plt.legend()
    savefig(out_path)


# ============================================================
# Extractors
# ============================================================

def extract_chunking_effects(comparison):
    models = available_models(comparison)
    labels, ests, lows, highs = [], [], [], []

    for model in models:
        r = get_result(comparison, model, "chunking_v2")
        if not r:
            continue
        ci = r.get("paired_gain_ci", {})
        labels.append(model)
        ests.append(fmt(r.get("paired_gain")))
        lows.append(fmt(ci.get("ci_low")))
        highs.append(fmt(ci.get("ci_high")))

    return labels, ests, lows, highs

def extract_interactivity_effects(comparison):
    models = available_models(comparison)
    labels, ests, lows, highs = [], [], [], []

    for model in models:
        r = get_result(comparison, model, "element_interactivity_v2")
        if not r:
            continue
        ci = r.get("paired_penalty_ci", {})
        labels.append(model)
        ests.append(fmt(r.get("paired_penalty")))
        lows.append(fmt(ci.get("ci_low")))
        highs.append(fmt(ci.get("ci_high")))

    return labels, ests, lows, highs

def extract_extraneous_effects(comparison):
    models = available_models(comparison)

    labels = []
    gap1, gap1_lo, gap1_hi = [], [], []
    gap2, gap2_lo, gap2_hi = [], [], []
    gap3, gap3_lo, gap3_hi = [], [], []

    for model in models:
        r = get_result(comparison, model, "extraneous_load_v2")
        if not r:
            continue

        ci1 = r.get("gap_irrelevant_minus_confusable_ci", {})
        ci2 = r.get("gap_irrelevant_minus_contradictory_adjacent_ci", {})
        ci3 = r.get("gap_confusable_minus_contradictory_adjacent_ci", {})

        labels.append(model)

        gap1.append(fmt(r.get("gap_irrelevant_minus_confusable")))
        gap1_lo.append(fmt(ci1.get("ci_low")))
        gap1_hi.append(fmt(ci1.get("ci_high")))

        gap2.append(fmt(r.get("gap_irrelevant_minus_contradictory_adjacent")))
        gap2_lo.append(fmt(ci2.get("ci_low")))
        gap2_hi.append(fmt(ci2.get("ci_high")))

        gap3.append(fmt(r.get("gap_confusable_minus_contradictory_adjacent")))
        gap3_lo.append(fmt(ci3.get("ci_low")))
        gap3_hi.append(fmt(ci3.get("ci_high")))

    return {
        "labels": labels,
        "irrelevant_minus_confusable": (gap1, gap1_lo, gap1_hi),
        "irrelevant_minus_contradictory_adjacent": (gap2, gap2_lo, gap2_hi),
        "confusable_minus_contradictory_adjacent": (gap3, gap3_lo, gap3_hi),
    }

def extract_accuracy_profiles(comparison):
    models = available_models(comparison)

    chunk_flat = []
    chunk_chunked = []

    inter_low = []
    inter_high = []

    extr_ir = []
    extr_co = []
    extr_ca = []

    labels = []

    for model in models:
        used = False

        rc = get_result(comparison, model, "chunking_v2")
        ri = get_result(comparison, model, "element_interactivity_v2")
        re = get_result(comparison, model, "extraneous_load_v2")

        if rc or ri or re:
            labels.append(model)
            used = True

        if used:
            chunk_flat.append(fmt(rc.get("flat_accuracy")) if rc else None)
            chunk_chunked.append(fmt(rc.get("chunked_accuracy")) if rc else None)

            inter_low.append(fmt(ri.get("low_accuracy")) if ri else None)
            inter_high.append(fmt(ri.get("high_accuracy")) if ri else None)

            extr_ir.append(fmt(re.get("irrelevant_accuracy")) if re else None)
            extr_co.append(fmt(re.get("confusable_accuracy")) if re else None)
            extr_ca.append(fmt(re.get("contradictory_adjacent_accuracy")) if re else None)

    return {
        "labels": labels,
        "chunking": [chunk_flat, chunk_chunked],
        "interactivity": [inter_low, inter_high],
        "extraneous": [extr_ir, extr_co, extr_ca],
    }


# ============================================================
# Main plotting
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True, help="paired_comparison.json")
    parser.add_argument("--output-dir", default="paired_plots", help="Directory for plots")
    args = parser.parse_args()

    comparison = load_json(args.input)
    ensure_dir(args.output_dir)

    # 1. Chunking gain
    labels, ests, lows, highs = extract_chunking_effects(comparison)
    if labels:
        barplot_with_ci(
            labels=labels,
            estimates=ests,
            ci_lows=lows,
            ci_highs=highs,
            title="Chunking Gain by Model",
            ylabel="Paired gain (chunked - flat)",
            out_path=os.path.join(args.output_dir, "chunking_gain_by_model.png"),
            color="#4C78A8",
        )

    # 2. Interactivity penalty
    labels, ests, lows, highs = extract_interactivity_effects(comparison)
    if labels:
        barplot_with_ci(
            labels=labels,
            estimates=ests,
            ci_lows=lows,
            ci_highs=highs,
            title="Interactivity Penalty by Model",
            ylabel="Paired penalty (low - high)",
            out_path=os.path.join(args.output_dir, "interactivity_penalty_by_model.png"),
            color="#E45756",
        )

    # 3. Extraneous sensitivity
    extr = extract_extraneous_effects(comparison)
    labels = extr["labels"]

    if labels:
        g1, g1_lo, g1_hi = extr["irrelevant_minus_confusable"]
        g2, g2_lo, g2_hi = extr["irrelevant_minus_contradictory_adjacent"]
        g3, g3_lo, g3_hi = extr["confusable_minus_contradictory_adjacent"]

        barplot_with_ci(
            labels=labels,
            estimates=g1,
            ci_lows=g1_lo,
            ci_highs=g1_hi,
            title="Extraneous Load Gap: Irrelevant - Confusable",
            ylabel="Paired gap",
            out_path=os.path.join(args.output_dir, "extraneous_gap_irrelevant_minus_confusable.png"),
            color="#54A24B",
        )

        barplot_with_ci(
            labels=labels,
            estimates=g2,
            ci_lows=g2_lo,
            ci_highs=g2_hi,
            title="Extraneous Load Gap: Irrelevant - Contradictory Adjacent",
            ylabel="Paired gap",
            out_path=os.path.join(args.output_dir, "extraneous_gap_irrelevant_minus_contradictory_adjacent.png"),
            color="#F58518",
        )

        barplot_with_ci(
            labels=labels,
            estimates=g3,
            ci_lows=g3_lo,
            ci_highs=g3_hi,
            title="Extraneous Load Gap: Confusable - Contradictory Adjacent",
            ylabel="Paired gap",
            out_path=os.path.join(args.output_dir, "extraneous_gap_confusable_minus_contradictory_adjacent.png"),
            color="#72B7B2",
        )

    # 4. Accuracy profile bars
    prof = extract_accuracy_profiles(comparison)

    if prof["labels"]:
        grouped_barplot(
            series_names=["flat", "chunked"],
            labels=prof["labels"],
            values_2d=prof["chunking"],
            title="Chunking Accuracy Profile",
            ylabel="Accuracy",
            out_path=os.path.join(args.output_dir, "chunking_accuracy_profile.png"),
            colors=["#9ECAE9", "#3182BD"],
        )

        grouped_barplot(
            series_names=["low", "high"],
            labels=prof["labels"],
            values_2d=prof["interactivity"],
            title="Interactivity Accuracy Profile",
            ylabel="Accuracy",
            out_path=os.path.join(args.output_dir, "interactivity_accuracy_profile.png"),
            colors=["#FB6A4A", "#A50F15"],
        )

        grouped_barplot(
            series_names=["irrelevant", "confusable", "contradictory_adjacent"],
            labels=prof["labels"],
            values_2d=prof["extraneous"],
            title="Extraneous Load Accuracy Profile",
            ylabel="Accuracy",
            out_path=os.path.join(args.output_dir, "extraneous_accuracy_profile.png"),
            colors=["#74C476", "#31A354", "#006D2C"],
        )

    print(f"Saved paired comparison plots to: {args.output_dir}")


if __name__ == "__main__":
    main()

Usage
python paired_comparison_plotter.py \
  --input paired_v2_runs/paired_comparison.json \
  --output-dir paired_plots

Files it creates

Inside paired_plots/:

chunking_gain_by_model.png
interactivity_penalty_by_model.png
extraneous_gap_irrelevant_minus_confusable.png
extraneous_gap_irrelevant_minus_contradictory_adjacent.png
extraneous_gap_confusable_minus_contradictory_adjacent.png
chunking_accuracy_profile.png
interactivity_accuracy_profile.png
extraneous_accuracy_profile.png
How to read the figures
Chunking gain

Higher positive bars mean the model benefits more from structure.

Interpretation:

small/tiny models should often show larger gains
stronger models may still benefit, but less dramatically
Interactivity penalty

Higher positive bars mean the model is more harmed by relational coupling.

Interpretation:

if high interactivity hurts consistently, that supports your load thesis
if weaker models show larger penalties, that supports bounded-capacity behavior
Extraneous gaps

Positive values mean the first condition is easier than the second.

Examples:

irrelevant - confusable > 0: confusable distractors are worse than harmless filler
irrelevant - contradictory_adjacent > 0: contradiction near the target causes stronger interference
confusable - contradictory_adjacent > 0: contradiction is even worse than near-miss confusion
Accuracy profiles

These are useful because effect sizes alone can hide the baseline.

Example:

two models might have the same chunking gain
but one could be strong overall while the other is near collapse

So profile charts give context.

Best next upgrade

The strongest next addition would be one of these:

publication-style figure compositor

combine several plots into one paper-ready panel figure

cross-model significance script

compare whether one model’s chunking gain is significantly larger than another’s

paired benchmark v3

add constraint stacking in matched form too

If you want the most useful next artifact for a paper or memo, I’d recommend:

publication-style figure compositor.

If you want the most useful next artifact for scientific rigor, I’d recommend:

cross-model significance script.

If you want, I can write either one next.
