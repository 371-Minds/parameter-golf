Statistical testing script

Below is a practical stats layer for your benchmark results.

It is designed to work from comparison.json and compute lightweight, interpretable statistics for:

constraint overload slope
chunking gain
interactivity penalty
distractor sensitivity
optional bootstrap confidence intervals

This is not a full inferential framework, but it is strong enough for:

research memos
benchmark reports
early paper drafts
deciding whether the effects are real and directionally stable
What this script computes

For each model and benchmark family:

Constraint stacking
regression slope of exact accuracy vs num_constraints
regression slope of mean constraint satisfaction vs num_constraints
Chunking
average chunked - flat exact accuracy
Element interactivity
average low - high exact accuracy penalty
Extraneous load
average exact accuracy by distractor type
pairwise gaps such as:
irrelevant - confusable
irrelevant - contradictory_adjacent
Bootstrap confidence intervals

For any effect computed from multiple condition values:

mean estimate
bootstrap lower / upper CI
Python script
import json
import math
import random
import argparse
from collections import defaultdict


# ============================================================
# IO
# ============================================================

def load_json(path):
    with open(path, "r", encoding="utf-8") as f:
        return json.load(f)


# ============================================================
# Helpers
# ============================================================

def leaderboard_rows(summary):
    return summary.get("leaderboard", [])

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

def infer_family(bench_name: str):
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

def mean(xs):
    xs = [x for x in xs if x is not None]
    return sum(xs) / len(xs) if xs else None

def percentile(xs, p):
    if not xs:
        return None
    xs = sorted(xs)
    k = (len(xs) - 1) * p
    f = math.floor(k)
    c = math.ceil(k)
    if f == c:
        return xs[int(k)]
    d0 = xs[f] * (c - k)
    d1 = xs[c] * (k - f)
    return d0 + d1

def bootstrap_ci(values, statistic_fn=mean, n_boot=2000, alpha=0.05, seed=42):
    vals = [v for v in values if v is not None]
    if not vals:
        return {"estimate": None, "ci_low": None, "ci_high": None, "n": 0}
    rng = random.Random(seed)
    stats = []
    n = len(vals)
    for _ in range(n_boot):
        sample = [vals[rng.randrange(n)] for _ in range(n)]
        stats.append(statistic_fn(sample))
    est = statistic_fn(vals)
    return {
        "estimate": est,
        "ci_low": percentile(stats, alpha / 2),
        "ci_high": percentile(stats, 1 - alpha / 2),
        "n": n,
    }

def linear_slope(xs, ys):
    pairs = [(x, y) for x, y in zip(xs, ys) if x is not None and y is not None]
    if len(pairs) < 2:
        return None
    xs = [p[0] for p in pairs]
    ys = [p[1] for p in pairs]
    xbar = sum(xs) / len(xs)
    ybar = sum(ys) / len(ys)
    num = sum((x - xbar) * (y - ybar) for x, y in zip(xs, ys))
    den = sum((x - xbar) ** 2 for x in xs)
    if den == 0:
        return None
    return num / den

def fmt(x, digits=4):
    if x is None:
        return "n/a"
    return f"{x:.{digits}f}"


# ============================================================
# Effect extraction
# ============================================================

def get_model_benchmarks(summary):
    result = {}
    for row in leaderboard_rows(summary):
        result[row["model"]] = row.get("benchmarks", {})
    return result

def extract_constraint_effects(by_condition):
    grouped_acc = defaultdict(list)
    grouped_css = defaultdict(list)

    for cond_key, stats in by_condition.items():
        cond = parse_condition_key(cond_key)
        if "num_constraints" not in cond:
            continue
        x = try_num(cond["num_constraints"])
        grouped_acc[x].append(stats.get("exact_accuracy"))
        grouped_css[x].append(stats.get("mean_constraint_satisfaction"))

    xs = sorted(grouped_acc.keys())
    ys_acc = [mean(grouped_acc[x]) for x in xs]
    ys_css = [mean(grouped_css[x]) for x in xs]

    return {
        "x_levels": xs,
        "accuracy_by_constraints": dict(zip(xs, ys_acc)),
        "css_by_constraints": dict(zip(xs, ys_css)),
        "accuracy_slope": linear_slope(xs, ys_acc),
        "css_slope": linear_slope(xs, ys_css),
    }

