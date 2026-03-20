Evaluation harness

Here’s a Python evaluation harness for the JSONL datasets above. It supports:

exact-match style grading
constraint-checker grading
aggregate metrics
per-condition breakdowns
failure taxonomy for constraint stacking

It assumes you have:

a dataset JSONL
a predictions JSONL with id and prediction
Expected prediction file format
{"id": "cs_test_000001", "prediction": "BkTae7"}
{"id": "cs_test_000002", "prediction": "A9kLm2"}

Python script
import json
import re
import math
import argparse
from collections import defaultdict, Counter
from typing import Dict, Any, List, Tuple

VOWELS = set("AEIOUaeiou")

# ============================================================
# IO
# ============================================================

def load_jsonl(path: str) -> List[Dict[str, Any]]:
    rows = []
    with open(path, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if line:
                rows.append(json.loads(line))
    return rows

# ============================================================
# Normalization
# ============================================================

def normalize_text(s: str) -> str:
    if s is None:
        return ""
    s = str(s).strip()
    s = re.sub(r"\s+", " ", s)
    return s

def normalize_for_exact(s: str) -> str:
    s = normalize_text(s)
    return s.strip()

# ============================================================
# Constraint checker
# ============================================================

def is_vowel(c: str) -> bool:
    return c in VOWELS

def char_class(c: str) -> str:
    if c.isdigit():
        return "digit"
    if c.islower():
        return "lowercase"
    if c.isupper():
        return "uppercase"
    return "other"

def count_class(s: str, klass: str) -> int:
    if klass == "digit":
        return sum(ch.isdigit() for ch in s)
    if klass == "uppercase":
        return sum(ch.isupper() for ch in s)
    if klass == "lowercase":
        return sum(ch.islower() for ch in s)
    if klass == "vowel":
        return sum(is_vowel(ch) for ch in s)
    raise ValueError(f"Unknown class {klass}")

def evaluate_constraint(pred: str, rule: Dict[str, Any]) -> bool:
    t = rule["type"]

    if t == "position":
        idx = rule["index"]
        return len(pred) > idx and pred[idx] == rule["value"]

    if t == "must_include":
        return rule["value"] in pred

    if t == "all_distinct":
        return len(set(pred)) == len(pred)

    if t == "count_class":
        return count_class(pred, rule["class"]) == rule["count"]

    if t == "before":
        a = pred.find(rule["a"])
        b = pred.find(rule["b"])
        return a != -1 and b != -1 and a < b

    if t == "not_adjacent_classes":
        for i in range(len(pred) - 1):
            c1 = char_class(pred[i])
            c2 = char_class(pred[i + 1])
            if (c1 == rule["class_a"] and c2 == rule["class_b"]) or \
               (c1 == rule["class_b"] and c2 == rule["class_a"]):
                return False
        return True

    return False

def evaluate_constraint_set(pred: str, constraints: List[Dict[str, Any]]) -> Dict[str, Any]:
    pred = normalize_text(pred)
    passed = []
    for rule in constraints:
        ok = evaluate_constraint(pred, rule)
        passed.append(ok)

    total = len(constraints)
    satisfied = sum(passed)
    exact = satisfied == total

    return {
        "exact": exact,
        "satisfied": satisfied,
        "total": total,
        "per_constraint": passed,
    }

# ============================================================
# Failure taxonomy for constraint tasks
# ============================================================

def classify_constraint_failure(pred: str, constraints: List[Dict[str, Any]], per_constraint: List[bool]) -> List[str]:
    labels = []
    pred = normalize_text(pred)

    if not pred:
        return ["empty_output"]

    # Any failed constraints?
    failed_rules = [r for r, ok in zip(constraints, per_constraint) if not ok]

    # omission-like: missing required included char or wrong length-ish implied by missing positions
    for rule in failed_rules:
        if rule["type"] == "must_include":
            labels.append("constraint_omission")
            break

    # recency override: last constraint satisfied but earlier ones fail
    if len(constraints) >= 2:
        earlier_fail = any(not ok for ok in per_constraint[:-1])
        last_ok = per_constraint[-1]
        if earlier_fail and last_ok:
            labels.append("recency_override")

    # position bias: later position correct while earlier required positions fail
    position_results = [(r, ok) for r, ok in zip(constraints, per_constraint) if r["type"] == "position"]
    if len(position_results) >= 2:
        failed_early = False
        succeeded_late = False
        for r, ok in position_results:
            if r["index"] <= 1 and not ok:
                failed_early = True
            if r["index"] >= 2 and ok:
                succeeded_late = True
        if failed_early and succeeded_late:
            labels.append("position_bias")

    # global consistency failure: some constraints pass, not all
    satisfied = sum(per_constraint)
    if 0 < satisfied < len(per_constraint):
        labels.append("global_consistency_failure")

    # format confusion: output is much longer than expected constrained code
    if len(pred) > 20 or " " in pred:
        labels.append("format_confusion")

    # duplication problem
    for rule in failed_rules:
        if rule["type"] == "all_distinct":
            labels.append("duplication_failure")
            break

    # class count issue
    for rule in failed_rules:
        if rule["type"] == "count_class":
            labels.append("count_mismatch")
            break

    return sorted(set(labels))

# ============================================================
# Exact match grading
# ============================================================

def evaluate_exact(pred: str, accepted_answers: List[str]) -> bool:
    pred_n = normalize_for_exact(pred)
    accepted_n = {normalize_for_exact(x) for x in accepted_answers}
    return pred_n in accepted_n

# ============================================================
# Core evaluation
# ============================================================

def index_predictions(pred_rows: List[Dict[str, Any]]) -> Dict[str, str]:
    out = {}
    for row in pred_rows:
        if "id" in row:
            out[row["id"]] = row.get("prediction", "")
    return out

def summarize_scores(values: List[float]) -> Dict[str, float]:
    if not values:
        return {"count": 0, "mean": 0.0}
    return {
        "count": len(values),
        "mean": sum(values) / len(values),
    }

def condition_key(condition: Dict[str, Any]) -> str:
    parts = []
    for k in sorted(condition.keys()):
        parts.append(f"{k}={condition[k]}")
    return "|".join(parts)

def evaluate_dataset(gold_rows: List[Dict[str, Any]], pred_map: Dict[str, str]) -> Dict[str, Any]:
    total = 0
    exact_correct = 0

    constraint_css_scores = []
    by_condition = defaultdict(lambda: {"n": 0, "exact": 0, "css_sum": 0.0})
    failure_counter = Counter()
    missing_predictions = 0

    detailed = []

    for row in gold_rows:
        rid = row["id"]
        pred = pred_map.get(rid, "")
        if rid not in pred_map:
            missing_predictions += 1

        grader = row["grader"]
        cond_key = condition_key(row.get("condition", {}))
        total += 1
        by_condition[cond_key]["n"] += 1

        result = {
            "id": rid,
            "experiment": row.get("experiment"),
            "condition": row.get("condition", {}),
            "prediction": pred,
        }

        if grader["type"] == "exact_match_normalized":
            accepted = grader.get("accepted_answers", [row["target"]])
            ok = evaluate_exact(pred, accepted)
            if ok:
                exact_correct += 1
                by_condition[cond_key]["exact"] += 1

            result["exact"] = ok
            result["target"] = row["target"]

        elif grader["type"] == "constraint_checker":
            constraints = grader["constraints"]
            ev = evaluate_constraint_set(pred, constraints)
            css = ev["satisfied"] / ev["total"] if ev["total"] > 0 else 0.0

            if ev["exact"]:
                exact_correct += 1
                by_condition[cond_key]["exact"] += 1

            by_condition[cond_key]["css_sum"] += css
            constraint_css_scores.append(css)

            failures = []
            if not ev["exact"]:
                failures = classify_constraint_failure(pred, constraints, ev["per_constraint"])
                for f in failures:
                    failure_counter[f] += 1

            result.update({
                "exact": ev["exact"],
                "constraint_satisfaction": css,
                "constraints_total": ev["total"],
                "constraints_satisfied": ev["satisfied"],
                "per_constraint": ev["per_constraint"],
                "failures": failures,
                "target_example": row["target"],
            })
        else:
            raise ValueError(f"Unknown grader type: {grader['type']}")

        detailed.append(result)

    overall_exact = exact_correct / total if total else 0.0

    condition_summary = {}
    for ck, stats in by_condition.items():
        n = stats["n"]
        condition_summary[ck] = {
            "n": n,
            "exact_accuracy": stats["exact"] / n if n else 0.0,
            "mean_constraint_satisfaction": stats["css_sum"] / n if n else 0.0,
        }

    summary = {
        "n_examples": total,
        "missing_predictions": missing_predictions,
        "exact_accuracy": overall_exact,
        "mean_constraint_satisfaction": (
            sum(constraint_css_scores) / len(constraint_css_scores)
            if constraint_css_scores else None
        ),
        "failure_counts": dict(failure_counter),
        "by_condition": condition_summary,
        "detailed_results": detailed,
    }
    return summary

# ============================================================
# Pretty printing
# ============================================================

def print_summary(summary: Dict[str, Any], max_conditions: int = 20) -> None:
    print("=== Overall ===")
    print(f"Examples: {summary['n_examples']}")
    print(f"Missing predictions: {summary['missing_predictions']}")
    print(f"Exact accuracy: {summary['exact_accuracy']:.4f}")
    if summary["mean_constraint_satisfaction"] is not None:
        print(f"Mean constraint satisfaction: {summary['mean_constraint_satisfaction']:.4f}")

    if summary["failure_counts"]:
        print("\n=== Failure counts ===")
        for k, v in sorted(summary["failure_counts"].items(), key=lambda x: (-x[1], x[0])):
            print(f"{k}: {v}")

    print("\n=== By condition ===")
    items = list(summary["by_condition"].items())
    items.sort(key=lambda x: (-x[1]["n"], x[0]))
    for ck, stats in items[:max_conditions]:
        print(
            f"{ck}\n"
            f"  n={stats['n']} "
            f"exact_accuracy={stats['exact_accuracy']:.4f} "
            f"mean_constraint_satisfaction={stats['mean_constraint_satisfaction']:.4f}"
        )

# ============================================================
# CLI
# ============================================================

def main():
    parser = argparse.ArgumentParser()
    parser.add_argument("--gold", required=True, help="Gold dataset JSONL")
    parser.add_argument("--pred", required=True, help="Predictions JSONL with id and prediction")
    parser.add_argument("--out", default=None, help="Optional output summary JSON")
    args = parser.parse_args()

    gold_rows = load_jsonl(args.gold)
    pred_rows = load_jsonl(args.pred)
    pred_map = index_predictions(pred_rows)

    summary = evaluate_dataset(gold_rows, pred_map)
    print_summary(summary)

    if args.out:
        with open(args.out, "w", encoding="utf-8") as f:
            json.dump(summary, f, indent=2, ensure_ascii=False)
        print(f"\nSaved summary to {args.out}")

if __name__ == "__main__":
    main()

Example usage
python evaluate.py --gold constraint_stacking_test.jsonl --pred my_preds.jsonl --out cs_summary.json


Or for another dataset:

python evaluate.py --gold extraneous_load_test.jsonl --pred my_preds.jsonl --out el_summary.json

What it computes
Overall
number of examples
missing predictions
exact accuracy
mean constraint satisfaction for constraint tasks
By condition

Grouped by the dataset’s condition fields, for example:

num_constraints=4|output_length=6|...
distractor_count=4|distractor_type=confusable|...
format=chunked|num_groups=4|...
interactivity=high|reasoning_hops=3|...
Failure counts for constraint stacking
constraint_omission
recency_override
position_bias
global_consistency_failure
format_confusion
duplication_failure
count_mismatch
empty_output
Recommended next improvements

If you want this to become a proper benchmark suite, the next upgrades should be:

1. Paired comparisons

For chunking and distractor experiments, generate paired twin examples so you can compare:

same latent world
different presentation

That gives much stronger statistical signal.

2. More granular failure taxonomy

For extraneous load and interactivity:

distractor substitution
partial rule use
independent fact success but integration failure
answer-source confusion
3. Confidence / logprob support

If your inference stack exposes token logprobs, add:

logprob of correct answer
entropy
calibration gap
4. Overload curve plotting

From the summary JSON, you can plot:

accuracy vs number of constraints
accuracy vs distractor count
chunked vs flat accuracy
low vs high interactivity

That’s where the theory becomes visually obvious.

Suggested prediction generator format

If you later want a full pipeline, use a generation script that emits:

{"id": "...", "prediction": "...", "raw_output": "...", "model": "my-model", "temperature": 0.0}


The evaluator only needs id and prediction, but richer metadata is useful.

Best next step

The most useful next artifact would probably be one of:

a model inference runner that reads the JSONL benchmark and produces prediction files
a plotting script to visualize overload curves from the evaluator output
a benchmark paper skeleton describing the methodology and hypotheses

If you want, I’d suggest 2 next because once you have curves, the overload story becomes immediately legible.
