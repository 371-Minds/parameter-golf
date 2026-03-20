Paired-condition benchmark v2 generator

Yes — this is the right next step.

A paired benchmark design is much stronger because you can compare:

the same latent item
under two different presentations
with everything else held constant

That lets you say things like:

“chunking improved performance on the same item”
“confusable distractors impaired performance relative to irrelevant distractors for the same underlying fact structure”
“high interactivity hurt more than low interactivity for matched content”

That is much stronger than comparing condition averages across separately generated examples.

What this v2 generator will produce

It generates paired JSONL benchmarks for:

Chunking v2

same facts
flat vs chunked

Extraneous Load v2

same base context and query
irrelevant vs confusable vs contradictory-adjacent distractors

Element Interactivity v2

matched content budget
low-interactivity vs high-interactivity versions

Each item includes:

pair_id
variant
base_item_id
same answer target across conditions where appropriate
explicit condition metadata
Design principles

This version tries to improve rigor by ensuring:

shared latent structure within pairs
auto-gradable targets
controlled surface variation
enough metadata for paired statistical tests later
Python script
import json
import random
import string
from typing import List, Dict, Any

random.seed(7)

# ============================================================
# Utilities
# ============================================================

def write_jsonl(path: str, rows: List[Dict[str, Any]]) -> None:
    with open(path, "w", encoding="utf-8") as f:
        for row in rows:
            f.write(json.dumps(row, ensure_ascii=False) + "\n")

def split_rows(rows, train_n, val_n, test_n):
    assert train_n + val_n + test_n <= len(rows)
    return (
        rows[:train_n],
        rows[train_n:train_n+val_n],
        rows[train_n+val_n:train_n+val_n+test_n],
    )

def shuffled(xs):
    xs = list(xs)
    random.shuffle(xs)
    return xs

def sample_name():
    first = random.choice([
        "Lina", "Omar", "Nia", "Tariq", "Mira", "Jon", "Sana", "Pavel",
        "Iris", "Noel", "Asha", "Kian", "Rhea", "Milo", "Anya", "Zane"
    ])
    last = random.choice([
        "Reed", "Stone", "Vale", "Cross", "Frost", "Quill", "Wren", "Dunn",
        "Hart", "Bishop", "Lane", "Pike", "Shaw", "Blake", "Kerr", "Sloan"
    ])
    return f"{first} {last}"

def sample_city():
    return random.choice([
        "Lisbon", "Kyoto", "Nairobi", "Oslo", "Lima", "Seoul",
        "Prague", "Accra", "Hobart", "Quito", "Dakar", "Bergen"
    ])

def sample_color():
    return random.choice([
        "red", "blue", "green", "yellow", "purple", "orange", "silver", "black"
    ])

def sample_pet():
    return random.choice([
        "cat", "dog", "rabbit", "parrot", "turtle", "hamster"
    ])

def sample_food():
    return random.choice([
        "soup", "pasta", "rice", "bread", "salad", "curry"
    ])

def sample_object():
    return random.choice([
        "lamp", "book", "chair", "clock", "bottle", "notebook", "scarf", "key"
    ])

def exact_grader(target: str):
    return {
        "type": "exact_match_normalized",
        "accepted_answers": [target]
    }

# ============================================================
# Chunking v2
# ============================================================

def make_chunking_pair(idx: int) -> List[Dict[str, Any]]:
    person = sample_name()
    city = sample_city()
    color = sample_color()
    pet = sample_pet()
    food = sample_food()
    obj = sample_object()

    facts = [
        f"{person} lives in {city}.",
        f"{person}'s favorite color is {color}.",
        f"{person} has a {pet}.",
        f"{person}'s preferred meal is {food}.",
        f"{person} carries a {obj}."
    ]

    query_type = random.choice(["lookup", "comparison"])
    if query_type == "lookup":
        question = f"What is {person}'s favorite color?"
        target = color
    else:
        question = f"Which does {person} have: a {pet} or a bicycle?"
        target = pet

    flat_prompt = "Read the passage and answer briefly.\n\n" + " ".join(facts) + f"\n\nQuestion: {question}"
    chunked_prompt = (
        "Read the structured profile and answer briefly.\n\n"
        f"Person: {person}\n"
        f"City: {city}\n"
        f"Favorite color: {color}\n"
        f"Pet: {pet}\n"
        f"Preferred meal: {food}\n"
        f"Carries: {obj}\n\n"
        f"Question: {question}"
    )

    pair_id = f"chunk_v2_pair_{idx:06d}"

    rows = []
    for variant, prompt, fmt in [
        ("flat", flat_prompt, "flat"),
        ("chunked", chunked_prompt, "chunked"),
    ]:
        rows.append({
            "id": f"{pair_id}_{variant}",
            "pair_id": pair_id,
            "base_item_id": idx,
            "experiment": "chunking_v2",
            "variant": variant,
            "prompt": prompt,
            "target": target,
            "condition": {
                "format": fmt,
                "query_type": query_type,
                "num_facts": len(facts),
            },
            "grader": exact_grader(target),
        })

    return rows

