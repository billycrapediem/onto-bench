# Golden ontology sources

Where to find ground-truth ("golden") ontologies for onto-bench. Two candidates, one confirmed and one potential.

## 1. Burr (HPI-Information-Systems) — has the golden ontology

- Repo: https://github.com/HPI-Information-Systems/Burr
- What it is: a benchmark for tools that derive ontologies from relational databases. It compares Ontop, RDB2Onto and an LLM-based approach. The associated paper is published via ACM.
- Where the golden ontology lives: each scenario folder holds two files.
  - a **SQL file**: database schema and sample instance data.
  - a **mapping file**: the ground-truth database-to-ontology mapping, stored as JSON and translated to D2RQ at evaluation time.
- Scenario sets:
  - `micro_benchmark/`: synthetic, isolated scenarios that probe specific behaviours.
  - `real-world/`: 46 scenarios from real database-to-ontology mappings, including proprietary SAP data.
- Evaluation code: `evaluator/`.
- Caveats:
  - There is no single central ontology file. The gold standard is defined per scenario through the JSON mappings.
  - No license is visible on the repo page. Check before redistributing any of the data.
  - Some real-world data is proprietary (SAP). Verify what is actually included in the repo.

## 2. SCOPE / SCION (arXiv 2607.21610) — potentially has a golden ontology

- Paper: https://arxiv.org/pdf/2607.21610 ("SCOPE and SCION: A Benchmark and an Auditable Reference Pipeline for Schema Induction and Fusion from Text", Hu et al.)
- Code and benchmark packages: https://github.com/wandugu/paper_scion
- Why it may help: SCOPE is built from 24 public information-extraction sources (15 relation-extraction, 9 event-extraction), normalized into "evaluation-only gold schema graphs". These are gold schemas, not necessarily ontologies in the OWL/RDF sense.
- Status: not verified. Confirm the gold schema graphs are in the released benchmark package, and check their format, before treating them as golden ontologies.
- License: the paper carries CC BY 4.0. The license of the code and data repo is not yet confirmed.

## Open questions

- Do Burr's JSON mappings, once translated, give a complete enough ontology to use as the target, or do we need to build ontology files from them?
- Are the SCOPE gold schema graphs close enough to ontologies (classes, relations, hierarchy) to be used alongside Burr?
