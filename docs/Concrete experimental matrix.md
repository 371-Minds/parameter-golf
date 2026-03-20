Concrete experimental matrix: testing “cognitive load” in tiny LLMs

Great — let’s make this executable.

I’ll give you a compact but serious research design with:

hypotheses
experimental factors
datasets/tasks
prompt templates
metrics
failure taxonomy
expected outcomes
how this ties back to parameter golf
Study objective

Test whether tiny LLMs exhibit load-sensitive breakdowns analogous to human cognitive load effects, specifically:

performance degradation under increasing simultaneous constraints
disproportionate harm from irrelevant information
benefit from chunking/scaffolding
benefit from curriculum
stronger sensitivity in smaller models than larger ones
High-level design
Independent variables
Model size

At minimum:

XS: ~1M–5M params
S: ~10M–30M params
M: ~60M–150M params

If compute is limited:

just do two sizes: tiny and small
Load manipulations
Constraint count
Distractor count
Element interactivity
Representation format: chunked vs unchunked
Curriculum condition
Context length
Noise placement
Dependent variables
exact accuracy
constraint satisfaction rate
bits per byte / perplexity
token-level error position
failure type
calibration/confidence if available
Master experiment table
Experiment	Main concept	IVs	DVs	Key comparison
E1	Constraint stacking	# constraints	exact pass, partial pass	low vs high simultaneous demands
E2	Extraneous load	distractor count/type	accuracy, logprob	relevant vs irrelevant info
E3	Chunking	chunked vs flat	accuracy, perplexity	same content, different organization
E4	Element interactivity	dependency coupling	accuracy, failure type	same length, different relational complexity
E5	Curriculum	curriculum vs shuffled	learning speed, final score	scaffolded learning vs raw exposure
E6	Context overload	total context length	accuracy by position	overload before context limit
E7	Noise placement	before/middle/interleaved/after	accuracy	disruption by placement
E1: Constraint stacking
Hypothesis

Tiny models will show threshold-like failure as the number of simultaneously active constraints increases.

Task type

Constrained generation or constrained classification.

Synthetic dataset design

Generate examples where the model must output a string satisfying N rules.

Example prompt template
Produce a 6-character code.

Rules:
1. The first character must be B.
2. The last character must be 7.
3. The third character must be K.
4. Include exactly one vowel.
5. Do not repeat any character.
6. Include exactly one digit.
7. The vowel cannot be next to the digit.
8. Character 2 must be lowercase.

Output only the code.


Create matched datasets for:

1 rule
2 rules
3 rules
…
8 rules
Controlled factors
output length fixed
alphabet fixed
rule syntax fixed
number of examples per level: e.g. 500–2,000
Metrics
exact full validity
per-constraint satisfaction
omission rate
contradiction rate
recency-weighted rule compliance
Failure taxonomy
drops earliest rule
drops middle rule
obeys only recent rule
local validity / global invalidity
repeated character despite explicit ban
Expected result

Tiny models:

maintain some rule adherence initially
then sharply lose global consistency as rules accumulate
preserve surface fluency longer than global correctness
E2: Extraneous load
Hypothesis

Irrelevant but plausible details harm tiny models more than equally long relevant details help them.

Task type

Simple QA / symbolic reasoning / reading comprehension.

Dataset conditions

For each base example, create four versions:

Clean
Relevant elaboration
Irrelevant distractor
Confusable distractor (semantically adjacent)
Base example
Mira has 4 blue cards and 3 green cards. She gives away 2 blue cards.
How many blue cards remain?

Irrelevant distractor
Mira has 4 blue cards and 3 green cards. She bought the cards on Tuesday. 
Her cousin likes marbles. One green card is shiny. She gives away 2 blue cards.
How many blue cards remain?

Confusable distractor
Mira has 4 blue cards and 3 green cards. She gives away 2 green cards earlier in the day.
Later, she gives away 2 blue cards.
How many blue cards remain?

Factors
distractor count: 0, 2, 4, 8
distractor type: irrelevant vs confusable
answer-relevant fact position
Metrics
exact accuracy
answer logprob
confusion type
attention mass if available
degradation slope by distractor count
Expected result

Confusable distractors should hurt much more than irrelevant fluff.
Tiny models should show stronger degradation slopes.

E3: Chunking vs unchunked presentation
Hypothesis

