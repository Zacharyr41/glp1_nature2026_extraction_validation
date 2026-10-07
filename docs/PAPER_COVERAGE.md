# Paper coverage and modularity — Part I (extraction validation)

What the published paper reports, what this repository (plus Part II) lets you reproduce, and
what is not available anywhere.

## 1. Display-item map

| Paper item | What it shows | Reproducible from | Status |
|---|---|---|---|
| Fig. 1 | Cohort flow diagram (35.12M → 16,061 / 14,788) | Part II (`step0a`, step1 logs) | counts only; the figure script is not released |
| Table 1 (top) | GPT-5.2 Low vs physician reference: P/R/F1 per domain, adjudicated 20 visits and single reviewer 31–100 | — | **not reproducible**: the per-visit physician scoring of the 100-visit set is not in `data/`; only the 3-reviewer subset (MD1–3) is |
| Table 1 (bottom) | PhenoTagger / BioNER / BERT Med NER vs ours: N, precision, negation error | `experiment2.py` Table 2, `experiment3_med.py` Table 2 | **reproducible** (medication N comes from `med_aggregate_stats.json`) |
| Table 2 | Verification-score distribution over 1M visits; fine-tuning cohort; analysis cohort | — | not reproducible; produced by the proprietary pipeline |
| Table 3 | Baseline characteristics by glycemic stratum | Part II `step0_analysis_population_table.py` | code released, data not |
| Fig. 2a | UMAP + two-level taxonomy of medication-switch reasons | `clustering/cluster_hierarchy.json` (labels and counts) | taxonomy **output** released; embeddings, k-means, UMAP, LLM labelling code not released |
| Fig. 2b | Knowledge-graph node counts, structured vs +unstructured | — | not released |
| Fig. 2c | One patient's atomic-extraction timeline | — | not released (patient-level) |
| Fig. 3a–e | Weight / HbA1c trajectories, KM curves | Part II `conf_int/gap_120/step4_*`, `step8_*` | code released, data not |
| Fig. 3f | PHQ-9, pain, waist trajectories | Part II `add_unstructured/*` | code released, data not |
| Extended Data Fig. 1–3 | Platform schematics, delta-extraction example | — | illustrations; prompt fragments in `prompts/` show the mechanism |
| Extended Data Fig. 4 | LoRA training-loss curves | — | not released (Azure fine-tuning logs) |
| Extended Data Table 1 | Inter-rater κ, three physicians, per model | `experiment2.py` Table 1, `experiment3_med.py` Table 1 | **reproducible** (also adds PABAK and degeneracy flags the paper does not print) |
| Extended Data Table 2 | GEE + Cox estimates, full vs structured-only | Part II | code released, data not |
| Extended Data Table 3 | Model-predicted change at 3/6/9/12 months | Part II `step4_predictive_plots*` | code released, data not |
| Extended Data Table 4 | Note-derived assessments | Part II `add_unstructured/*` | code released, data not |
| Extended Data Table 5 | BiolarkGSC+ benchmark: base, SFT, PP, multi-agent | `experiment3.py` | **reproducible** for every LLM row; the "Encoders & early LLMs" rows are quoted from the literature |
| Supplementary Table 1 | Attrition across persistence definitions | Part II `step0a_samplesize_analysis.py` | code released, data not |
| Supplementary Note 2, Listings 1–2 | Extraction prompts | `prompts/phenotype_prompt.txt` | fragment only (delta extraction prompt is present; shared instruction constants are not) |

In short: **the three validation tables that rest on frozen labels (Table 1 bottom, Extended Data
Tables 1 and 5) reproduce exactly from this repository with no data access.** Everything that
touches patient-level EHR data has released code (Part II) but no data, and everything that is
the extraction platform itself has neither.

## 2. What the paper describes that has no code here

