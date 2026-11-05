# Environment synthesis landscape

Where research on synthesizing environments for agents stands, with a focus on data agents and enterprise-style data. Written 2026-11-05.

**Status: not verified.** Everything below comes from web search summaries. The sources themselves could not be fetched, because this session's network policy blocks arxiv.org and most paper sites. Numbers and method details may be wrong. Read the paper before citing any of them.

## Closest prior work to onto-bench

[research-direction.md](research-direction.md) says nobody has built synthetic environments for messy data lakes. That is no longer true. These 2025–2026 papers are close:

| Work | What it does | Overlap with onto-bench |
|---|---|---|
| **Era by Eon** ([arXiv 2609.09853](https://arxiv.org/pdf/2609.09853)) | Generates a whole company "estate": many systems that share one entity graph. Computes answers exactly from the generated records. Injects controlled dirty values and labeled unanswerable questions. A companion paper on hidden knowledge is [arXiv 2609.30055](https://arxiv.org/abs/2609.30055). | Very high: multi-source enterprise lake, exact ground truth, noise injection |
| **Data World** (Meituan LongCat, [arXiv 2610.21656](https://arxiv.org/abs/2610.21656)) | Defines the analytical mechanism (explicit causal rules) first, then generates messy heterogeneous files. Environments, tasks and rubrics all come from that mechanism, and agents are scored against a hidden record. Meant for training and evaluation. Ships a 100-task benchmark. | Very high: "hidden truth first, data second", the same structure as ontology → tables |
| **AgentMercury** ([arXiv 2608.20634](https://arxiv.org/pdf/2608.20634)) | 4,326 synthetic company environments and 43,300 investigation tasks. A program checks the environment state; no LLM judge. | High: enterprise scale, verifiable tasks |
| **CORGI** / "Agent Bain vs. Agent McKinsey" ([arXiv 2510.07309](https://arxiv.org/pdf/2510.07309v1)) | Synthetic business databases built in stages (schema design, data simulation, multi-agent evaluation). Reported about 21% harder than BIRD. | Medium: synthetic business databases, text-to-SQL scope |

**Where onto-bench can still differ:**

- **A real world as the seed.** These works generate their worlds from scratch. onto-bench starts from real databases with a real gold ontology mapping (Burr's NPD and Mondial), so the as-is split is real data, not synthetic.
- **Grading tied to the ontology.** Each task follows a path through the ontology, so process scores (which tables, which joins) come from the mapping. This is not proven new; check how Era by Eon and Data World score the steps an agent takes.
- **Paired perturbations.** Variants that change the surface but not the meaning, and variants that change the meaning but not the surface (see [research-framing.md](research-framing.md)). These test understanding, not only how much noise an agent tolerates.

**To do:** read Era by Eon and Data World in full before deciding Q29 (the core contribution).

## Trends

### 1. Environments as training data for agentic RL

Reinforcement learning (RL) needs interaction that can be checked at a scale hand-built benchmarks can't supply, so the environment itself has become the thing being synthesized.

- **Agent World Model** (Snowflake, ICML 2026, [arXiv 2602.10090](https://arxiv.org/abs/2602.10090)):
  - Pipeline: scenario → task → database schema and data → tool interface (MCP) → verification code.
  - Output: 1,000 environments backed by SQL databases, with about 35 tools each.
  - Reward: database state compared before and after the agent acts, plus an LLM judge.
  - Result: Qwen3 models trained only on these environments generalized to BFCLv3, τ²-bench and MCP-Universe.
  - Code: `Snowflake-Labs/agent-world-model`.
- **EnvScaler** (RUC, Findings of ACL 2026, [arXiv 2601.05808](https://arxiv.org/abs/2601.05808)):
  - Builds environment skeletons through topic mining and logic modeling, with an automatic quality check.
  - Then writes scenarios with rule-based checks on each trajectory.
  - Output: 191 environments and about 7K scenarios, used for supervised fine-tuning and RL.
- **AgentScaler** (Tongyi/Alibaba, [arXiv 2509.13311](https://arxiv.org/abs/2509.13311)):
  - Builds read-write database environments from tool collections.
  - Keeps a trajectory only when the final database state matches the gold state.
  - A 4B model reportedly performs comparably to 30B models on τ-bench.
- Related, seen only in listings: CodeGym, AutoForge, ScaleEnv, CuES. Overview: [Environment Scaling for Agentic RL](https://jxzhangjhu.github.io/blog/2026/environment-scaling-for-agentic-rl/) (blog, 2026).

### 2. Specification first, data second

Fix the ground truth before generating anything, so correctness holds by construction instead of being checked by an LLM afterwards.

- **AutoGym** (NeurIPS 2026, [arXiv 2609.22592](https://arxiv.org/pdf/2609.22592)): writes a blueprint of the solution space and the checks before building the environment.
- **Data World**: as above, causal rules first.
- **RingSQL** ([arXiv 2601.05451](https://arxiv.org/abs/2601.05451)): SQL templates that work on any schema give correct ground truth; an LLM only rewrites the wording of each question.

An ontology plus mapping fits this pattern: it is the specification the data comes from.

### 3. Environments run on code and databases, not simulated by an LLM

- [arXiv 2605.07247](https://papers.cool/arxiv/2605.07247) documents LLM-simulated environments that hallucinate, contradict themselves and drift silently in state, which corrupts the reward signal.
- Agent World Model makes the same argument for code-driven environments.

This supports onto-bench's choice of real SQL databases in a sandbox.

### 4. Text-to-SQL data synthesis: from volume to control

| Pipeline | Approach | Note |
|---|---|---|
| **OmniSQL** ([arXiv 2503.02240](https://arxiv.org/abs/2503.02240)) | SynSQL-2.5M: about 2.5M samples (database, SQL, question, reasoning steps) over 16k+ LLM-synthesized SQLite databases seeded from web tables | The main precedent for synthesizing schemas, but the databases are small and clean |
| **EvolSQL** ([arXiv 2601.04875](https://arxiv.org/abs/2601.04875)) | Evolves seed SQL with small edits to its syntax tree, raising complexity step by step | Reportedly beats SynSQL-2.5M using about 1/18 of the data |
| **SING-SQL** ([arXiv 2509.25672](https://arxiv.org/html/2509.25672v1)) | Tailors data to one target database. Splits the schema into parts, generates SQL at several difficulty levels, repairs it, and re-generates for under-covered columns | Coverage balancing carries over to task generation on large schemas |
| **RingSQL** ([arXiv 2601.05451](https://arxiv.org/abs/2601.05451)) | Templates for ground truth, LLM for wording. 5k pairs over 160 databases | Ground truth by construction |
| **Business-logic-driven synthesis** ([arXiv 2601.14518](https://arxiv.org/html/2601.14518v1)) | Generates evaluation data for business-intelligence questions on private enterprise databases, driven by business logic rather than schema templates | Top models reportedly reach about 43% on the hardest tier. Close to onto-bench's "business definitions" axis |

Takeaway: how hard and varied the data is, and how well that can be controlled, now matters more than raw volume. All of these produce question–SQL pairs, not multi-step environments for agents.

### 5. Enterprise realism is where models fail

| Benchmark | Setting | Reported top score |
|---|---|---|
| Spider 1.0 / BIRD | Academic, clean | ~91% / ~73–77% |
| **Spider 2.0** (ICLR 2025, [arXiv 2411.07763](https://arxiv.org/abs/2411.07763)) | 632 tasks on real BigQuery and Snowflake databases, often over 1,000 columns | 21.3% (o1-preview agent) |
| **BIRD-Ent / Spider-Ent** (EMNLP 2026 Findings, [paper](https://aclanthology.org/2026.findings-emnlp.1971.pdf)) | Enterprise-scale rebuilds of BIRD and Spider | 39.1% / 60.5% |
| **BEAVER** | Private enterprise warehouses | Near 0% (as cited by arXiv 2604.25149) |
| **KramaBench** (ICLR 2026, [arXiv 2506.06541](https://arxiv.org/html/2506.06541v1)) | Data lake: ~1,700 files from 24 sources | ~50–55% end to end |

Semantic knowledge as part of the environment:

- **EntSQL** ([arXiv 2606.03363](https://arxiv.org/html/2606.03363)) pairs each database with a long enterprise document of private business knowledge, such as metric definitions and fiscal-year rules. It has 1,066 examples across 5 domains, drawn from real user Q&A logs.
- The **semantic-layer study** ([arXiv 2604.25149](https://arxiv.org/pdf/2604.25149)) measures how adding a semantic layer changes accuracy and hallucination.
- **DAComp** ([arXiv 2512.04324](https://arxiv.org/pdf/2512.04324)) has 210 tasks covering the full data lifecycle, including building semantic layers.

For onto-bench, the semantic layer (data dictionary, column comments, business definitions) can be a dial: absent, partial, stale or complete.

### 6. Rewards for each step, not just the final answer

- **SQL-ASTRA** (ACL 2026 Findings, [arXiv 2603.16161](https://arxiv.org/pdf/2603.16161)): rewards each step's execution result and combines them over the whole trajectory. Reportedly about 5% over a binary reward on BIRD.
- **CAPER** ([arXiv 2606.03327](https://arxiv.org/abs/2606.03327)): a reward model for each clause, trained by flipping parts of the SQL syntax tree.
- **Reward-SQL** ([arXiv 2505.04671](https://arxiv.org/abs/2505.04671)): process supervision for text-to-SQL.
- **EnvScaler**: rule-based checks on each trajectory.

These works score SQL steps or tool calls. None found scores whether the agent understood what the data means, which is what onto-bench's ontology paths could do.

### 7. Evaluation only vs training

- **AgenticDataBench** ([arXiv 2607.01647](https://arxiv.org/abs/2607.01647)): 344 tasks generated from a skill hierarchy and refined by experts, each with its own evaluation function. Evaluation only.
- **DAB** ([arXiv 2603.20576](https://arxiv.org/abs/2603.20576)): 54 enterprise queries with ground truth built by hand. Evaluation only.
- **DARE-bench** (ICLR 2026, [arXiv 2602.24288](https://arxiv.org/abs/2602.24288)): about 6.3K tasks with execution-checked rewards, for RL training.

Generating environments at scale is what makes a training use possible, which bears on the open "training use" question in research-direction.md.

## Open gaps

- **Checking realism is the weakest area.**
  - The 2026 survey "Agentic Environment Engineering for LLMs" ([arXiv 2606.12191](https://arxiv.org/pdf/2606.12191)) groups the limits of synthetic environments under correctness, difficulty, diversity and fidelity. It names the failure modes: inconsistencies that teach the wrong lessons, overfitting to templates, and simulators that miss real behavior.
  - Few papers check synthetic results against real ones. One exception tested 31 LLM user simulators against 451 real people on τ-bench ([arXiv 2603.11245](https://arxiv.org/abs/2603.11245)).
  - onto-bench's two splits (real data as-is, and real data with logged perturbations) could serve as that check. A strong method should be a gap between scores on perturbed and real data that shrinks as perturbations are calibrated.
- **Shortcuts left by generation.** BIRD-Ent warns that patterns left over from the generation process can make tasks easy to game. Generated noise may need its own checks.
- **Transfer to real data agents is unproven.** Evidence that training on synthetic environments transfers to real ones comes from tool use (Agent World Model) and web agents (Fara-1.5, [arXiv 2606.20785](https://arxiv.org/abs/2606.20785)). None was found for data lakes. Data World says real-world transfer is still open.
- **Few environments model meaning, as opposed to messiness.** Most pipelines add mess (dirty values, more tables, stale docs). Fewer control semantic distance, meaning naming, join hops, business definitions and how values are encoded. That is onto-bench's angle.

## Next steps

1. Allow arxiv.org (and huggingface.co, aclanthology.org) in the session's network settings, then re-run the research with claim checking.
2. Read Era by Eon, Data World and AgentMercury in full. Compare their ground-truth construction, noise taxonomy and grading against the onto-bench design.
3. Update the "nobody has done this" claim under "Why it's worth doing" in [research-direction.md](research-direction.md).
4. Use the survey's four limits (correctness, difficulty, diversity, fidelity) as the checklist for how onto-bench validates its environments.
