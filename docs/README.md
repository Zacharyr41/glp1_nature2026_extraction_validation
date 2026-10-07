# Documentation index — Part I (extraction validation)

| Document | Answers |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | How this repo fits in the paper's platform; Mermaid diagrams of the data flow with file references; execution order |
| [CODE_REFERENCE.md](CODE_REFERENCE.md) | Function-level documentation of every script, metric definitions, matching algorithm, line references |
| [DATA_SCHEMA.md](DATA_SCHEMA.md) | Every input and output file format, field by field |
| [PAPER_COVERAGE.md](PAPER_COVERAGE.md) | Which paper tables and figures reproduce here, which need Part II, which are not released; modularity assessment |
| [REUSE_GUIDE.md](REUSE_GUIDE.md) | How to run the same validation on a different clinical question or on MIMIC, and what you must write yourself |

Companion repository: `glp1_nature2026_a1c_analysis` (Part II, the GLP-1 outcome pipeline) has
the same five documents under `docs/`, including the MIMIC guide for the outcome analysis.

Quick facts:

- Python 3.12+, `pip install -r code/requirements.txt`, `bash code/run.sh`.
- Tests: `python3 -m unittest discover -s code -p 'test_*.py' -v` (10 tests, no third-party deps).
- Fully reproducible offline: Table 1 (model comparison block), Extended Data Table 1, Extended Data Table 5.
