# Research framing

What onto-bench measures, whether that ability matters, and what prior work says. Written 2026-10-26.

## The ability we evaluate

**Semantic grounding**: can an agent work out what data means when that meaning is not written down, and use it to answer a user's question?

Setup: ontology (hidden truth) → rendered into tables plus noise → the agent sees the tables and a question → answer. To answer correctly, the agent has to recover the part of the ontology the question touches.

Most benchmarks grade only the final answer. Holding the ontology lets us also grade the agent's "map": which concepts it identified, which tables it tied them to, and which constraints it respected.

Sub-abilities, each measurable on its own:

1. **Linking**: user word → concept → table, column or value.
2. **Exploration**: what the agent looks at to resolve ambiguity, and how efficiently.
3. **Reading meaning from values**: inferring formats, units and codes when names don't reveal them.
4. **Respecting structure**: cardinality, duplicates, soft-deleted rows, fan-out on joins.
5. **Abstention**: recognising an ambiguous question or one the data can't answer.

"Semantic distance" between user intent and data is a set of separate dials: naming, join hops, definitions (business terms built from several conditions) and values (encodings, units).

## Revised hypothesis

The first version, "data agents fail mainly by misunderstanding what data means, not by writing bad queries", is only half supported. The revised version:

> Strong agents usually locate the right data but misread its meaning at the value and definition level (formats, units, keys, business definitions), and these errors are silent. No benchmark controls this semantic distance, so we don't know how much of the failure it explains.

The bottleneck has moved. Earlier models (around 2023) failed to *find* the right schema. Recent agents (2025–2026) mostly find it, then misread values or compute the wrong thing.

### Evidence for

- **BIRD** (Li et al., NeurIPS 2023), an analysis of 500 ChatGPT errors:
  - 41.6% wrong schema linking
  - 40.8% misunderstanding database content
  - 17.6% misreading the hint knowledge

  https://arxiv.org/pdf/2305.03111
- **PV-SQL** (2026): database misinterpretation causes 41.3% of errors on BIRD, and question misinterpretation 24.8%. https://arxiv.org/pdf/2604.17653
- **Sequeda, Allemang & Jacob** (2023): on a real insurance schema, GPT-4 zero-shot scores 16% on SQL and 54% with an OWL ontology plus mappings. This is the closest prior work to our oracle-ontology ablation. Its limit is scale: one schema, 43 questions. https://arxiv.org/abs/2311.07509
- **Allemang & Sequeda, "Ontologies to the Rescue!"** (2024): ontology-based checking and repair of generated queries reaches 72%, including 8% "I don't know". https://arxiv.org/abs/2405.11706
- **BEAVER** (2024): LLMs do poorly on real enterprise data warehouses (99 tables / 1,544 columns, and 366 tables / 2,708 columns). https://arxiv.org/abs/2409.02038

### Evidence against, or nuance

- **The Death of Schema Linking?** (Maamari et al., NeurIPS 2024 TRL workshop): strong models use the right schema elements even among many irrelevant ones, as long as the schema fits in context. https://arxiv.org/abs/2408.07702
- **DAB** (Ma, Shankar et al., 2026), a classification of wrong answers:
  - 15% wrong data selection
  - 40% wrong plan
  - 45% wrong implementation

  The authors write that agents "generally identify the correct tables and columns". However, many implementation errors are misread values: a regex picks part of an ISBN as a year, or "MALE" matches inside "FEMALE". DAB also deliberately injects reformatted join keys and labels buried in free text. https://arxiv.org/abs/2603.20576
- **KramaBench** (ICLR 2026): the best system scores 55% end to end and 62% with perfect retrieval. Models identify up to 42% of the needed data tasks but fully implement only 20%. https://arxiv.org/abs/2506.06541
- **SANA** (2026): data analysis is the persistent bottleneck. Search matters mainly at large data-lake scale (LakeQA), much less in KramaBench. https://arxiv.org/abs/2606.13904

### Experiment that tests the hypothesis

**Oracle-ontology ablation**: run agents with and without the gold ontology (or the relevant slice) in context, across semantic-distance and noise levels.

- **Large gap that grows with distance:** grounding is the bottleneck.
- **Small gap:** failures come from elsewhere (planning, computation), and the framing must change.

## Testing "understanding" with paired perturbations

Because we know which changes affect meaning, we can build two kinds of variant from one ontology:

- **Surface changes, meaning stays the same** (rename columns, split or merge tables, denormalize): the answer **should not** change.
- **Meaning changes, surface stays the same** (dollars → cents while the column is still `amount`, remapped status codes): the answer **should** change.

An agent that understands passes both. A name-matching agent fails both.

A task counts as "observation-necessary" only if an agent that sees just the schema fails it and the same agent with data access passes.

## Should the benchmark be discovery-only?

