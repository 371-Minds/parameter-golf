Markdown results compiler from comparison.json

Perfect. This will save you a lot of manual work.

This script:

reads comparison.json
extracts benchmark metrics
generates a markdown report
includes:
overall leaderboard
per-benchmark tables
chunking gain
interactivity penalty
optional lightweight interpretation snippets

You can paste the output into:

README.md
RESULTS.md
a paper draft
an internal memo
Python script
import json
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

def collect_benchmark_names(summary):
    names = set()
    for row in leaderboard_rows(summary):
        names.update(row.get("benchmarks", {}).keys())
    return sorted(names)

def fmt(x, digits=3):
    if x is None:
        return "n/a"
    if isinstance(x, str):
        return x
    return f"{x:.{digits}f}"

def md_table(headers, rows):
    out = []
    out.append("| " + " | ".join(headers) + " |")
    out.append("|" + "|".join(["---"] * len(headers)) + "|")
    for row in rows:
        out.append("| " + " | ".join(str(x) for x in row) + " |")
    return "\n".join(out)

def average(vals):
    vals = [v for v in vals if v is not None]
    return sum(vals) / len(vals) if vals else None


# ============================================================
# Section builders
# ============================================================

def build_overall_leaderboard(summary):
    headers = [
        "Model",
        "Macro Exact",
        "Macro CSS",
    ]
    rows = []
    for row in leaderboard_rows(summary):
        rows.append([
            row["model"],
            fmt(row.get("macro_exact_accuracy")),
            fmt(row.get("macro_constraint_satisfaction")),
        ])
    return "## Overall Leaderboard\n\n" + md_table(headers, rows)

def build_benchmark_overview(summary):
    bench_names = collect_benchmark_names(summary)
    headers = ["Model"] + bench_names
    rows = []

    for row in leaderboard_rows(summary):
        model = row["model"]
        vals = []
        for bench in bench_names:
            bench_summary = row.get("benchmarks", {}).get(bench, {})
            vals.append(fmt(bench_summary.get("exact_accuracy")))
        rows.append([model] + vals)

    return "## Benchmark Exact Accuracy Overview\n\n" + md_table(headers, rows)

def build_constraint_section(summary, bench_name):
    models = leaderboard_rows(summary)

    all_constraint_levels = set()
    for row in models:
        by_condition = row.get("benchmarks", {}).get(bench_name, {}).get("by_condition", {})
        for cond_key in by_condition:
            cond = parse_condition_key(cond_key)
            if "num_constraints" in cond:
                all_constraint_levels.add(try_num(cond["num_constraints"]))

    all_constraint_levels = sorted(all_constraint_levels)
    if not all_constraint_levels:
        return None

    headers = ["Model"] + [f"{c} constraints" for c in all_constraint_levels]
    rows_acc = []
    rows_css = []

    for row in models:
        model = row["model"]
        by_condition = row.get("benchmarks", {}).get(bench_name, {}).get("by_condition", {})

        acc_by_level = defaultdict(list)
        css_by_level = defaultdict(list)

        for cond_key, stats in by_condition.items():
            cond = parse_condition_key(cond_key)
            if "num_constraints" not in cond:
                continue
            level = try_num(cond["num_constraints"])
            acc_by_level[level].append(stats.get("exact_accuracy"))
            css_by_level[level].append(stats.get("mean_constraint_satisfaction"))

        rows_acc.append([model] + [fmt(average(acc_by_level[c])) for c in all_constraint_levels])
        rows_css.append([model] + [fmt(average(css_by_level[c])) for c in all_constraint_levels])

    text = f"## Constraint Stacking: `{bench_name}`\n\n"
    text += "### Exact Accuracy by Number of Constraints\n\n"
    text += md_table(headers, rows_acc)
    text += "\n\n### Mean Constraint Satisfaction by Number of Constraints\n\n"
    text += md_table(headers, rows_css)
    text += "\n"
    return text

def build_extraneous_section(summary, bench_name):
    models = leaderboard_rows(summary)

    distractor_types = set()
    distractor_counts = set()

    for row in models:
        by_condition = row.get("benchmarks", {}).get(bench_name, {}).get("by_condition", {})
        for cond_key in by_condition:
            cond = parse_condition_key(cond_key)
            if "distractor_type" in cond:
                distractor_types.add(cond["distractor_type"])
            if "distractor_count" in cond:
                distractor_counts.add(try_num(cond["distractor_count"]))

    distractor_types = sorted(distractor_types)
    distractor_counts = sorted(distractor_counts)

    if not distractor_types:
        return None

    headers = ["Model"] + [f"{dt}" for dt in distractor_types]
    rows = []

    for row in models:
        model = row["model"]
        by_condition = row.get("benchmarks", {}).get(bench_name, {}).get("by_condition", {})
        grouped = defaultdict(list)

        for cond_key, stats in by_condition.items():
            cond = parse_condition_key(cond_key)
            dt = cond.get("distractor_type")
            if dt is None:
                continue
            grouped[dt].append(stats.get("exact_accuracy"))

        rows.append([model] + [fmt(average(grouped[dt])) for dt in distractor_types])

    text = f"## Extraneous Load: `{bench_name}`\n\n"
    text += "### Accuracy by Distractor Type\n\n"
    text += md_table(headers, rows)
    text += "\n"

    if distractor_counts:
        headers2 = ["Model"] + [f"{dc} distractors" for dc in distractor_counts]
        rows2 = []
        for row in models:
            model = row["model"]
            by_condition = row.get("benchmarks", {}).get(bench_name, {}).get("by_condition", {})
            grouped = defaultdict(list)

            for cond_key, stats in by_condition.items():
                cond = parse_condition_key(cond_key)
                dc = cond.get("distractor_count")
                if dc is None:
                    continue
                grouped[try_num(dc)].append(stats.get("exact_accuracy"))

            rows2.append([model] + [fmt(average(grouped[dc])) for dc in distractor_counts])

        text += "\n### Accuracy by Distractor Count\n\n"
        text += md_table(headers2, rows2)
        text += "\n"

    return text

