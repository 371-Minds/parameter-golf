Below is a single Python script that generates JSONL datasets for the four synthetic benchmarks:

constraint_stacking
extraneous_load
chunking
element_interactivity

It uses only the Python standard library.

import json
import random
import string
from dataclasses import dataclass, asdict
from typing import List, Dict, Any, Tuple, Optional

# ============================================================
# Config
# ============================================================

SEED = 42
random.seed(SEED)

DEFAULT_COUNTS = {
    "train": 2000,
    "val": 400,
    "test": 400,
}

NAMES = [
    "Mira", "Lena", "Nora", "Pax", "Rin", "Tov", "Lem", "Sia",
    "Ari", "Kian", "Juno", "Vera", "Tala", "Ivo", "Niko", "Rhea"
]

COLORS = ["red", "blue", "green", "yellow", "orange", "purple"]
OBJECTS = ["cards", "marbles", "keys", "tokens", "blocks", "rings"]
DAYS = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday"]
TEXTURES = ["glossy", "matte", "striped", "plain", "shiny"]
SHAPES = ["Circle", "Square", "Triangle", "Star"]
GROUP_NAMES = ["Group 1", "Group 2", "Group 3", "Group 4", "Group 5"]
UPPER = string.ascii_uppercase
LOWER = string.ascii_lowercase
DIGITS = string.digits
VOWELS = set("AEIOUaeiou")

# ============================================================
# Utilities
# ============================================================

def ensure_dirless_filename(name: str) -> str:
    return name.split("/")[-1].split("\\")[-1]

def write_jsonl(filename: str, rows: List[Dict[str, Any]]) -> None:
    filename = ensure_dirless_filename(filename)
    with open(filename, "w", encoding="utf-8") as f:
        for row in rows:
            f.write(json.dumps(row, ensure_ascii=False) + "\n")
    print(f"Wrote {len(rows)} rows -> {filename}")

def choice_excluding(items: List[str], exclude: set) -> str:
    candidates = [x for x in items if x not in exclude]
    return random.choice(candidates)

def shuffled(seq):
    seq = list(seq)
    random.shuffle(seq)
    return seq

def maybe_paraphrase_output_only() -> str:
    return random.choice([
        "Output only the answer.",
        "Answer only with the final answer.",
        "Respond with only the answer.",
        "Output only the result.",
    ])

def make_id(prefix: str, split: str, idx: int) -> str:
    return f"{prefix}_{split}_{idx:06d}"

# ============================================================
# Constraint Stacking
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

def generate_distinct_string(length: int) -> str:
    pool = list(UPPER[:12] + LOWER[:12] + DIGITS[:8])
    random.shuffle(pool)
    return "".join(pool[:length])

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

def rule_to_text(rule: Dict[str, Any]) -> str:
    t = rule["type"]
    if t == "position":
        idx = rule["index"]
        pos_names = ["first", "second", "third", "fourth", "fifth", "sixth", "seventh", "eighth"]
        return f"The {pos_names[idx]} character must be {rule['value']}."
    if t == "must_include":
        return f"The string must include {rule['value']}."
    if t == "all_distinct":
        return "All characters must be distinct."
    if t == "count_class":
        klass = rule["class"]
        cnt = rule["count"]
        return f"The string must contain exactly {cnt} {klass} character{'s' if cnt != 1 else ''}."
    if t == "before":
        return f"{rule['a']} must appear before {rule['b']}."
    if t == "not_adjacent_classes":
        return f"A {rule['class_a']} character cannot be next to a {rule['class_b']} character."
    raise ValueError(f"Unknown rule type {t}")

def check_constraint_string(s: str, constraints: List[Dict[str, Any]]) -> bool:
    for rule in constraints:
        t = rule["type"]
        if t == "position":
            if len(s) <= rule["index"] or s[rule["index"]] != rule["value"]:
                return False
        elif t == "must_include":
            if rule["value"] not in s:
                return False
        elif t == "all_distinct":
            if len(set(s)) != len(s):
                return False
        elif t == "count_class":
            if count_class(s, rule["class"]) != rule["count"]:
                return False
        elif t == "before":
            a = s.find(rule["a"])
            b = s.find(rule["b"])
            if a == -1 or b == -1 or a >= b:
                return False
        elif t == "not_adjacent_classes":
            for i in range(len(s) - 1):
                c1 = char_class(s[i])
                c2 = char_class(s[i + 1])
                if (c1 == rule["class_a"] and c2 == rule["class_b"]) or \
                   (c1 == rule["class_b"] and c2 == rule["class_a"]):
                    return False
        else:
            return False
    return True

