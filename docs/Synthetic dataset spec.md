Synthetic dataset spec for the 4 core experiments

Perfect. I’ll define this so you could hand it to an engineer or use it yourself to generate data consistently.

The goal is to create fully controlled synthetic benchmarks where:

complexity is tunable
confounds are minimized
each experiment isolates one load variable
outputs are auto-gradable

I’ll structure each benchmark with:

purpose
task format
generation schema
example instances
labels / grading
difficulty knobs
anti-cheat controls
recommended splits
Global dataset principles

Use these rules across all 4 datasets:

1. Keep language simple and templated

You want model failure due to load, not prose ambiguity.

2. Auto-grade everything

Each example should have:

prompt
target
metadata
difficulty
condition
3. Separate surface variation from structural variation

Use paraphrase templates, but don’t let wording become the main challenge.

4. Hold out compositional structures, not just lexical items

Otherwise the model may memorize pattern shells.

5. Generate enough examples per condition

Recommended:

train: 20k–100k per task if training on them
eval: 1k–5k per condition
test: 1k+ per major difficulty band
Common JSONL schema

Use a shared record format:

{
  "id": "e1_train_000001",
  "experiment": "constraint_stacking",
  "condition": {
    "num_constraints": 4,
    "output_length": 6,
    "rule_types": ["prefix", "suffix", "exact_position", "uniqueness"]
  },
  "prompt": "Produce a 6-character code...\n",
  "target": "BkQm7a",
  "grader": {
    "type": "constraint_checker",
    "constraints": [...]
  },
  "metadata": {
    "seed": 123,
    "split": "train"
  }
}


You can vary the grader.type by dataset.

Dataset 1: Constraint Stacking Benchmark
Purpose

Measure breakdown as the number of simultaneously active constraints increases.

Task format

The model must generate a short string satisfying all listed rules.

Output domain

Use controlled alphabets:

uppercase letters
lowercase letters
digits

Example output lengths:

5
6
7
8

Keep length fixed within a sub-condition.

Rule families

Define a library of rule types. Example rule types:

Position rules
first character is B
last character is 7
third character is lowercase
second character is vowel
Count rules
exactly one digit
exactly two vowels
exactly three uppercase letters
Membership rules
must include K
must include one vowel
must not include X
Adjacency rules
no repeated adjacent characters
digit cannot be next to vowel
B must appear before 7
Uniqueness rules
all characters must be distinct
no character may appear more than twice
Class-pattern rules
pattern must be Uppercase-lowercase-digit-uppercase-lowercase
exactly one lowercase character
Condition design
Core variable

num_constraints = 1..8

Secondary difficulty knobs
output length
rule interaction level
whether rules are independent or coupled
Recommended condition grid
Condition	Output length	Constraints
C1	5	1
C2	5	2
C3	6	3
C4	6	4
C5	6	5
C6	7	6
C7	7	7
C8	8	8
Generation algorithm
Step 1

Sample a valid target string first.

Step 2

Derive constraints from that target string.

This is much better than sampling random rules and hoping they’re satisfiable.

Step 3

Ensure the chosen constraints are:

non-redundant
not contradictory
not trivially implied by one another
Step 4

Generate prompt from templates.

Prompt template variants
Template A
Produce a {length}-character code.

Rules:
1. {rule_1}
2. {rule_2}
3. {rule_3}

Output only the code.

Template B
Write one string of length {length} that satisfies all conditions:
- {rule_1}
- {rule_2}
- {rule_3}

Answer with only the string.

Example instance
{
  "id": "cs_test_001",
  "experiment": "constraint_stacking",
  "condition": {
    "num_constraints": 4,
    "output_length": 6
  },
  "prompt": "Produce a 6-character code.\n\nRules:\n1. The first character must be B.\n2. The last character must be 7.\n3. Include exactly one vowel.\n4. All characters must be distinct.\n\nOutput only the code.",
  "target": "BkTae7",
  "grader": {
    "type": "constraint_checker",
    "constraints": [
      {"type": "position", "index": 0, "value": "B"},
      {"type": "position", "index": 5, "value": "7"},
      {"type": "count_class", "class": "vowel", "count": 1},
      {"type": "all_distinct": true}
    ]
  }
}

Grading

Do not require exact match to target.
Require only constraint satisfaction.

This is important because multiple valid answers may exist.

Anti-cheat controls
vary prompt wording
vary target length
vary alphabet
ensure valid answers are not always unique
hold out rule combinations in test set
Recommended splits
train: rule combinations seen
val: seen rule types, unseen combinations
test: unseen combinations and slightly harder interaction patterns
Dataset 2: Extraneous Load Benchmark
Purpose

Measure effect of irrelevant and confusable distractors.

Task format

Simple question answering over short passages.

The answer should be deterministic and usually short:

integer
color
label
yes/no
item name
Base problem families

Use a few structurally simple families:

Family A: counting
“Lena has 5 blue marbles and gives away 2.”
Family B: lookup
“Box R contains the silver key.”
Family C: attribute retrieval
“The triangle is green.”
Family D: state update
“Door A starts closed, then opens.”

