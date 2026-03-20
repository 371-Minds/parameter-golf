> **Status: Reference — exploratory experiment list**
> This document is a raw chat export containing a broad list of 10 candidate experiments (E1–E10). It is not the primary build spec. The **Tier 1 build targets** are the 4 benchmarks defined in [Synthetic dataset spec.md](Synthetic%20dataset%20spec.md). For the focused 7-experiment research design, see [Concrete experimental matrix.md](Concrete%20experimental%20matrix.md).

---

Experiments to test whether tiny LLMs show human-like “cognitive overload”

Yes — here’s a concrete way to operationalize your idea instead of leaving it metaphorical.

The goal is not to prove LLMs have human cognition.
The goal is to test whether they exhibit load-sensitive performance breakdowns with the same structural signature as human cognitive load:

more simultaneous constraints → worse performance
irrelevant information hurts performance
chunking/schema support helps performance
sequencing complexity helps
overload produces specific failure modes, not just uniform degradation

That is testable.

Core hypothesis

A tiny LLM behaves like a bounded-capacity processing system:

Performance↓ as active dependency load↑

and

Performance↓ faster when extraneous load competes with task-relevant structure

while

Performance↑ when information is chunked, scaffolded, or curriculum-ordered

This is basically a translational version of cognitive load theory.

Experimental framework

You want to isolate three kinds of load:

Intrinsic load: actual task complexity
Extraneous load: irrelevant or poorly structured information
Schema support / germane load: features that make abstraction easier

Then measure how tiny models fail.

Recommended setup

Use multiple model sizes so you can observe scaling of overload effects:

very tiny: 1M–10M params
small: 10M–50M
medium baseline: 100M+ if feasible

For parameter-golf relevance, tiny models matter most.

Use the same:

tokenizer
optimizer
train steps
eval format

Then build synthetic tasks where complexity is precisely controlled.

Experiment 1: Constraint stacking
Question

Do models degrade nonlinearly as more simultaneous constraints must be satisfied?

Task

Prompt the model with outputs that must satisfy increasing numbers of conditions.

Example task:

Output a 6-character string that:

starts with B
ends with 7
contains exactly two vowels
has no repeated characters
the third character must be K
characters 2 and 5 must be lowercase
must include one digit
vowels cannot be adjacent

Vary number of constraints from 1 to 8.

Measures
exact constraint satisfaction rate
first-error position
probability mass on valid outputs
type of failure:
omission
contradiction
recency bias
local compliance / global failure
Prediction

Tiny models will:

do fine at low constraint counts
show threshold-like collapse after a small number
disproportionately satisfy recent or salient constraints
fail globally while preserving local fluency

This mirrors human overload under multi-rule tasks.

Experiment 2: Intrinsic vs extraneous load separation
Question

Does irrelevant but plausible information impair performance more than equivalent relevant information?

Task

Give a reasoning task with matched length but different informational utility.

Condition A: clean

Sarah has 3 red boxes and 2 blue boxes. She gives away 1 blue box. How many blue boxes remain?

Condition B: extraneous

Sarah has 3 red boxes and 2 blue boxes. The boxes were bought in April. Her cousin likes triangles. One red box has a sticker. She gives away 1 blue box. How many blue boxes remain?

Control for token count.

Measures
answer accuracy
logprob on correct answer
latency if measured
attention distribution if available
degradation as distractor count increases
Prediction

Tiny models should suffer disproportionately from distractors, especially semantically adjacent distractors.

That is a model analogue of extraneous load.

Experiment 3: Chunking benefit
Question

Do models perform better when information is grouped into meaningful schemas rather than presented atomistically?

Task

Same underlying task, two representations.

Unchunked

A, D, F, K are in group 1. B, E, J are in group 2. C, G, H, I are in group 3...

Chunked

Group 1: A D F K
Group 2: B E J
Group 3: C G H I

Or for sequence tasks:

Unchunked

1 0 0 1 1 0 1 0 0 1 1 1

Chunked

1001-1010-0111