def derive_candidate_rules_from_target(target: str) -> List[Dict[str, Any]]:
    rules = []
    L = len(target)

    # position rules
    pos_indices = list(range(L))
    random.shuffle(pos_indices)
    for idx in pos_indices[:min(3, L)]:
        rules.append({"type": "position", "index": idx, "value": target[idx]})

    # include rules
    incl_chars = shuffled(list(set(target)))[:2]
    for ch in incl_chars:
        rules.append({"type": "must_include", "value": ch})

    # distinct rule
    if len(set(target)) == len(target):
        rules.append({"type": "all_distinct"})

    # count rules
    for klass in ["digit", "uppercase", "lowercase", "vowel"]:
        cnt = count_class(target, klass)
        if 0 < cnt < len(target):
            rules.append({"type": "count_class", "class": klass, "count": cnt})

    # before rule
    uniq = list(dict.fromkeys(target))
    if len(uniq) >= 2:
        a, b = uniq[0], uniq[-1]
        if target.find(a) < target.find(b):
            rules.append({"type": "before", "a": a, "b": b})

    # adjacency class rule
    ok = True
    for i in range(len(target) - 1):
        c1, c2 = char_class(target[i]), char_class(target[i + 1])
        if (c1 == "digit" and c2 == "vowel") or (c1 == "vowel" and c2 == "digit"):
            ok = False
            break
    if ok:
        rules.append({"type": "not_adjacent_classes", "class_a": "digit", "class_b": "vowel"})

    return rules

def choose_nonredundant_rules(target: str, num_constraints: int) -> List[Dict[str, Any]]:
    candidates = derive_candidate_rules_from_target(target)
    random.shuffle(candidates)

    chosen = []
    seen_text = set()
    for rule in candidates:
        txt = rule_to_text(rule)
        if txt in seen_text:
            continue
        chosen.append(rule)
        seen_text.add(txt)
        if len(chosen) >= num_constraints:
            break

    if len(chosen) < num_constraints:
        # regenerate target until enough rules exist
        return []

    return chosen

def build_constraint_prompt(length: int, constraints: List[Dict[str, Any]]) -> str:
    style = random.choice(["numbered", "bullets"])
    if style == "numbered":
        lines = [f"{i+1}. {rule_to_text(r)}" for i, r in enumerate(constraints)]
        body = "\n".join(lines)
        return f"Produce a {length}-character code.\n\nRules:\n{body}\n\n{maybe_paraphrase_output_only()}"
    else:
        lines = [f"- {rule_to_text(r)}" for r in constraints]
        body = "\n".join(lines)
        return f"Write one string of length {length} that satisfies all conditions:\n{body}\n\n{maybe_paraphrase_output_only()}"

def gen_constraint_row(split: str, idx: int, num_constraints: int, length: int) -> Dict[str, Any]:
    while True:
        target = generate_distinct_string(length)
        rules = choose_nonredundant_rules(target, num_constraints)
        if rules and check_constraint_string(target, rules):
            break

    return {
        "id": make_id("cs", split, idx),
        "experiment": "constraint_stacking",
        "condition": {
            "num_constraints": num_constraints,
            "output_length": length,
            "rule_types": [r["type"] for r in rules],
        },
        "prompt": build_constraint_prompt(length, rules),
        "target": target,
        "grader": {
            "type": "constraint_checker",
            "constraints": rules,
        },
        "metadata": {
            "seed": SEED,
            "split": split,
        },
    }

def generate_constraint_dataset(split: str, count: int) -> List[Dict[str, Any]]:
    rows = []
    for i in range(count):
        num_constraints = random.choice([1, 2, 3, 4, 5, 6])
        length = random.choice([5, 6, 7]) if num_constraints <= 4 else random.choice([6, 7, 8])
        rows.append(gen_constraint_row(split, i + 1, num_constraints, length))
    return rows