Keep the core reasoning easy so distractor effects are visible.

Condition types
Clean

Only relevant facts.

Irrelevant distractors

Extra facts unrelated to target.

Confusable distractors

Extra facts of same semantic type but irrelevant.

Contradictory distractors

Similar facts about different objects that tempt substitution.

Difficulty knobs
number of distractors: 0, 2, 4, 8
distractor type: irrelevant, confusable, contradictory-adjacent
location of relevant fact: early, middle, late
Example base item
Clean
Mira has 4 blue cards and 3 green cards. She gives away 2 blue cards.
How many blue cards remain?


Target: 2

Irrelevant
Mira has 4 blue cards and 3 green cards. She bought them on Tuesday. Her cousin likes marbles.
One green card is glossy. She gives away 2 blue cards.
How many blue cards remain?


Target: 2

Confusable
Mira has 4 blue cards and 3 green cards. Earlier, she gives away 2 green cards.
Later, she gives away 2 blue cards.
How many blue cards remain?


Target: 2

Generation schema
Step 1

Generate a clean base problem.

Step 2

Generate distractor candidates from typed templates:

temporal
irrelevant preference
object descriptions
same-domain but different-color/count/object
Step 3

Insert distractors into predefined positions.

Metadata fields
{
  "base_family": "counting",
  "distractor_count": 4,
  "distractor_type": "confusable",
  "relevant_fact_position": "middle"
}

Example JSONL
{
  "id": "el_test_001",
  "experiment": "extraneous_load",
  "condition": {
    "distractor_count": 4,
    "distractor_type": "confusable",
    "relevant_fact_position": "middle"
  },
  "prompt": "Mira has 4 blue cards and 3 green cards. Earlier, she gives away 2 green cards. One card is glossy. Later, she gives away 2 blue cards.\nHow many blue cards remain?\nAnswer with only the number.",
  "target": "2",
  "grader": {
    "type": "exact_match_normalized"
  }
}

Anti-cheat controls
randomize names, objects, colors, quantities
ensure distractors don’t accidentally determine answer
use balanced answer distributions
keep token length matched across conditions as much as possible
Recommended split design

Important: hold out some combinations like:

specific names
certain object-color pairs
some distractor templates

That way you test load sensitivity, not memorization.

Dataset 3: Chunking Benchmark
Purpose

Test whether grouped/structured presentation improves performance over flat presentation with identical content.

Task format

Present the same information in two formats:

flat
chunked

Then ask a retrieval, comparison, or classification question.

Information families
Family A: group membership

Symbols mapped to groups.

Family B: category tables

Items mapped to attributes.

Family C: sequential blocks

Sequence partitioned into meaningful units.

Family D: nested lists

Objects with sub-properties.

Example family: group membership
Flat
A is in Group 1. D is in Group 1. F is in Group 1.
B is in Group 2. E is in Group 2. J is in Group 2.
C is in Group 3. G is in Group 3. H is in Group 3.

Question: Which group is F in?

Chunked
Group 1: A, D, F
Group 2: B, E, J
Group 3: C, G, H

Question: Which group is F in?


Target: Group 1

Additional task variants
Comparison

“Are A and F in the same group?”

Set completion

“Name one item in Group 2.”

Attribute query

“Which item is both blue and square?”

Difficulty knobs
number of groups
items per group
number of attributes
query complexity
whether query requires one-hop retrieval or two-hop integration
Generation schema
Step 1

Sample a relational table:

items
groups
attributes
Step 2

Render into two equivalent surface forms:

flat sentences
chunked list/table-like block
Step 3

Generate matched questions.

Condition metadata
{
  "format": "chunked",
  "num_groups": 4,
  "items_per_group": 3,
  "query_type": "lookup"
}

Example JSONL
{
  "id": "ch_test_001",
  "experiment": "chunking",
  "condition": {
    "format": "flat",
    "num_groups": 3,
    "items_per_group": 3,
    "query_type": "lookup"
  },
  "prompt": "A is in Group 1. D is in Group 1. F is in Group 1. B is in Group 2. E is in Group 2. J is in Group 2. C is in Group 3. G is in Group 3. H is in Group 3.\nQuestion: Which group is F in?\nAnswer only with the group name.",
  "target": "Group 1",
  "grader": {
    "type": "exact_match_normalized"
  }
}

Critical control

Content must be exactly identical across flat vs chunked conditions.

Only presentation changes.

Anti-cheat controls
randomize labels
vary whether chunking is by line breaks, bullets, or headers
ensure answer priors are balanced
hold out some query types at test time
Recommended evaluation

Pair each flat example with a chunked twin.
This allows direct paired comparisons.

Dataset 4: Element Interactivity Benchmark
Purpose

Test whether performance depends more on dependency coupling than on raw amount of information.

This is probably the most conceptually important dataset.

Task format

Same approximate length and number of facts, but vary whether facts are independent or must be jointly integrated.

Two primary conditions
Low interactivity

Facts are independent lookups.

High interactivity

Facts interact; answer depends on combining them.

Family 1: symbol-rule mapping
Low interactivity example
Rin uses Circle.
Pax uses Square.
Tov uses Triangle.
Lem uses Star.