Here "discovery" means locating the relevant data.

**No, not as the main contribution.** DAB (15% data-selection errors), KramaBench (+7 points from perfect retrieval) and SANA all suggest that finding data is not where strong agents fail.

Discovery tasks are still worth keeping:
- **At very large scale:** thousands of tables, too large to fit in context.
- **At high naming distance:** but then they test meaning more than search.
- **As a diagnostic layer:** they separate "failed to find" from "failed to use". DAB found top agents spend about 20% of tool calls exploring, and both over- and under-exploring hurt.

Plan: discovery is level 1 of a layered benchmark. The levels are discovery → grounding → correct answer → insight.

## Insight-discovery tasks

### Prior work

- **InsightBench** (Sahu et al., ICLR 2025): 100 synthetic business datasets with annotator-planted insights. https://arxiv.org/abs/2407.06423
- **InsightEval** (ACL Findings 2026) found errors in InsightBench data and goals, and showed that recall-only scoring ignores wrong insights. It adds precision, F1 and novelty. It does not test dirty data or spurious patterns. https://arxiv.org/abs/2511.22884
- **DiscoveryBench** (Majumder et al., ICLR 2025): 264 real tasks and 903 synthetic ones. The best system scores 25%. https://arxiv.org/abs/2407.01725
- **HypoBench** (2025): synthetic hypothesis discovery. 93.8% at base difficulty, 38.8% as difficulty rises. https://arxiv.org/abs/2504.11524
- **DDR-Bench** (2026): open-ended "deep data research", checklist-graded. https://arxiv.org/abs/2602.02039
- **DataGovBench** (2026): table QA and table insight tasks on government open data. https://arxiv.org/abs/2607.06482
- **DAComp** (2025): data agents across the full lifecycle, including misleading insights from wrong formulas. https://arxiv.org/abs/2512.04324
- **CausalDS** (2026) and **Ice Cream Doesn't Cause Drowning** (2025): statistical traps (confounding, Simpson's paradox, selection bias). https://arxiv.org/abs/2607.08093, https://arxiv.org/abs/2505.13770
- **Top-k Insights** (Tang et al., SIGMOD 2017) and **QuickInsights** (Ding et al., SIGMOD 2019): formal insight types (trend, outlier, attribution, correlation and others), useful as a planting taxonomy. https://www.microsoft.com/en-us/research/publication/extracting-top-k-insights-multi-dimensional-data/

**Gap:** no insight benchmark found so far combines planted insights with semantic noise. In ours, true insights appear only under correct reading of the data, and false ones appear under a misreading.

### Generation recipe

1. **Generate data from the ontology with a simple causal model.** Each attribute is produced from its parents (e.g., region → price → churn), so the ground truth is known.
2. **Plant typed insights** using the SIGMOD taxonomy. Store each as a structured record: measure, aggregation, segment, time window, direction, size.
3. **Gate true insights behind correct reading of the data.** The signal appears only after converting units, removing duplicates, excluding test accounts or matching reformatted keys.
4. **Plant decoys:** false insights that appear only under a misreading. For example, test accounts create a fake spike, or fan-out on a one-to-many join inflates one segment.
5. **Include null datasets** with no planted effect, to measure fabrication.
6. **Give goals, not "find insights"** (e.g., "Why did Q3 revenue drop?").
7. **Grade by structured matching.** An LLM maps the agent's text into the record format, then fields are compared deterministically. Report:
   - true-insight recall
   - decoy rate
   - fabrication rate on null data

Headline result only this setup can produce: "X% of false insights trace back to a specific grounding error", since we know which decoy each false insight came from.

**Difficulty dials:** effect size versus noise, number of joins to reach the segment, amount of grounding required, number of decoys.

## Risks

- **Unrealistic noise** measures puzzle-solving, not real grounding. Calibrate noise types against real schema-to-ontology drift; Burr's 46 real-world scenarios can help (see `golden-ontology-sources.md`).
- **Trajectory scoring can punish valid alternative paths.** Use the ontology to define sets of equivalent sources.
- **Tasks that are too clean:** questions generated from ontology paths are tidier than real requests. Include some vague ones.
- **Memorization:** public ontologies may be in training data. Rename at the concept level.
- **Synthetic causal data can be too easy** (HypoBench reaches 93.8% at base difficulty). Effect size and noise must be dials.

## Open questions

- Where is the line between a "meaning" error and a "plan" or "implementation" error? DAB's examples mix the two. We need our own error taxonomy, decided before running experiments.
- Do the Burr mappings give enough structure (definitions, constraints) to build "meaning changes, surface stays the same" variants, or must we add them ourselves?
- Some numbers above come from abstracts or search summaries (PV-SQL, HypoBench, DAComp, SANA). Check them against the full papers before citing.