def build_chunking_section(summary, bench_name):
    models = leaderboard_rows(summary)

    headers = ["Model", "Flat", "Chunked", "Gain"]
    rows = []

    for row in models:
        model = row["model"]
        by_condition = row.get("benchmarks", {}).get(bench_name, {}).get("by_condition", {})

        flat_vals = []
        chunked_vals = []

        for cond_key, stats in by_condition.items():
            cond = parse_condition_key(cond_key)
            fmt_type = cond.get("format")
            if fmt_type == "flat":
                flat_vals.append(stats.get("exact_accuracy"))
            elif fmt_type == "chunked":
                chunked_vals.append(stats.get("exact_accuracy"))

        flat_avg = average(flat_vals)
        chunked_avg = average(chunked_vals)
        gain = None if flat_avg is None or chunked_avg is None else (chunked_avg - flat_avg)

        rows.append([model, fmt(flat_avg), fmt(chunked_avg), fmt(gain)])

    text = f"## Chunking: `{bench_name}`\n\n"
    text += md_table(headers, rows)
    text += "\n"
    return text

def build_interactivity_section(summary, bench_name):
    models = leaderboard_rows(summary)

    headers = ["Model", "Low Interactivity", "High Interactivity", "Penalty"]
    rows = []

    for row in models:
        model = row["model"]
        by_condition = row.get("benchmarks", {}).get(bench_name, {}).get("by_condition", {})

        low_vals = []
        high_vals = []

        for cond_key, stats in by_condition.items():
            cond = parse_condition_key(cond_key)
            interactivity = cond.get("interactivity")
            if interactivity == "low":
                low_vals.append(stats.get("exact_accuracy"))
            elif interactivity == "high":
                high_vals.append(stats.get("exact_accuracy"))

        low_avg = average(low_vals)
        high_avg = average(high_vals)
        penalty = None if low_avg is None or high_avg is None else (low_avg - high_avg)

        rows.append([model, fmt(low_avg), fmt(high_avg), fmt(penalty)])

    text = f"## Element Interactivity: `{bench_name}`\n\n"
    text += md_table(headers, rows)
    text += "\n"
    return text

def build_interpretation_section(summary):
    rows = leaderboard_rows(summary)
    if not rows:
        return "## Interpretation\n\nNo results available."

    best_model = rows[0]["model"]

    notes = [
        f"- Top macro performer: **{best_model}**.",
        "- Compare exact accuracy to mean constraint satisfaction to test whether global coordination fails before local rule handling.",
        "- Check whether chunked presentation consistently outperforms flat presentation.",
        "- Check whether confusable distractors hurt more than irrelevant distractors.",
        "- Check whether high interactivity creates larger penalties as fact coupling increases.",
    ]

    return "## Interpretation Prompts\n\n" + "\n".join(notes)

def build_full_report(summary, title="Benchmark Results"):
    parts = [f"# {title}", ""]

    parts.append(build_overall_leaderboard(summary))
    parts.append("")
    parts.append(build_benchmark_overview(summary))
    parts.append("")

    bench_names = collect_benchmark_names(summary)
    for bench_name in bench_names:
        family = infer_family(bench_name)
        section = None
        if family == "constraint_stacking":
            section = build_constraint_section(summary, bench_name)
        elif family == "extraneous_load":
            section = build_extraneous_section(summary, bench_name)
        elif family == "chunking":
            section = build_chunking_section(summary, bench_name)
        elif family == "element_interactivity":
            section = build_interactivity_section(summary, bench_name)

        if section:
            parts.append(section)
            parts.append("")

    parts.append(build_interpretation_section(summary))
    parts.append("")

    return "\n".join(parts)


# ============================================================
# Main
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--input", required=True, help="comparison.json")
    parser.add_argument("--output", default=None, help="Optional markdown output file")
    parser.add_argument("--title", default="Benchmark Results", help="Report title")
    args = parser.parse_args()

    summary = load_json(args.input)
    report = build_full_report(summary, title=args.title)

    if args.output:
        with open(args.output, "w", encoding="utf-8") as f:
            f.write(report)
        print(f"Saved markdown report to {args.output}")
    else:
        print(report)


if __name__ == "__main__":
    main()

Example usage

Print to terminal:

python compile_results_md.py --input comparison.json


Write to a markdown file:

python compile_results_md.py --input comparison.json --output RESULTS.md --title "Tiny LLM Cognitive Load Results"

What it produces

It generates sections like:

# Tiny LLM Cognitive Load Results
## Overall Leaderboard
## Benchmark Exact Accuracy Overview
## Constraint Stacking: ...
## Extraneous Load: ...
## Chunking: ...
## Element Interactivity: ...
## Interpretation Prompts
Why this is useful

This gives you an immediate bridge from raw experiment outputs to:

repo documentation
a shareable memo
a results appendix
a paper draft skeleton
Best next step after this

At this point, the strongest next additions would be one of:

statistical testing script

paired bootstrap confidence intervals
slope comparisons
significance tests

LaTeX paper draft

convert the memo and results structure into paper form

paired-condition benchmark v2

stronger causal evidence for chunking and distractor effects

If you want the most scientifically valuable next thing, I’d recommend 1.
I can write a stats script that takes your condition-level outputs and computes effect sizes plus confidence intervals for:

chunking gain
interactivity penalty
distractor sensitivity
overload slope differences