Question: Which shape does Tov use?


Target: Triangle

High interactivity example
Rin uses the shape that Pax does not use.
Pax uses Square if Tov is inactive.
Tov is active only when Lem does not use Star.
Lem uses Star.

Question: Which shape does Rin use?


Target determined by multi-rule integration.

Family 2: door/key state system
Low interactivity
Door A uses Key 2.
Door B uses Key 4.
Door C uses Key 1.
Door D uses Key 3.

Question: Which key opens Door C?

High interactivity
Door A uses the key that Door B does not use.
Door B uses Key 4 if Door C is locked.
Door C is locked when Door D uses Key 3.
Door D uses Key 3.

Question: Which key opens Door A?

Family 3: attribute conjunction
Low interactivity

Independent fact retrieval.

High interactivity

Need all constraints:

object that is blue
but only among those active on Tuesday
excluding items stored in box C
and preferring objects paired with triangle
Difficulty knobs
number of facts fixed: 4, 6, 8
interaction degree: 1-hop, 2-hop, 3-hop
branching factor
presence of negation
Generation strategy
Low interactivity generation

Create independent mapping table.

High interactivity generation

Start from a latent world state and derive relational rules that uniquely determine target answer.

Important:

ensure unique answer
avoid contradictions
keep syntax simple
use same token-length envelope as low-interactivity cases
Metadata
{
  "interactivity": "high",
  "num_facts": 4,
  "reasoning_hops": 3,
  "uses_negation": true
}

Example JSONL
{
  "id": "ei_test_001",
  "experiment": "element_interactivity",
  "condition": {
    "interactivity": "high",
    "num_facts": 4,
    "reasoning_hops": 2
  },
  "prompt": "Door A uses the key that Door B does not use. Door B uses Key 4 if Door C is locked. Door C is locked when Door D uses Key 3. Door D uses Key 3.\nQuestion: Which key opens Door A?\nAnswer only with the key name.",
  "target": "Key 1",
  "grader": {
    "type": "exact_match_normalized"
  }
}

Critical controls

You want:

same approximate token count
same number of entities
same answer vocabulary
only dependency structure changed

That’s what makes this a real interactivity test.

Unified split strategy

For all datasets, use three split types:

Train

Seen template families, seen rule families.

Validation

Seen primitives, unseen combinations.

Test-A

Unseen combinations of known primitives.

Test-B

Unseen lexical surface forms.

Test-C

Slightly extrapolated difficulty.

This lets you distinguish:

memorization
compositional generalization
overload robustness
Recommended dataset sizes
If training models on these tasks

Per experiment:

train: 50k
val: 5k
test: 5k
If only probing pretrained models

Per condition:

500–1,000 examples
total 5k–20k per experiment is enough
Evaluation file format

You may also want a separate grading reference file:

{
  "id": "cs_test_001",
  "accepted_answers": null,
  "constraint_program": {...},
  "metadata": {...}
}


For exact-match tasks:

{
  "id": "el_test_001",
  "accepted_answers": ["2", "two"]
}

Difficulty schedule across experiments

To compare across tasks, map each instance to a rough load score:

L=w1​C+w2​D+w3​I+w4​N
Where:
C
= number of constraints
D
= distractor count
I
= interaction degree
N

= normalized information length

Not because this is “the truth,” but because it helps organize analysis.

Suggested naming convention
cs = constraint stacking
el = extraneous load
ch = chunking
ei = element interactivity

Examples:

cs_train_000001
el_val_004382
ch_testA_000928
ei_testC_000114
Quality checks before use

Before running experiments, validate that:

Constraint stacking
all prompts are satisfiable
no redundant rule leakage
multiple valid outputs exist in many cases
Extraneous load
distractors do not accidentally alter answer
confusable distractors are balanced
answer distribution is uniform
Chunking
flat and chunked versions are semantically identical
token lengths are reasonably comparable
Element interactivity
low/high conditions are length-matched
high-interactivity examples require true integration
answers are uniquely determined
Strongest minimal benchmark pack

If you want the cleanest first version, build this:

Constraint stacking
lengths: 6
constraints: 1–6
1,000 eval examples each
Extraneous load
distractors: 0, 2, 4, 8
types: irrelevant, confusable
1,000 eval each
Chunking
flat vs chunked
lookup and comparison queries
2,000 paired examples
Element interactivity
low vs high
4 facts, 6 facts
1-hop vs 3-hop
1,000 each

That’s enough to get clear overload signatures.

What this benchmark suite should reveal

If your theory holds, you should see:

accuracy collapse faster with rising constraints in tiny models
confusable distractors hurt more than irrelevant fluff
chunked presentation systematically helps
interaction-heavy tasks are much harder than equal-length independent ones

That combination would be a very strong empirical basis for “LLM cognitive load” as a useful engineering concept.

Best next step

The logical next artifact would be one of these:

a formal benchmark spec document you can share with collaborators
Python data generators for all four datasets
an evaluation harness with graders and metrics
a small paper-style methodology section

If you want, I can do 2 next: write the Python generators for these four synthetic datasets.