def generate_chunking_v2(n_pairs: int) -> List[Dict[str, Any]]:
    rows = []
    for i in range(n_pairs):
        rows.extend(make_chunking_pair(i))
    return rows

# ============================================================
# Extraneous load v2
# ============================================================

def make_extraneous_base():
    person = sample_name()
    city = sample_city()
    color = sample_color()
    pet = sample_pet()
    obj = sample_object()

    relevant = [
        f"{person} lives in {city}.",
        f"{person}'s favorite color is {color}.",
        f"{person} owns a {pet}.",
        f"{person} keeps a {obj} in the office."
    ]
    question = f"What city does {person} live in?"
    target = city

    return {
        "person": person,
        "city": city,
        "color": color,
        "pet": pet,
        "obj": obj,
        "relevant": relevant,
        "question": question,
        "target": target,
    }

def irrelevant_distractors(base):
    return [
        f"The hallway contains a mural.",
        f"Yesterday there was light rain.",
        f"A nearby shop sells pencils.",
    ]

def confusable_distractors(base):
    other_city = random.choice([c for c in [
        "Lisbon", "Kyoto", "Nairobi", "Oslo", "Lima", "Seoul",
        "Prague", "Accra", "Hobart", "Quito", "Dakar", "Bergen"
    ] if c != base["city"]])

    return [
        f"{base['person']}'s cousin lives in {other_city}.",
        f"A postcard from {other_city} was on the desk.",
        f"Someone once guessed that {base['person']} lived in {other_city}."
    ]

def contradictory_adjacent_distractors(base):
    other_city = random.choice([c for c in [
        "Lisbon", "Kyoto", "Nairobi", "Oslo", "Lima", "Seoul",
        "Prague", "Accra", "Hobart", "Quito", "Dakar", "Bergen"
    ] if c != base["city"]])

    return [
        f"{base['person']} lives in {other_city}.",
        f"Correction: ignore the previous sentence if contradicted by the profile.",
        f"{base['person']}'s favorite color is {base['color']}."
    ]

def make_extraneous_pair(idx: int) -> List[Dict[str, Any]]:
    base = make_extraneous_base()
    pair_id = f"extr_v2_pair_{idx:06d}"

    variants = [
        ("irrelevant", irrelevant_distractors(base)),
        ("confusable", confusable_distractors(base)),
        ("contradictory_adjacent", contradictory_adjacent_distractors(base)),
    ]

    rows = []
    for variant, distractors in variants:
        blocks = base["relevant"][:]
        insert_pos = random.randint(1, len(blocks))
        context = blocks[:insert_pos] + distractors + blocks[insert_pos:]

        prompt = (
            "Read the passage and answer briefly.\n\n"
            + " ".join(context)
            + f"\n\nQuestion: {base['question']}"
        )

        rows.append({
            "id": f"{pair_id}_{variant}",
            "pair_id": pair_id,
            "base_item_id": idx,
            "experiment": "extraneous_load_v2",
            "variant": variant,
            "prompt": prompt,
            "target": base["target"],
            "condition": {
                "distractor_type": variant,
                "distractor_count": len(distractors),
                "num_relevant_facts": len(base["relevant"]),
            },
            "grader": exact_grader(base["target"]),
        })

    return rows

def generate_extraneous_v2(n_pairs: int) -> List[Dict[str, Any]]:
    rows = []
    for i in range(n_pairs):
        rows.extend(make_extraneous_pair(i))
    return rows

# ============================================================
# Element interactivity v2
# ============================================================

def make_low_interactivity_item(person, city, color, pet, food):
    facts = [
        f"{person} lives in {city}.",
        f"{person}'s favorite color is {color}.",
        f"{person} owns a {pet}.",
        f"{person}'s preferred meal is {food}.",
    ]
    question = f"What pet does {person} own?"
    target = pet
    return facts, question, target

def make_high_interactivity_item(person, city, color, pet, food):
    code = random.choice(["alpha", "beta", "gamma", "delta"])
    facts = [
        f"The person in {city} uses code {code}.",
        f"The person with code {code} prefers {food}.",
        f"The person who prefers {food} owns a {pet}.",
        f"The person who owns a {pet} likes {color}.",
        f"{person} lives in {city}.",
    ]
    question = f"What pet does {person} own?"
    target = pet
    return facts, question, target