| Component (paper section) | Available? | Notes |
|---|---|---|
| Agent-assisted longitudinal note extraction (Methods, "Agent-assisted…") | No | Only `prompts/phenotype_prompt.txt` (one agent's prompt builder) |
| Visit-to-visit delta tracking: added / modified / removed | Prompt only | `_differential_prompt` in the phenotype prompt file |
| Atomic-statement verification, 0–3 support score | Prompt only | `prompts/verification_prompt.txt`; no scoring code |
| Ontology mapping (UMLS, SNOMED CT, HPO, ICD, LOINC, RxNorm) | No | `experiment3.py` has an HPO loader usable for label/synonym/hierarchy lookup only |
| Concept discovery (embeddings, k-means sweep, silhouette, LLM labelling, UMAP) | Output only | `clustering/cluster_hierarchy.json` |
| Knowledge-graph construction and storage | No | The graph schema is described in prose only |
| Agentic analytic environment (VS Code agents, research skills) | No | Its product is the wide CSV Part II consumes |
| Fine-tuning (Azure LoRA for GPT-4.1 Mini/Nano; Qwen 2.5 4B on A40s) | No | Hyper-parameters are in the Methods; outputs of the tuned models are in `data/sft-*.json` |
| Distillation label generation (GPT-5 Mini over 100k visits) | No | |
| Multi-model aggregation | Yes | Union in `experiment3.py` lines 408–447 |
| Task-based post-processing | Yes | `is_valid_phenotype` in `experiment3.py` |
| Physician review interface (Supplementary Note 1) | No | Its exports are `data/MD*.json` |
| Blind adjudication | Records only | `adjudication/` |
| Cohort construction, GEE, Cox, note-derived assessments | Part II | see the sibling repository |

## 3. Modularity assessment

How much of this repository can be lifted out and reused as a unit?

| Unit | Reusable as-is? | Coupling |
|---|---|---|
| `code/agreement.py` | **Yes.** Pure functions over `{item: {rater: label}}` with a 3-class encoding. | Only `N_CLASSES` and the Present/Absent/Rejected semantics are fixed. |
| HPO loader + ancestor closure (`experiment3.load_hpo`, `hpo_ancestors`, `resolve_hpo_id`) | **Yes**, with a copy-out. | Module-level globals populated at import; `HP:` prefix filter. |
| Three-layer matcher (`gold_expansion`, `check_match`, `_is_informative`) | **Yes**, for any ontology that supplies labels, synonyms and `is_a`. | Depends on the module globals above and on `GENERIC_TOKENS` / `FUNCTION_WORDS`. |
| Macro/micro scorer (`update_name_class_stats`, `print_macro_row`) | Mostly. | Class = normalised surface string; lives inside `main()` as a closure. |
| Post-processing filter (`is_valid_phenotype`) | **Domain-specific.** | Encodes GSC+ conventions (no diseases, no genotypes, no "normal"). |
| Review-metric functions (`experiment2.calculate_*`) | Yes, given records with `is_missing / is_correct / original_negation / corrected_negation`. | Source names and file names are literals. |
| `experiment4.py` | Partly. | Window-label scheme `prev_N`/`post_N`, `meta` keys and the diabetes exclusion are literals; the delta/slope/plot logic is generic. |
| `run.sh` | Yes. | Generic step runner. |

Overall: **moderately modular at the function level, not at all at the package level.** There is
no `__init__.py`, no configuration file, no CLI for experiments 2 and 3, and every script resolves
`../data` relative to its own location. Reuse means copying functions or editing literals, not
importing a library. The one deliberate abstraction (`agreement.py`) exists because code review
found two inconsistent estimators.

## 4. Honest limits of what the released numbers prove

- The physician-review data cover 24–27 visits per reviewer for phenotypes and a smaller
  medication set; the 100-visit single-reviewer validation in Table 1 is not released.
- `med_aggregate_stats.json` is an opaque count file; the per-item reviews behind N=344/179/221
  are not released, so medication precision cannot be re-derived or re-sliced.
- GSC+ predictions were generated upstream; the prompts and decoding settings that produced
  `data/base-*.json` are not fully recoverable from `prompts/`.
- Agreement κ for the GPT medication pipeline is degenerate (constant marginals); the paper
  reports a prevalence-adjusted value, which `agreement.py` prints as PABAK.