Then ask recall / transformation / classification questions.

Measures
accuracy
token-level perplexity
robustness under increasing sequence length
Prediction

Performance improves significantly with chunked structure, especially in tiny models.

That parallels human schema/chunking effects.

Experiment 4: Curriculum vs no curriculum
Question

Does ordered exposure reduce overload and improve abstraction in tiny models?

Training conditions

Train identical tiny models on:

randomized examples
easy-to-hard curriculum
short-context to long-context
high-frequency patterns first
Tasks

Synthetic grammar or algorithmic tasks:

bracket matching
copy with noise
simple symbolic substitution
short arithmetic forms
nested patterns
Measures
final task accuracy
learning speed
generalization to harder held-out tasks
sample efficiency
Prediction

Tiny models benefit more from curriculum than larger ones because curriculum reduces early optimization overload.

That is directly CLT-aligned.

Experiment 5: Split-attention interference
Question

Do multiple simultaneously relevant information sources cause more interference than serial presentation?

Task

Present two relevant tables or rule sets either:

side by side / interleaved
one first, then one second in structured order

Then ask a query requiring integration.

Example:

Rule set A defines symbol meanings
Rule set B defines action mappings
Query requires using both
Conditions
serial structured
interleaved mixed
mixed plus distractors
Measures
integration accuracy
failure type:
uses only one table
substitutes one rule for another
recency-weighted answer
Prediction

Tiny models will fail more under interleaving than serial presentation even when information content is identical.

That resembles split-attention effects in human learning.

Experiment 6: Context window overload curve
Question

Does increasing context create overload even before formal context-limit truncation?

Task

Put the answer-relevant information at fixed positions and vary total context length by adding:

irrelevant text
weakly relevant text
competing similar facts

Then ask the same question.

Measures
accuracy vs context length
effect of relevant fact position
primacy/recency bias
interference from semantically similar distractors
Prediction

Tiny models degrade well before hard context exhaustion.
They especially fail when distractors are semantically confusable.

That’s a strong indicator of capacity competition, not just truncation.

Experiment 7: Element interactivity

This is one of the strongest CLT concepts.

Question

Does performance drop more for tasks requiring interaction among many elements than for tasks with the same number of elements but low interdependence?

Task

Compare:

Low interactivity

Recall 8 independent facts.

High interactivity

Use 8 facts whose relationships jointly determine the answer.

Example:

low: “Which city was paired with label X?”
high: “Which city satisfies all 4 interacting constraints?”

Same total info volume, different dependency structure.

Measures
accuracy
confidence gap
error decomposition
Prediction

High element interactivity causes much larger drops than low interactivity at equal length.

That would strongly support a CLT-style interpretation.

Experiment 8: Schema induction transfer
Question

Once a tiny model learns an abstract pattern, does performance rebound on structurally similar tasks?

Task

Train/evaluate on synthetic families:

mapping symbols via hidden rules
hierarchical rewrite rules
simple formal grammars

Then test:

same structure, new surface symbols
same symbols, new structure
both changed
Measures
transfer accuracy
compression/perplexity gains
robustness to relabeling
Prediction

If “germane load” is meaningful in LLMs, once schemas form, performance improves dramatically on structurally homologous tasks.

This separates rote fitting from schema-building.

Experiment 9: Noise placement sensitivity
Question

Does the location of noise matter in a way analogous to attentional disruption?

Task

Inject the same amount of irrelevant material:

before relevant information
in the middle
after relevant information
interleaved within it
Measures
answer accuracy
degradation by placement
recall of task-critical facts
Prediction

Interleaved noise hurts more than front-loaded or end-loaded noise.

That mirrors disruption of coherent encoding.

Experiment 10: Compression as overload proxy

Since you’re thinking about parameter golf, this matters a lot.

Question

Does overload show up as worse compression on strings with high dependency interaction, not just high entropy?

Task

Construct evaluation sets with matched token frequencies but different dependency complexity:

simple repetitive text
local syntax text
long-range dependency text
nested / cross-referential text
distractor-heavy but answer-simple text
Measures
bits per byte
next-token accuracy by dependency distance
calibration degradation
Prediction

Tiny models will show disproportionate compression loss on high-interactivity text, not just rare-token text.

That’s exactly what a CLT-style theory would predict.

Failure taxonomy to capture

Do not just record “wrong.”
Record how wrong.

I’d classify errors into:

recency override: latest constraint dominates
saliency override: vivid token pattern dominates
partial compliance: local correctness, global failure
constraint dropping: one or more requirements disappear
interference substitution: distractor inserted into answer logic
schema failure: fails to generalize known structure
coordination collapse: individually known facts fail when combined

This taxonomy is critical.
Human cognitive overload also has patterned failure modes, not random failure.

Best synthetic benchmarks for this

If you want high control, use synthetic tasks first:

Dyck languages / bracket matching
symbol rewriting
table lookup with distractors
constrained sequence generation
finite-state transition tasks
nested dependency completion
relational reasoning with controlled variables

Then validate on naturalistic tasks:

reading comprehension with distractors
instruction following with multiple constraints
noisy summarization
structured extraction from cluttered text
Analysis strategy
1. Plot overload curves

For each model size, plot performance against:

number of constraints
distractor count
dependency depth
context length
interactivity score

You want to see whether degradation is:

linear
thresholded
model-size-dependent
2. Compare matched-information conditions

Same token count, different structure:

relevant vs irrelevant
chunked vs unchunked
low vs high interactivity

This is where CLT-style claims get teeth.

3. Fit a simple load model

You can model performance as:

P=σ(α+β1​R−β2​I−β3​X−β4​D+β5​S)
Where:
R
= model capacity
I
= intrinsic complexity
X
= extraneous load
D
= dependency interaction
S

= schema support

You don’t need this to be “true” in a deep sense — it’s a useful empirical abstraction.

Strongest evidence that would support your theory

I’d consider your thesis strongly supported if you observe all of these:

1. Nonlinear collapse with added simultaneous constraints

Not just gradual decline — threshold behavior.

2. Equal-length irrelevant info hurts more than equal-length relevant info helps

This cleanly separates extraneous load.

3. Chunking/scaffolding improves performance at constant content

This suggests schema-sensitive load management.

4. High element interactivity hurts more than simple length

This is one of the most important CLT signatures.

5. Curriculum especially benefits tiny models

Shows overload is partly optimization-structural.

6. Failure modes are systematic

Recency bias, omitted constraints, interference substitutions.

If you get those six, your idea becomes very compelling.

What would falsify or weaken it?

Your theory would weaken if:

degradation is explained entirely by token length
irrelevant info hurts no more than relevant info of equal length
chunking gives no consistent benefit
interactivity doesn’t matter beyond size
larger and smaller models fail in the same shape
error modes are random rather than patterned

Then “cognitive load” would just be poetic language for context length effects.

Parameter golf version of the hypothesis

This is the version I’d use for that challenge:

In tiny language models, benchmark performance is dominated not only by parameter count but by how efficiently training and architecture minimize extraneous representational load and accelerate schema formation under strict capacity limits.

That’s a strong, researchable claim.

A minimal publishable experiment set

If you wanted to make this into a serious note/paper, I’d do just 4 core experiments:

Constraint stacking
Relevant vs extraneous distractors
Chunked vs unchunked representation
High vs low element interactivity

Across 3 model sizes.

That’s enough to establish the central phenomenon.

My interpretation

I think you’re likely right in the structural sense.

Not:

“LLMs think like humans”

But:

“Both humans and tiny LLMs are bounded learners whose performance depends heavily on how complexity is organized, chunked, and interfered with.”

That’s a much stronger and more defensible claim.

And honestly, if this holds, it could become a design principle:

Small-model training should be treated as cognitive load engineering.

Which is a pretty powerful frame.

If you want, I can next turn this into either:

a concrete experimental matrix with datasets, metrics, and prompts, or
a short research note / hypothesis memo you could share with others
