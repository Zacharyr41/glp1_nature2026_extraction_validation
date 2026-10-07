# Architecture — Part I (extraction validation)

This repository is **Part I** of the code behind *Computable longitudinal patient journeys from
structured and unstructured EHR data* (Kim, Foty et al., *Nature Medicine* 2026,
doi:10.1038/s41591-026-04695-x). It contains the **evaluation** of the authors' LLM extraction
platform, not the platform itself. Part II (the GLP-1 outcome analysis) is the sibling repository
`glp1_nature2026_a1c_analysis`.

## 1. Where this repo sits in the paper's system

The paper describes a five-layer platform. Only the shaded layer is released here; the rest is
proprietary (see the Code availability statement) or lives in Part II.

```mermaid
flowchart LR
    subgraph platform["RespondHealth platform (proprietary, NOT released)"]
        A[De-identified EHR<br/>Sidus Insights / Harris<br/>35M patients] --> B[Structured data<br/>standardise / clean]
        A --> C[Visit notes]
        C --> D["Agentic LLM extraction<br/>(medication agent, phenotype agent, ...)<br/>prompts/phenotype_prompt.txt (fragment)"]
        D --> E["Atomic-statement verification<br/>0-3 support score<br/>prompts/verification_prompt.txt (fragment)"]
        E --> F[Ontology grounding<br/>UMLS / HPO / RxNorm / ICD / LOINC / SNOMED]
        F --> G["Concept discovery<br/>embeddings + k-means + LLM labels<br/>output: clustering/cluster_hierarchy.json"]
        B --> H[(Patient-centred<br/>knowledge graph)]
        F --> H
        H --> I[Agentic analytic layer<br/>cohort builder, variable derivation]
    end
    I -->|"one wide CSV per study<br/>(step8g_...csv)"| P2["Part II repo<br/>glp1_nature2026_a1c_analysis"]
    D -.->|model outputs + physician labels| P1
    subgraph P1["THIS REPO — Part I validation (released)"]
        X1[experiment2.py<br/>phenotype physician review] 
        X2[experiment3_med.py<br/>medication physician review]
        X3[experiment3.py<br/>GSC+ HPO benchmark]
        X4[experiment4.py<br/>HPO prevalence delta plots]
        AG[agreement.py<br/>kappa / PABAK]
        X1 --> AG
        X2 --> AG
    end
```

## 2. Inside this repo

Everything is batch: `code/run.sh` runs the unit tests and four experiment scripts, each reading
frozen inputs under `data/` and writing a text log or PNG under `results/`. There is no shared
Python package other than `code/agreement.py`; each experiment is a standalone script with
hard-coded relative paths (`os.path.join(script_dir, '..', 'data')`).

```mermaid
flowchart TB
    subgraph inputs["data/ (frozen evaluation inputs)"]
        MD["MD1.json, MD2.json, MD3.json<br/>physician phenotype reviews"]
        MDm["MD1_med.json, MD2_med.json, MD3_med.json<br/>physician medication reviews"]
        MAS["med_aggregate_stats.json<br/>pre-computed medication precision counts"]
        GOLD["GSCplus_test_gold.tsv<br/>BiolarkGSC+ gold (206 abstracts, 1,949 HPO annotations)"]
        OBO["hp.obo<br/>HPO 2025-05-06"]
        PRED["base-*.json (6), sft-*.json (3)<br/>model phenotype predictions on GSC+"]
        TL["hpo_timeline_prevalence.json<br/>HPO prevalence per window, 15k patients"]
    end

    subgraph code["code/"]
        AG[agreement.py]
        E2[experiment2.py]
        E3M[experiment3_med.py]
        E3[experiment3.py]
        E4[experiment4.py]
        T[test_experiment3.py]
        RUN[run.sh]
    end

    subgraph results["results/"]
        R2[experiment2_results.txt<br/>Tables 1, 2, 2b, 3]
        R3M[experiment3_med_results.txt]
        R3[experiment3_results.txt<br/>macro/micro P R F1 per model]
        R4[experiment4*.txt + 4 PNGs]
        RT[tests_results.txt]
    end

    MD --> E2 --> R2
    MDm --> E3M --> R3M
    MAS --> E3M
    GOLD --> E3 --> R3
    OBO --> E3
    PRED --> E3
    OBO --> E4
    TL --> E4 --> R4
    AG --> E2
    AG --> E3M
    E3 --> T --> RT
    RUN -.runs in order.-> T
    RUN -.-> E2
    RUN -.-> E3M
    RUN -.-> E3
    RUN -.-> E4

    subgraph unused["Not read by any script (reference material)"]
        ADJ["adjudication/*.csv, adjudication_notes.md"]
        CL["clustering/cluster_hierarchy.json"]
        PR["prompts/*.txt"]
    end
```

### Execution order (`code/run.sh`)

| Step | Command | Output | Paper item |
|---|---|---|---|
| 1 | `python3 -m unittest discover -s . -p 'test_*.py'` | `results/tests_results.txt` | — |
| 2 | `python3 experiment2.py` | `results/experiment2_results.txt` | Table 1 (phenotype rows), Extended Data Table 1 (phenotype κ) |
| 3 | `python3 experiment3_med.py` | `results/experiment3_med_results.txt` | Table 1 (medication rows), Extended Data Table 1 (medication κ) |
| 4 | `python3 experiment3.py` | `results/experiment3_results.txt` | Extended Data Table 5 (BiolarkGSC+) |
| 5 | `python3 experiment4.py --json ... --data_dir ../data --output_dir ../results` | `results/experiment4_results.txt`, two PNGs | Exploratory phenotype-prevalence figure (not a numbered display item) |
| 6 | same with `--no-normalize --suffix experiment4_unnormalized` | sensitivity figures | — |

`run.sh` uses `set -euo pipefail`, records each step's exit code, echoes the last 20 lines of a
failing log to stderr, keeps running the remaining steps, and exits non-zero at the end if any
step failed.

## 3. Design properties worth knowing

- **Reproducible offline.** No network, no model calls, no PHI. All LLM outputs were generated
  upstream and are frozen in `data/`. Running `run.sh` reproduces every number in `results/`.
- **Scoring is independent of ontology assignment.** In `experiment3.py` the gold HPO ID drives
  synonym and hierarchy expansion; the model's own `hpo_matched_name` is only an extra surface
  string. Extraction quality and normalisation quality are therefore separately measurable.
- **One agreement estimator.** `agreement.py` is the only kappa implementation. It reports Cohen's
  κ and PABAK side by side with p_o and p_e and flags degenerate pairs, so the medication table's
  κ = 0.000 at 95.8 % agreement is explained rather than hidden.
- **Minimal dependencies.** Only `experiment4.py` needs third-party packages (`numpy`,
  `matplotlib`). The other three experiments and the tests are standard-library Python.
- **No configuration layer.** Paths, model lists, reviewer file names, source keys, and display
  names are literals inside each script. See `REUSE_GUIDE.md` for the exact edit points.

See `CODE_REFERENCE.md` for function-level documentation, `DATA_SCHEMA.md` for every input and
output format, `PAPER_COVERAGE.md` for what the paper reports that is and is not reproducible
here, and `REUSE_GUIDE.md` for how to run the same validation on another clinical question or on
MIMIC.