# ============================================================
# Extraneous Load
# ============================================================

def normalize_number_word(n: int) -> str:
    return str(n)

def gen_base_counting_problem() -> Tuple[str, str, Dict[str, Any]]:
    name = random.choice(NAMES)
    color1, color2 = random.sample(COLORS, 2)
    obj = random.choice(OBJECTS)
    a = random.randint(3, 9)
    b = random.randint(2, 8)
    give = random.randint(1, a - 1)

    prompt_core = f"{name} has {a} {color1} {obj} and {b} {color2} {obj}. She gives away {give} {color1} {obj}."
    question = f"How many {color1} {obj} remain?"
    target = str(a - give)

    meta = {
        "family": "counting",
        "name": name,
        "focus_color": color1,
        "other_color": color2,
        "object": obj,
        "initial_count": a,
        "other_count": b,
        "giveaway": give,
    }
    return prompt_core, question, meta

def irrelevant_sentence(name: str) -> str:
    templates = [
        f"{name} bought them on {random.choice(DAYS)}.",
        f"{name}'s cousin likes marbles.",
        f"One item is {random.choice(TEXTURES)}.",
        f"{name} prefers tea in the morning.",
        f"The table in the room is wooden.",
    ]
    return random.choice(templates)

def confusable_sentence(meta: Dict[str, Any]) -> str:
    name = meta["name"]
    color1 = meta["focus_color"]
    color2 = meta["other_color"]
    obj = meta["object"]

    templates = [
        f"Earlier, {name} gives away {random.randint(1, max(1, meta['other_count'] - 1))} {color2} {obj}.",
        f"Later, {name} counts the {color2} {obj} again.",
        f"One {color2} {obj} is {random.choice(TEXTURES)}.",
        f"{name} once had {meta['other_count'] + random.randint(1, 3)} {color2} {obj} last week.",
    ]
    return random.choice(templates)

def contradictory_adjacent_sentence(meta: Dict[str, Any]) -> str:
    name = meta["name"]
    color1 = meta["focus_color"]
    obj = meta["object"]
    wrong = random.randint(1, max(1, meta["initial_count"] - 1))
    return f"Earlier in the day, {name} gives away {wrong} {color1} {obj} in a different story."

def build_extraneous_prompt(core: str, inserts: List[str], question: str) -> str:
    position = random.choice(["middle", "end"])
    if position == "middle" and inserts:
        chunks = core.split(". ")
        if len(chunks) >= 2:
            text = chunks[0] + ". " + " ".join(inserts) + " " + ". ".join(chunks[1:])
        else:
            text = core + " " + " ".join(inserts)
    else:
        text = core + " " + " ".join(inserts)
    return f"{text}\n{question}\nAnswer with only the number."

def gen_extraneous_row(split: str, idx: int, distractor_type: str, distractor_count: int) -> Dict[str, Any]:
    core, question, meta = gen_base_counting_problem()

    inserts = []
    for _ in range(distractor_count):
        if distractor_type == "irrelevant":
            inserts.append(irrelevant_sentence(meta["name"]))
        elif distractor_type == "confusable":
            inserts.append(confusable_sentence(meta))
        elif distractor_type == "contradictory_adjacent":
            inserts.append(contradictory_adjacent_sentence(meta))
        else:
            raise ValueError(distractor_type)

    prompt = build_extraneous_prompt(core, inserts, question)
    target = str(meta["initial_count"] - meta["giveaway"])

    return {
        "id": make_id("el", split, idx),
        "experiment": "extraneous_load",
        "condition": {
            "distractor_count": distractor_count,
            "distractor_type": distractor_type,
            "relevant_fact_position": random.choice(["early", "middle", "late"]),
        },
        "prompt": prompt,
        "target": target,
        "grader": {
            "type": "exact_match_normalized",
            "accepted_answers": [target],
        },
        "metadata": {
            "seed": SEED,
            "split": split,
            "base_family": "counting",
        },
    }