def extract_chunking_effects(by_condition):
    flat = []
    chunked = []

    for cond_key, stats in by_condition.items():
        cond = parse_condition_key(cond_key)
        fmt_type = cond.get("format")
        if fmt_type == "flat":
            flat.append(stats.get("exact_accuracy"))
        elif fmt_type == "chunked":
            chunked.append(stats.get("exact_accuracy"))

    flat_avg = mean(flat)
    chunked_avg = mean(chunked)
    gain = None if flat_avg is None or chunked_avg is None else (chunked_avg - flat_avg)

    return {
        "flat_values": flat,
        "chunked_values": chunked,
        "flat_mean": flat_avg,
        "chunked_mean": chunked_avg,
        "chunking_gain": gain,
        "chunking_gain_ci": bootstrap_ci(
            [(c - f) for c, f in zip(chunked[:min(len(flat), len(chunked))], flat[:min(len(flat), len(chunked))])],
            statistic_fn=mean,
        ) if flat and chunked else {"estimate": None, "ci_low": None, "ci_high": None, "n": 0},
    }

def extract_interactivity_effects(by_condition):
    low = []
    high = []

    for cond_key, stats in by_condition.items():
        cond = parse_condition_key(cond_key)
        inter = cond.get("interactivity")
        if inter == "low":
            low.append(stats.get("exact_accuracy"))
        elif inter == "high":
            high.append(stats.get("exact_accuracy"))

    low_avg = mean(low)
    high_avg = mean(high)
    penalty = None if low_avg is None or high_avg is None else (low_avg - high_avg)

    return {
        "low_values": low,
        "high_values": high,
        "low_mean": low_avg,
        "high_mean": high_avg,
        "interactivity_penalty": penalty,
        "interactivity_penalty_ci": bootstrap_ci(
            [(l - h) for l, h in zip(low[:min(len(low), len(high))], high[:min(len(low), len(high))])],
            statistic_fn=mean,
        ) if low and high else {"estimate": None, "ci_low": None, "ci_high": None, "n": 0},
    }

def extract_extraneous_effects(by_condition):
    grouped = defaultdict(list)

    for cond_key, stats in by_condition.items():
        cond = parse_condition_key(cond_key)
        dt = cond.get("distractor_type")
        if dt is not None:
            grouped[dt].append(stats.get("exact_accuracy"))

    type_means = {k: mean(v) for k, v in grouped.items()}

    def gap(a, b):
        if a not in type_means or b not in type_means:
            return None
        return type_means[a] - type_means[b]

    return {
        "type_means": type_means,
        "gap_irrelevant_vs_confusable": gap("irrelevant", "confusable"),
        "gap_irrelevant_vs_contradictory_adjacent": gap("irrelevant", "contradictory_adjacent"),
        "gap_confusable_vs_contradictory_adjacent": gap("confusable", "contradictory_adjacent"),
    }


# ============================================================
# Main analysis
# ============================================================

def analyze(summary):
    model_benchmarks = get_model_benchmarks(summary)
    report = {}

    for model, benchmarks in model_benchmarks.items():
        model_report = {}

        for bench_name, bench_summary in benchmarks.items():
            family = infer_family(bench_name)
            by_condition = bench_summary.get("by_condition", {})

            if family == "constraint_stacking":
                model_report[bench_name] = {
                    "family": family,
                    **extract_constraint_effects(by_condition),
                }

            elif family == "chunking":
                model_report[bench_name] = {
                    "family": family,
                    **extract_chunking_effects(by_condition),
                }

            elif family == "element_interactivity":
                model_report[bench_name] = {
                    "family": family,
                    **extract_interactivity_effects(by_condition),
                }

            elif family == "extraneous_load":
                model_report[bench_name] = {
                    "family": family,
                    **extract_extraneous_effects(by_condition),
                }

        report[model] = model_report

    return report


# ============================================================
# Markdown formatter
# ============================================================