Chunked organization reduces effective load and improves performance even when content is unchanged.

Task type

Recall, classification, transformation, table lookup.

Example dataset

Map symbols to groups.

Flat condition
A in group 1. D in group 1. F in group 1. K in group 1.
B in group 2. E in group 2. J in group 2.
C in group 3. G in group 3. H in group 3. I in group 3.

Question: Which group is F in?

Chunked condition
Group 1: A, D, F, K
Group 2: B, E, J
Group 3: C, G, H, I

Question: Which group is F in?


Or for transformation:

flat list vs visually grouped sequence blocks
Factors
number of elements
query position
grouping strength
Metrics
accuracy
perplexity
robustness as list size grows
retrieval latency if available
Expected result

Chunked formats especially help tiny models.

E4: Element interactivity
Hypothesis

Performance drops more from dependency interaction than from raw information count.

Key idea

Keep total number of facts constant while varying whether they interact.

Low-interactivity example
Red maps to circle.
Blue maps to square.
Green maps to triangle.
Yellow maps to star.

Question: What does green map to?

High-interactivity example
Red maps to the shape that is not square.
Blue maps to the shape used when green is inactive.
Green is active only if yellow is not star.
Yellow is star only when blue is square.

Question: Which color maps to triangle?


Or relational tables:

independent lookup vs multi-rule integration
Factors
total facts fixed
dependency degree: 1, 2, 3, 4-way interaction
Metrics
exact accuracy
first reasoning error
whether answer used subset of rules only
performance drop vs interactivity level
Expected result

High-interactivity tasks will hurt more than equivalent-length low-interactivity tasks.
This is one of the strongest CLT-style signals.

E5: Curriculum vs shuffled training
Hypothesis

Curriculum reduces overload during learning and disproportionately benefits tiny models.

Training tasks

Use synthetic tasks with controllable difficulty:

bracket matching
copy with distractors
sequence transformation
lookup with increasing distractors
arithmetic with increasing carry depth
Training conditions
Shuffled
Easy → hard
Short context → long context
Low interactivity → high interactivity
Example curriculum ladder

For bracket matching:

depth 1
depth 2
depth 3
depth 4

For constrained generation:

1 rule
2 rules
3 rules
4 rules
Metrics
training loss trajectory
final held-out generalization
sample efficiency
transfer to harder unseen variants
Expected result

Tiny models should gain:

faster early learning
higher final task accuracy
better compositional generalization
E6: Context overload curve
Hypothesis

Tiny models overload before hitting formal context limits, especially under distractor-rich contexts.

Task type

Needle-in-haystack with controlled semantic clutter.

Conditions

Insert answer-relevant fact in a fixed location. Add:

random irrelevant text
semantically similar distractors
contradictory distractors
repeated but irrelevant pattern text
Example
[long context...]
The access code for Vault R is 48291.
[long context...]

Question: What is the access code for Vault R?


Vary:

total context length
distractor similarity
answer position
Metrics
exact retrieval accuracy
logprob on correct answer
primacy/recency effect
false recall rate
Expected result

Performance decays before hard truncation and decays faster when distractors are similar.

E7: Noise placement sensitivity
Hypothesis

Interleaving noise with relevant information disrupts more than placing it before or after.

Dataset variants

Same information, same total token count.

Noise before
Noise after
Noise in middle
Noise interleaved clause-by-clause
Example

Relevant:

Nora has 5 orange keys. She gives away 2 orange keys.
How many orange keys remain?


Interleaved:

Nora has 5 orange keys. Her desk is wooden. She gives away 2 orange keys.
Her friend owns boots. How many orange keys remain?

Metrics
accuracy
confusion rate
degradation by placement
Expected result

Interleaved > middle > before/after in harmfulness.

Model matrix

A good practical matrix:

Model	Params	Purpose
Tiny-A	3M	extreme bounded-capacity regime
Tiny-B	15M	parameter-golf relevant
Small-C	60M	comparison for scaling effects

If you can afford one more:

Medium-D: 120M–300M for shape comparison
Dataset size recommendations

For each experiment:

train/eval synthetic generation: 5k–50k examples depending on task
eval-only probing set: 1k–5k examples per condition
use held-out random seeds for generation

For publishable stability:

at least 3 seeds per model
ideally 5 seeds
Prompt templates
Constraint stacking prompt
You must produce an output satisfying all rules.

Rules:
{rules}

Output only the answer.

Distractor prompt
Read the information carefully and answer the question.

{context}

Answer with only the final answer.

Chunking prompt
Study the information below.

{formatted_context}

Question: {query}
Answer only with the answer.

Interactivity prompt
Use all relevant rules below.

{rule_set}

Question: {query}
Answer only with the answer.


Keep prompts minimal to avoid prompt-format variance becoming its own confound.

Metrics in detail
1. Exact accuracy

Binary: answer correct or not.

2. Constraint satisfaction score

For constrained generation:

\text{CSS} = \frac{\text{# satisfied constraints}}{\text{# total constraints}}

This is crucial because tiny models may partially comply.

3. Overload slope

For condition level k:

Δk​=P(k)−P(k+1)

Look for sharp negative transitions.

4. Area under load curve

For performance across load levels:

AULC=k=1∑K​P(k)

Higher is better.

5. Interference penalty

Compare clean vs distractor:

IP=Pclean​−Pdistractor​
6. Chunking gain

Compare chunked vs flat:

CG=Pchunked​−Pflat​
7. Interactivity penalty

Compare low vs high interactivity at equal token count:

EP=Plow-int​−Phigh-int​
8. Compression metric

For next-token tasks:

bits per byte
perplexity
token accuracy by dependency distance
Error annotation schema

For each wrong response, assign one or more labels:

constraint_omission
recency_override
saliency_override
distractor_substitution
partial_schema_use
global_consistency_failure
independent_fact_success_but_integration_failure
position_bias
format_confusion

This matters because the shape of failure is your strongest evidence.

Statistical analysis
Recommended tests
mixed-effects logistic regression for accuracy
ANOVA on condition effects if simplified
bootstrap confidence intervals for overload curves
pairwise comparisons with correction
Example regression
accuracy ~ model_size + constraint_count + distractor_type + interactivity
         + chunking + curriculum + model_size:constraint_count
         + model_size:distractor_type + (1|seed)

Main interactions to look for
smaller models × higher constraints
smaller models × confusable distractors
smaller models × high interactivity
smaller models × chunking benefit
smaller models × curriculum benefit

Those interactions are where your theory lives.

Expected signature pattern

If your hypothesis is right, you should see:

Tiny models
steep overload curves
strong distractor sensitivity
big chunking gains
big curriculum gains
large interactivity penalties
systematic recency/omission errors
Larger models
flatter overload curves
smaller chunking gain
reduced distractor sensitivity
more graceful degradation

That differential scaling is critical.

Connection to parameter golf

This experiment suite can feed directly into architecture/training design.

If tiny models are load-sensitive in these ways, then parameter-golf optimization should prioritize:

minimizing distractor/interference sensitivity
maximizing schema formation early
structuring training by complexity
choosing architectures that reduce coordination burden
evaluating not just aggregate loss, but overload resilience

In other words:

the best tiny models may be the ones that are best engineered against representational overload.

That’s your real bridge from theory to competition.

Minimal viable version

If you want the fastest path to signal, run just this:

Models
3M
15M
60M
Experiments
E1 constraint stacking
E2 distractors
E3 chunking
E4 element interactivity
Metrics
accuracy
constraint satisfaction
failure type labels

That alone is enough to get a very meaningful result.

Strong version

If you want the full research-grade suite:

add E5 curriculum
add E6 context overload
add E7 noise placement
run 3–5 seeds each
fit regression + overload curves
include compression metrics
Suggested writeup structure

If you later turn this into a memo or paper:

Title

Do Tiny Language Models Exhibit Cognitive-Load-Like Failure Modes?

Sections
Motivation
Load-theoretic framework
Experimental design
Results by manipulation
Failure-mode analysis
Implications for small-model training
Relevance to parameter-efficient language modeling
My recommendation

If you’re serious, I would start with this exact sequence:

Constraint stacking
Distractor sensitivity
Chunking benefit
Element interactivity

Because together they test:

overload by quantity
overload by irrelevance
relief by organization
overload by dependency coupling

That’s the cleanest proof set.

If you want, next I can do either:

generate a synthetic dataset spec for all 4 core experiments, or
write the actual Python benchmark harness for these experiments
