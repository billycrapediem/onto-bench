# onto-bench: research direction

_Working notes from the grilling session, 2026-10-26. Status: in progress._

## The idea

Treat an ontology as the ground truth behind a data lake, and use it to generate and grade tasks for data agents. The ontology says what every table means and how the tables connect, which gives us:

- **Controlled distance** between what the user asks and where the data actually lives.
- **Process grading**: we know which tables a task needs, so we can check whether the agent accessed the right ones.
- **Trajectory analysis**: we can see where in the pipeline the agent went wrong, not just whether the final answer is right.

Two research questions sit under this:

1. **Environment generation**: making schemas and tables more complex and noisy.
2. **Task generation**: testing whether the agent truly understands the meaning of the data, which it can only do by observing the data.

The original idea also named data synthesis and training as possible uses, beyond benchmarking.

### Why it's worth doing

- **Process-level ground truth comes automatically.** Existing benchmarks either grade only the final answer or annotate steps by hand (DataClaw, InsightBench). Here, the ontology mapping says which tables each task needs.
- **Distance becomes measurable.** The gap between the question and where the data lives can be varied and plotted against accuracy. That yields findings, not just a leaderboard.
- **Timing.** Synthetic environments for training agents are an active area (e.g. Agent World Model), but nobody has done this for messy data lakes.

### Main risks

- Being seen as "yet another benchmark". The process scores are the main defense.
- Realism: whether perturbed data looks like real mess.
- The same fact can live in several tables, so there can be several valid paths to one answer.
- Scope: the generator, the task compiler and the grader are three separate systems.

## Decisions so far

| # | Topic | Decision |
|---|---|---|
| Q1 | Main contribution | A benchmark with a leaderboard |
| Q2 | What's under test | Multi-step data agents in a logged sandbox (schema browsing, SQL, maybe Python). Single-shot text-to-SQL appears only as a baseline |
| Q3 | Source of the world | Real databases that come with a gold database-to-ontology mapping |
| Q4 | Lake scope (v1) | SQL-only relational tables from multiple sources, 50–300 tables, partial metadata (column comments, a partly stale data dictionary). No unstructured documents |
| Q6 | Leaderboard | Final-answer accuracy plus process scores, with cost (steps, tokens) as a secondary column |
| Q7 | Splits | Two: the real data as-is, and the real data with logged perturbations (noisier, more complex) |
| Q8 | Ground truth | The gold mapping run on the original real database. Errors already in the data count as part of the world. Questions sensitive to known errors are excluded |
| Q9 | Answer format | The agent submits a final result (a value or small table), matched against the gold result (order ignored unless asked, numeric tolerance). The agent's code feeds the process scores |
| Q10 | Harness | A shared reference harness for the main leaderboard, plus an open track where any agent system can enter if it uses the sandbox's tools |
| Q13 | Task form | Tasks are natural-language questions that test full-pipeline data analysis |
| Q23 | Data source | Burr is the main source. NPD uses Burr's mapping paired with the original NPD ontology file. Mondial's ontology graph is built from Burr's mappings. Email the Burr authors about the license |

## Task generation and grading (Q15–17)

- Questions are generated from the **topology of the ontology**, by following a path of links through the ontology graph.
- Each question breaks down into **subtasks** along that path. Each subtask's answer is one link on the path.
- Grading looks at:
  1. **Gold-file coverage**: did the agent access the tables that the path maps to?
  2. **Context understanding**: judged from the code the agent writes.
  3. **Final answer**: as in Q9.

## Open questions (high level)

- **Q29: Core contribution.** Task generation, environment generation, or both?
  - (a) Tasks only: questions over the real databases as they are.
  - (b) Environments only: messier versions of the databases with fixed questions.
  - (c) Both, with task generation as the core.
  - Recommendation: (c). Burr gives only two usable databases (116 tables, ~300k rows), and environment generation is what turns them into many varied lakes. Burr's micro-benchmark patterns could serve as a starting catalog of transformations. **Not yet decided.**
- **Headline finding:** what the benchmark should show. Depends on Q29.
- **Training use:** whether the generator also produces training data or stays a benchmark only. Depends on Q29.

## Data source (Q23, decided)

- NPD: Burr's mapping, paired with the ontology file from the original NPD benchmark.
- Mondial: build the ontology graph from Burr's mappings.
- **To do:** email the Burr authors to confirm the data can be redistributed.

**Facts (checked 2026-10-26):**

Burr's `real-world/` folder holds 4 databases:

| Database | Domain | Tables | Real rows | Usable |
|---|---|---|---|---|
| `npd_factpages` | Oil and gas (Norwegian continental shelf) | 70 | ~245k | Yes |
| `mondial` | World geography | 46 | ~55k | Yes |
| `iswc` | Conference data | 9 | ~96 | Too small |
| `rba` | SAP reference architecture | 7 | None (schema only) | No |

Burr also has a `micro_benchmark/` folder: 54 small scenarios with 1–6 tables each (116 tables in total), schema only, no rows. Each scenario isolates one schema pattern, such as n:m tables, class hierarchies, normalization levels, cryptic attribute names or missing keys. The whole repo is about 59 MB.

- Burr's repo has **no license file**. Last push May 2026. The paper (SIGMOD 2026) is CC BY 4.0, but that covers the paper, not the data.
- Burr has **no central ontology file**. The ground truth lives in the mappings:
  - **NPD** (`map_d2rq.ttl`): a D2RQ mapping with 80 class maps and 787 property bridges. It uses the original NPD vocabulary (`npdv:` = `http://sws.ifi.uio.no/vocab/npd-v2#`), so it lines up directly with the original NPD ontology file.
  - **Mondial** (`mappings/*.json`, 20 files): each file lists classes, object properties with the SQL join that implements them, and data properties with their columns. This already forms the graph: classes are nodes and object properties are edges, each with a known join. One concept can map to several tables (Country has 3 class maps).
- **How Burr's ground truth was made** (from the Burr paper, SIGMOD 2026, and the original NPD paper, ISWC 2013):
  - **Mondial:** the ontology is the existing Mondial ontology. The mapping from database to ontology was **created by hand by the Burr authors**, who also added the missing foreign keys by hand. 46 tables map to about 30 concepts, so many tables are not concepts at all: dimension tables, n:m tables, and one column that holds two concepts (mountain or volcano, depending on a type column).
  - **NPD:** the ontology and D2RQ mapping come from the original authors (Skjæveland, Lian, Horrocks, University of Oslo). They were **semi-automatic**: an initial ontology and mapping were generated from the CSV files and schema (roughly one class per table), then extended by hand with axioms, mostly general superclasses. Burr cleaned them up. The gap between concepts and tables is therefore smaller than in Mondial. Most of the difficulty comes from size (71 tables, about 1,000 columns) and from concepts spread over several tables.
  - **ISWC:** a hand-written example mapping from the D2RQ authors (Bizer and Seaborne).
  - **SAP enterprise:** the most expert-built. A modeling expert interviewed domain experts, built the domain model, then wrote the mapping (8 denormalized tables to 20 concepts and 68 relations). No rows are public.
  - **Micro-benchmark:** scenarios created by hand by the Burr authors.