def to_markdown(analysis):
    lines = ["# Statistical Effects Summary", ""]

    for model, benchmarks in analysis.items():
        lines.append(f"## {model}")
        lines.append("")

        for bench_name, result in benchmarks.items():
            family = result["family"]
            lines.append(f"### {bench_name}")
            lines.append("")

            if family == "constraint_stacking":
                lines.append(f"- Accuracy slope vs constraints: `{fmt(result['accuracy_slope'])}`")
                lines.append(f"- Constraint satisfaction slope vs constraints: `{fmt(result['css_slope'])}`")
                lines.append("")

            elif family == "chunking":
                ci = result["chunking_gain_ci"]
                lines.append(f"- Flat mean accuracy: `{fmt(result['flat_mean'])}`")
                lines.append(f"- Chunked mean accuracy: `{fmt(result['chunked_mean'])}`")
                lines.append(f"- Chunking gain: `{fmt(result['chunking_gain'])}`")
                lines.append(
                    f"- Bootstrap CI: `[{fmt(ci['ci_low'])}, {fmt(ci['ci_high'])}]` "
                    f"(n={ci['n']})"
                )
                lines.append("")

            elif family == "element_interactivity":
                ci = result["interactivity_penalty_ci"]
                lines.append(f"- Low-interactivity mean accuracy: `{fmt(result['low_mean'])}`")
                lines.append(f"- High-interactivity mean accuracy: `{fmt(result['high_mean'])}`")
                lines.append(f"- Interactivity penalty: `{fmt(result['interactivity_penalty'])}`")
                lines.append(
                    f"- Bootstrap CI: `[{fmt(ci['ci_low'])}, {fmt(ci['ci_high'])}]` "
                    f"(n={ci['n']})"
                )
                lines.append("")

            elif family == "extraneous_load":
                type_means = result.get("type_means", {})
                for k in sorted(type_means.keys()):
                    lines.append(f"- Mean accuracy with `{k}` distractors: `{fmt(type_means[k])}`")
                lines.append(f"- Irrelevant - Confusable gap: `{fmt(result['gap_irrelevant_vs_confusable'])}`")
                lines.append(
                    f"- Irrelevant - Contradictory Adjacent gap: "
                    f"`{fmt(result['gap_irrelevant_vs_contradictory_adjacent'])}`"
                )
                lines.append(
                    f"- Confusable - Contradictory Adjacent gap: "
                    f"`{fmt(result['gap_confusable_vs_contradictory_adjacent'])}`"
                )
                lines.append("")

    return "\n".join(lines)


# ============================================================
# Main
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True, help="comparison.json")
    parser.add_argument("--json-out", default=None, help="Optional analysis json output")
    parser.add_argument("--md-out", default=None, help="Optional markdown summary output")
    args = parser.parse_args()

    summary = load_json(args.input)
    analysis = analyze(summary)

    if args.json_out:
        with open(args.json_out, "w", encoding="utf-8") as f:
            json.dump(analysis, f, indent=2, ensure_ascii=False)
        print(f"Saved analysis JSON to {args.json_out}")

    md = to_markdown(analysis)

    if args.md_out:
        with open(args.md_out, "w", encoding="utf-8") as f:
            f.write(md)
        print(f"Saved markdown summary to {args.md_out}")
    else:
        print(md)


if __name__ == "__main__":
    main()

Example usage

Write both machine-readable and markdown summaries:

python stats_effects.py \
  --input comparison.json \
  --json-out effects.json \
  --md-out EFFECTS.md


Or just print markdown:

python stats_effects.py --input comparison.json

How to interpret the outputs
Constraint stacking slope

If the slope is more negative, the model degrades faster as constraints increase.

Example:

Model A: -0.07
Model B: -0.18

Then Model B is more overload-sensitive on that axis.

Chunking gain

If chunked - flat > 0, structure is helping.

Example:

gain = 0.11
CI = [0.07, 0.15]

That’s strong evidence that organization reduces effective burden.

Interactivity penalty

If low - high > 0, coupling hurts.

Example:

penalty = 0.14
CI = [0.09, 0.20]

That supports the claim that interdependence matters beyond raw fact count.

Distractor gaps

If:

irrelevant - confusable > 0
irrelevant - contradictory_adjacent > 0

then interference quality matters more than mere prompt length.

Important limitation of this version

For chunking and interactivity, this script uses simple grouped condition summaries from comparison.json.

That means:

it is not item-paired
CI estimates are based on aggregated condition-level values, not per-example matched pairs

So this is useful but not ideal.

The scientifically stronger next version

The best upgrade would be:

A paired-effects stats script

Using per-item results, you could compute:

paired bootstrap deltas
McNemar tests
exact matched comparisons
condition-controlled effect sizes

That would be substantially stronger than working from aggregate summaries.

Best next step

At this point, the strongest possible move is:

upgrade the benchmark generator to paired-condition v2
upgrade the evaluator to preserve per-example structured outputs
run real models and produce a first empirical note

If you want, I can do 1 next:
a paired-condition benchmark v2 generator that makes the chunking, distraction, and interactivity experiments much more scientifically rigorous.