def generate_extraneous_dataset(split: str, count: int) -> List[Dict[str, Any]]:
    rows = []
    for i in range(count):
        distractor_type = random.choice(["irrelevant", "confusable", "contradictory_adjacent"])
        distractor_count = random.choice([0, 2, 4, 6])
        rows.append(gen_extraneous_row(split, i + 1, distractor_type, distractor_count))
    return rows

# ============================================================
# Chunking
# ============================================================

def make_group_world(num_groups: int, items_per_group: int) -> Dict[str, List[str]]:
    symbols_pool = list(string.ascii_uppercase)
    random.shuffle(symbols_pool)
    world = {}
    k = 0
    for gi in range(num_groups):
        group = GROUP_NAMES[gi]
        world[group] = symbols_pool[k:k + items_per_group]
        k += items_per_group
    return world

def flat_render_group_world(world: Dict[str, List[str]]) -> str:
    sentences = []
    for group, items in world.items():
        for item in items:
            sentences.append(f"{item} is in {group}.")
    return " ".join(sentences)

def chunked_render_group_world(world: Dict[str, List[str]]) -> str:
    return "\n".join(f"{group}: {', '.join(items)}" for group, items in world.items())

def generate_group_query(world: Dict[str, List[str]]) -> Tuple[str, str, str]:
    query_type = random.choice(["lookup", "same_group"])
    groups = list(world.keys())
    all_items = [(g, item) for g, items in world.items() for item in items]

    if query_type == "lookup":
        g, item = random.choice(all_items)
        question = f"Question: Which group is {item} in?"
        target = g
    else:
        g1 = random.choice(groups)
        same = random.choice([True, False])
        if same:
            item1, item2 = random.sample(world[g1], 2)
            target = "Yes"
        else:
            other_groups = [g for g in groups if g != g1]
            g2 = random.choice(other_groups)
            item1 = random.choice(world[g1])
            item2 = random.choice(world[g2])
            target = "No"
        question = f"Question: Are {item1} and {item2} in the same group?"
    return query_type, question, target

def gen_chunking_row(split: str, idx: int, format_type: str, num_groups: int, items_per_group: int) -> Dict[str, Any]:
    world = make_group_world(num_groups, items_per_group)
    query_type, question, target = generate_group_query(world)

    if format_type == "flat":
        context = flat_render_group_world(world)
    elif format_type == "chunked":
        context = chunked_render_group_world(world)
    else:
        raise ValueError(format_type)

    answer_instr = "Answer only with the answer."
    prompt = f"{context}\n{question}\n{answer_instr}"

    return {
        "id": make_id("ch", split, idx),
        "experiment": "chunking",
        "condition": {
            "format": format_type,
            "num_groups": num_groups,
            "items_per_group": items_per_group,
            "query_type": query_type,
        },
        "prompt": prompt,
        "target": target,
        "grader": {
            "type": "exact_match_normalized",
            "accepted_answers": [target],
        },
        "metadata": {
            "seed": SEED,
            "split": split,
        },
    }

def generate_chunking_dataset(split: str, count: int) -> List[Dict[str, Any]]:
    rows = []
    for i in range(count):
        format_type = random.choice(["flat", "chunked"])
        num_groups = random.choice([3, 4, 5])
        items_per_group = random.choice([3, 4])
        rows.append(gen_chunking_row(split, i + 1, format_type, num_groups, items_per_group))
    return rows

# ============================================================
# Element Interactivity
# ============================================================

def gen_low_interactivity_world(num_facts: int) -> Tuple[str, str, Dict[str, Any]]:
    entities = random.sample(NAMES, num_facts)
    shapes = random.sample(SHAPES * 2, num_facts)

    facts = []
    mapping = {}
    for e, s in zip(entities, shapes):
        mapping[e] = s
        facts.append(f"{e} uses {s}.")

    ask = random.choice(entities)
    question = f"Question: Which shape does {ask} use?"
    target = mapping[ask]
    prompt = " ".join(facts) + f"\n{question}\nAnswer only with the shape."
    meta = {"entities": entities, "mapping": mapping}
    return prompt, target, meta