- The [original NPD benchmark](https://github.com/ontop/npd-benchmark) has the same data plus an OWL 2 QL ontology and R2RML mappings, under Apache-2.0. Unmaintained since 2020.
- [ACME Insurance](https://github.com/datadotworld/cwd-benchmark-data) (data.world) has an OWL ontology, R2RML mappings and 43 human-written questions, under Apache-2.0. Its data is toy-sized (dozens of rows).
- No real source found has multiple systems that share entities. The largest real mapped database is NPD at ~70 tables. As-is lakes will therefore be single-source with 46–70 tables, below the Q4 target.
- Correction to [golden-ontology-sources.md](golden-ontology-sources.md): it says Burr has "46 real-world scenarios". The repo actually has 4 real-world databases. The 46 is likely Mondial's table count.

## Other sources checked

**SCOPE / SCION** ([arXiv 2607.21610](https://arxiv.org/abs/2607.21610), ICML 2026, [repo](https://github.com/wandugu/paper_scion)), checked 2026-10-26. Not a fit as a data source.

- **Task:** induce a schema graph from raw text. There are no tables, no SQL and no mappings to databases.
- **Gold schemas:** flat lists of typed edges drawn from 24 information-extraction datasets (15 relation, 9 event; English and Chinese).
  - 507 relation edges (81 entity types, 415 relation types). 153 of them are untyped `Entity → Entity`.
  - 1,036 event-role edges (253 event types).
  - No class hierarchy, no attributes or columns, no constraints. Per-source schemas are tiny: conll04 has 3 entity types and 5 relations.
- **Instance data:** sentences annotated with relation mentions (e.g. "Bethany Located_In Conn."). There is no entity resolution and there are no numeric attributes, so the data can't support aggregation-style analysis questions.
- **License:** code Apache-2.0, paper CC BY 4.0. Each underlying dataset keeps its own license.
- **Possible use:** related work on ontology induction, not a source of ground truth.

**Other datasets with a semantic layer over tables** (surveyed 2026-10-26):

| Dataset | Semantic layer | Data | Fit for onto-bench |
|---|---|---|---|
| [ACME Insurance](https://github.com/datadotworld/cwd-benchmark-data) + [dbt version](https://github.com/dbt-labs/semantic-layer-llm-benchmarking) | OWL + R2RML; dbt semantic models (metrics, dimensions) | Toy-sized, ~13 tables | Validation slice: business domain, human-written questions |
| [RODI](https://www.cs.ox.ac.uk/isg/tools/RODI/) | OWL + R2RML | Conference (small, synthetic) + Mondial + NPD | Its conference scenarios map one ontology onto several schema designs: real examples of how the same data gets laid out differently |
| [Spider4SPARQL](https://github.com/ckosten/spider4sparql) | Ontop-bootstrapped OWL + mappings for 166 Spider DBs (20 in repo) | Small Spider DBs, ~9.7k human questions | Broad (138 domains), but the ontology mirrors the schema, so there is no gap between concepts and tables |
| [BIRD](https://bird-bench.github.io/) | Per-column descriptions and value meanings; per-question evidence | 95 DBs, 37 domains | Could be lifted into ontologies to add business domains |
| [LiveSQLBench](https://benchmarklist.com/benchmarks/livesqlbench/) | Hierarchical knowledge base (definitions, formulas, dependencies) and column meanings | ~20 DBs per release, up to 1,340+ columns | Same as BIRD; its knowledge base is close to derived concepts |
| [Contoso semantic-layer study](https://arxiv.org/abs/2604.25149) | 4 KB hand-written markdown (measures, conventions) | Contoso retail in ClickHouse, 100 questions | Release not stated |
| [AtScale leaderboard](https://atscale.com/?p=24977) | AtScale semantic model | TPC-DS (synthetic) | Synthetic data |
| [Spider 2.0-DBT](https://blog.jetbrains.com/databao/2026/02/how-databao-agent-ranked-1-spider-2-0-dbt/) | dbt project YAML and model docs | Real dbt projects | Tasks build models, not answer questions |
| [SOTAB](https://paperswithcode.com/dataset/wdc-sotab), SemTab, GitTables | Column types and properties from schema.org, Wikidata or DBpedia | Single web tables | Poor: no multi-table databases, no joins |

**Conclusion:** none beats Burr for real multi-table data plus a real ontology plus a mapping. If two domains prove too narrow, the most realistic way to add business domains is to lift BIRD or LiveSQLBench databases into ontologies using their human-written annotations.

## Set aside for now

- Team, timeline and venue (ACL or ICML were mentioned).
- Details of the split design and the perturbation catalog.
- Question difficulty. The proposed knobs were the number of hops, the number of hops the question doesn't mention, and the final operation (lookup, filter, aggregate, compare). Deferred until the high level is settled.
- Topics raised but not pursued: unanswerable questions, the method for generating natural-language questions, a schema-only baseline check, insight discovery (planted insights), contamination, building multi-source lakes, a glossary visible to the agent, answers from model memory.

## Related work

- [Sequeda et al. 2023](https://arxiv.org/abs/2311.07509v1) and [Allemang & Sequeda 2024](https://arxiv.org/html/2405.11706v1): adding an ontology layer raised GPT-4 accuracy on an insurance database from 16% to 54%.
- [LiveSQLBench](https://benchmarklist.com/benchmarks/livesqlbench/): a hand-curated hierarchical knowledge base. Top models score about 30–43%.
- [DataClaw (2026)](https://awesomepapers.io/papers/zhang2026dataclaw): process-level evaluation of data agents, using milestones annotated by hand.
- [InsightBench (ICLR 2025)](https://arxiv.org/abs/2407.06423v3): insights planted by hand in business datasets.
- [TACO (2026)](https://arxiv.org/pdf/2606.14201): text-to-SQL at data-lake scale (about 13k tables).
- Synthetic text-to-SQL pipelines ([OmniSQL](https://huggingface.co/papers/2503.02240), [EvolSQL](https://arxiv.org/pdf/2601.04875), [RingSQL](https://arxiv.org/abs/2601.05451v1), [SING-SQL](https://arxiv.org/html/2509.25672v1)): difficulty comes mostly from SQL complexity.
- [Agent World Model (Snowflake, 2026)](https://arxiv.org/pdf/2602.10090v2): synthetic database-backed environments for agent reinforcement learning. They are tool-use apps, not data lakes.

**Positioning:** none of these derive process-level ground truth automatically from an ontology. That is the gap onto-bench targets.