def make_interactivity_pair(idx: int) -> List[Dict[str, Any]]:
    person = sample_name()
    city = sample_city()
    color = sample_color()
    pet = sample_pet()
    food = sample_food()

    low_facts, low_q, low_target = make_low_interactivity_item(person, city, color, pet, food)
    high_facts, high_q, high_target = make_high_interactivity_item(person, city, color, pet, food)

    pair_id = f"inter_v2_pair_{idx:06d}"

    rows = []
    for variant, facts, q, target, inter in [
        ("low", low_facts, low_q, low_target, "low"),
        ("high", high_facts, high_q, high_target, "high"),
    ]:
        prompt = "Read the passage and answer briefly.\n\n" + " ".join(facts) + f"\n\nQuestion: {q}"

        rows.append({
            "id": f"{pair_id}_{variant}",
            "pair_id": pair_id,
            "base_item_id": idx,
            "experiment": "element_interactivity_v2",
            "variant": variant,
            "prompt": prompt,
            "target": target,
            "condition": {
                "interactivity": inter,
                "num_facts": len(facts),
            },
            "grader": exact_grader(target),
        })

    return rows

def generate_interactivity_v2(n_pairs: int) -> List[Dict[str, Any]]:
    rows = []
    for i in range(n_pairs):
        rows.extend(make_interactivity_pair(i))
    return rows

# ============================================================
# Main
# ============================================================

def main():
    chunk_rows = generate_chunking_v2(600)
    extr_rows = generate_extraneous_v2(600)
    inter_rows = generate_interactivity_v2(600)

    random.shuffle(chunk_rows)
    random.shuffle(extr_rows)
    random.shuffle(inter_rows)

    c_train, c_val, c_test = split_rows(chunk_rows, 800, 200, 200)
    e_train, e_val, e_test = split_rows(extr_rows, 1200, 300, 300)
    i_train, i_val, i_test = split_rows(inter_rows, 800, 200, 200)

    write_jsonl("chunking_v2_train.jsonl", c_train)
    write_jsonl("chunking_v2_val.jsonl", c_val)
    write_jsonl("chunking_v2_test.jsonl", c_test)

    write_jsonl("extraneous_load_v2_train.jsonl", e_train)
    write_jsonl("extraneous_load_v2_val.jsonl", e_val)
    write_jsonl("extraneous_load_v2_test.jsonl", e_test)

    write_jsonl("element_interactivity_v2_train.jsonl", i_train)
    write_jsonl("element_interactivity_v2_val.jsonl", i_val)
    write_jsonl("element_interactivity_v2_test.jsonl", i_test)

    print("Generated paired-condition v2 datasets:")
    print("  - chunking_v2_{train,val,test}.jsonl")
    print("  - extraneous_load_v2_{train,val,test}.jsonl")
    print("  - element_interactivity_v2_{train,val,test}.jsonl")

if __name__ == "__main__":
    main()

Important note about splitting

This version shuffles rows and then splits rows directly.

That is okay for quick use, but not ideal for paired experiments, because the pair variants can get split across train/val/test.

For serious use, you should split by pair_id, not by row.

So here is the corrected version of the splitting logic you should really use.

Pair-safe split helper

Replace the splitting logic with this:

def split_by_pair_id(rows, train_frac=0.67, val_frac=0.17, test_frac=0.16):
    grouped = {}
    for row in rows:
        grouped.setdefault(row["pair_id"], []).append(row)

    pair_ids = list(grouped.keys())
    random.shuffle(pair_ids)

    n = len(pair_ids)
    n_train = int(n * train_frac)
    n_val = int(n * val_frac)
    n_test = n - n_train - n_val

    train_ids = set(pair_ids[:n_train])
    val_ids = set(pair_ids[n_train:n_train+n_val])
    test_ids = set(pair_ids[n_train+n_val:])

    train_rows, val_rows, test_rows = [], [], []

    for pid, group in grouped.items():
        if pid in train_ids:
            train_rows.extend(group)
        elif pid in val_ids:
            val_rows.extend(group)
        else:
            test_rows.extend(group)

    return train_rows, val_rows, test_rows


Then use:

c_train, c_val, c_test = split_by_pair_id(chunk_rows)
e_train, e_val, e_test = split_by_pair_id(extr_rows)
i_train, i_val, i_test = split_by_pair_id(inter_rows)


That is the proper way.

What this enables statistically

With pair_id, you can later compute:

Chunking

Per-pair delta:

Δi​=Acc(chunkedi​)−Acc(flati​)
Interactivity

Per-pair penalty:

Δi​=Acc(lowi​)−Acc(highi​)
Extraneous load

Per-pair distractor sensitivity:

Δi(confusable)​=Acc(irrelevanti​)−Acc(confusablei​)

and

Δi(contradictory)​=Acc(irrelevanti​)−Acc(contradictoryi​)

This is far stronger than aggregate condition differences.

Strong next step

Now the natural next artifact is:

a paired evaluator + paired stats script

That script would:

preserve per-item correctness
group by pair_id
compute paired deltas
bootstrap paired confidence intervals
optionally run McNemar tests for matched binary outcomes

That is the statistically serious version of this project.

If you want, I can write that next: a paired-evaluation and paired-effects analysis pipeline for the v2 datasets.