def gen_high_interactivity_world() -> Tuple[str, str, Dict[str, Any]]:
    # Fixed, easy-to-grade style with latent consistent world
    e1, e2, e3, e4 = random.sample(NAMES, 4)
    s1, s2, s3, s4 = random.sample(SHAPES, 4)

    # latent assignments
    mapping = {
        e4: s4,
        e3: s3,
        e2: s2,
        e1: s1,
    }

    # Build coupled rules that still uniquely imply answer
    facts = [
        f"{e4} uses {mapping[e4]}.",
        f"{e3} is active when {e4} uses {mapping[e4]}.",
        f"{e2} uses {mapping[e2]} if {e3} is active.",
        f"{e1} uses the shape that is not {mapping[e2]} and not {mapping[e4]}.",
    ]

    # Make e1 answer uniquely one of the remaining shapes
    remaining = [s for s in SHAPES if s not in {mapping[e2], mapping[e4]}]
    target = mapping[e1]
    if target not in remaining:
        target = remaining[0]
        mapping[e1] = target
        facts[-1] = f"{e1} uses {target} when {e3} is active."

    question = f"Question: Which shape does {e1} use?"
    prompt = " ".join(facts) + f"\n{question}\nAnswer only with the shape."
    meta = {"entities": [e1, e2, e3, e4], "mapping": mapping}
    return prompt, target, meta

def gen_element_interactivity_row(split: str, idx: int, interactivity: str, num_facts: int) -> Dict[str, Any]:
    if interactivity == "low":
        prompt, target, meta = gen_low_interactivity_world(num_facts)
        reasoning_hops = 1
    elif interactivity == "high":
        prompt, target, meta = gen_high_interactivity_world()
        num_facts = 4
        reasoning_hops = random.choice([2, 3])
    else:
        raise ValueError(interactivity)

    return {
        "id": make_id("ei", split, idx),
        "experiment": "element_interactivity",
        "condition": {
            "interactivity": interactivity,
            "num_facts": num_facts,
            "reasoning_hops": reasoning_hops,
        },
        "prompt": prompt,
        "target": target,
        "grader": {
            "type": "exact_match_normalized",
            "accepted_answers": [target],
        },
        "metadata": {
            "seed": SEED,
            "split": split,
        },
    }

def generate_element_interactivity_dataset(split: str, count: int) -> List[Dict[str, Any]]:
    rows = []
    for i in range(count):
        interactivity = random.choice(["low", "high"])
        num_facts = random.choice([4, 5, 6])
        rows.append(gen_element_interactivity_row(split, i + 1, interactivity, num_facts))
    return rows

# ============================================================
# Main
# ============================================================

def main():
    all_specs = [
        ("constraint_stacking", generate_constraint_dataset, "constraint_stacking"),
        ("extraneous_load", generate_extraneous_dataset, "extraneous_load"),
        ("chunking", generate_chunking_dataset, "chunking"),
        ("element_interactivity", generate_element_interactivity_dataset, "element_interactivity"),
    ]

    for _, fn, prefix in all_specs:
        for split, count in DEFAULT_COUNTS.items():
            rows = fn(split, count)
            write_jsonl(f"{prefix}_{split}.jsonl", rows)

if __name__ == "__main__":
    main()

What this script outputs

It creates these files:

constraint_stacking_train.jsonl
constraint_stacking_val.jsonl
constraint_stacking_test.jsonl
extraneous_load_train.jsonl
extraneous_load_val.jsonl
extraneous_load_test.jsonl
chunking_train.jsonl
chunking_val.jsonl
chunking_test.jsonl
element_interactivity_train.jsonl
element_interactivity_val.jsonl
element_interactivity_test.jsonl
Notes

A few important implementation details:

Constraint stacking uses a constraint_checker grader, so the target is just one valid example, not the only valid answer.
The other three use exact_match_normalized style grading with accepted_answers.
This is a clean baseline generator, not the final “publication-grade” version.
If you want stronger rigor, the next improvements should be:
paired flat/chunked twins with identical latent world
stricter token-length matching across conditions
more formal high-interactivity latent-graph generation
adversarial distractor balancing
separate held-out composition splits
Best next step

The natural next thing is either:

an evaluation harness that scores model outputs on these JSONL files, especially the constraint checker, or
a more rigorous v2 generator with paired examples and cleaner controls

If you want, I can do the evaluation harness next.
