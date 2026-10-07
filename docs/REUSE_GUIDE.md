# Reuse guide — running this validation on a different clinical question or on MIMIC

This repository validates *outputs* of an extraction pipeline. It does not contain the pipeline.
So "reusing it" means: you build (or already have) an extractor, you collect physician labels and
a gold corpus in the formats below, and you re-point four scripts at them. The outcome-analysis
side of the paper is covered in the sibling repository `glp1_nature2026_a1c_analysis/docs/REUSE_GUIDE.md`.

## 1. What you get for free, and what you must write

| Need | Status in this repo | What you have to write |
|---|---|---|
| Inter-rater agreement (κ, PABAK, degeneracy) | `code/agreement.py`, reusable unchanged | A loader that turns your review export into `{item_key: {reviewer: 0/1/2}}` |
| Precision / recall / negation-error metrics from physician review | `experiment2.py` functions, reusable | A review tool (or spreadsheet) that produces records with `is_correct`, `is_missing`, `original_negation`, `corrected_negation` |
| Benchmark against a public gold corpus with ontology-aware matching | `experiment3.py` matcher, HPO loader, macro/micro scorer | A gold parser for your corpus format; an OBO loader for your ontology if not HPO; a new validity filter |
| Multi-model union | `experiment3.py` lines 408–447 | nothing |
| Prevalence-delta figure around an index event | `experiment4.py` | A script that produces `hpo_timeline_prevalence.json` from your patient-level extractions |
| **The LLM extractor itself** (prompts, delta tracking, verification, ontology mapping) | **Absent** | Everything: see §4 |
| Concept discovery (embedding + clustering + labelling) | Output only | Everything |
| Knowledge graph | Absent | Everything |

## 2. Adapting to a different clinical question (same data source)

The experiments are question-agnostic except for a handful of literals.

### 2.1 Physician-review experiments (`experiment2.py`, `experiment3_med.py`)

1. Export reviews in the `data/MD*.json` schema (`DATA_SCHEMA.md` §1–2). The required fields are
   `visit_key`, `source`, a surface string, a within-visit position (`start_pos/end_pos` or
   `finding_index`), `reviewer`, `is_missing`, `is_correct`, `original_negation`,
   `corrected_negation`, `label`.
2. Edit `SOURCE_DISPLAY` (and `MED_SECTIONS`) to name your extractors.
3. Edit the file-name lists (`MD1.json` … ) or, better, replace them with a glob.
4. If your task has no negation concept (for example lab values), set `original_negation ==
   corrected_negation` everywhere; the "negation error" column will be zero and "other error"
   carries all incorrect extractions.
5. Decide the agreement label semantics. The 3-class encoding (Present / Absent / Rejected) is
   what `agreement.py` assumes; for a binary correct/incorrect task use classes 0 and 2 only.

### 2.2 Gold-corpus benchmark (`experiment3.py`)

| Change | Where |
|---|---|
| Different corpus format | rewrite `parse_gold_standard` to return `{doc_id: {"phenotypes": [(surface, concept_id)]}}` |
| Different ontology (SNOMED, RxNorm, ICD, MONDO…) | rewrite `load_hpo` to return `names`, `synonyms`, `parents`, `alt_ids`, `obsolete` dicts keyed by your IDs; drop the `HP:` prefix filter at line 60. The matcher only needs labels, synonyms and a parent map. |
| Different "what counts as an entity" protocol | rewrite `is_valid_phenotype`; the current one implements GSC+ rules (no diseases, no genotypes, no normal findings) |
| Your models | edit the `model_files` list in `main()` and `combined_models` |
| Prediction schema | `prediction_tokens` reads `name` and `hpo_matched_name`; map your fields onto those |

Keep the three-layer matcher as is. It was tuned after review to reject clinically opposite
pairs; the pinned test cases in `test_experiment3.py` are a good regression harness and should
be extended with opposites from your domain (e.g. hyper/hypokalaemia, brady/tachycardia).

### 2.3 Prevalence deltas (`experiment4.py`)

Produce a JSON with the §7 schema of `DATA_SCHEMA.md`: for each window label `prev_N` / `post_N`,
the per-concept count of patients with at least one extraction in the cumulative window, plus
per-window encounter counts for normalisation. Change `EXCLUDE_CODES` to the concept(s) that are
your indication rather than an outcome. Replace the HPO name map if you use another vocabulary.

## 3. Adapting to MIMIC (MIMIC-IV with MIMIC-IV-Note, or MIMIC-III)

MIMIC gives you the two inputs the paper's platform starts from — structured tables and free-text
notes — but not any of the intermediate products this repo evaluates. Expect to write the whole
extraction layer.

### 3.1 Mapping MIMIC to the paper's inputs

| Paper concept | MIMIC-IV source | Notes |
|---|---|---|
| Patient | `hosp.patients` (`subject_id`) | `anchor_age`, `anchor_year` instead of DOB; ages > 89 are shifted |
| Visit / encounter | `hosp.admissions` (`hadm_id`), `icu.icustays`, `hosp.emar` | Ambulatory visits as in the paper do not exist; MIMIC is inpatient / ED |
| Clinical note | `mimic-iv-note` `discharge.csv`, `radiology.csv` (MIMIC-III: `NOTEEVENTS` with many note types) | One discharge summary per admission; no longitudinal outpatient narrative. Delta extraction across visits is only meaningful across *admissions* |
| Medications (structured) | `hosp.prescriptions`, `hosp.emar`, `hosp.pharmacy` | Drug names free text + NDC/GSN; map to RxNorm yourself (RxNav API or `rxnorm` tables) |
| Diagnoses | `hosp.diagnoses_icd` (ICD-9 and ICD-10) | billing codes, assigned at discharge |
| Labs (HbA1c etc.) | `hosp.labevents` + `hosp.d_labitems` (HbA1c itemid 50852; glucose 50931) | LOINC codes available in `d_labitems` for many items |
| Vitals / weight | `icu.chartevents` (admission weight itemid 226512, daily weight 224639), `hosp.omr` (outpatient weight, BMI, BP in MIMIC-IV ≥2.0) | `omr` is the only outpatient-style measurement table |
| Phenotypes / symptoms | none — must be extracted from notes | HPO is a reasonable target; the GSC+ benchmark transfers directly |
| Verification score | none | you implement the 0–3 judge (`prompts/verification_prompt.txt` gives the rubric) |

### 3.2 Which experiments transfer directly

- **experiment3 (GSC+ benchmark)**: runs unchanged; it does not touch MIMIC at all. Use it to pick
  and tune your extractor before touching PHI-bearing notes.
- **experiment2 / experiment3_med**: transfer once you have a review export. Sample N discharge
  summaries, run your extractor, have ≥2 clinicians label every extracted statement in the
  schema of §2.1, add missed items as `is_missing` rows, blind a third reviewer to adjudicate
  disagreements (the `adjudication/*.csv` files show the record layout the authors used).
- **experiment4**: transfers once you have per-patient extraction timelines; index event could be
  first insulin order, first ICU admission, etc. MIMIC time-shifting is per patient and consistent
  within patient, so "days from index" is valid.

### 3.3 What you must build (ordered)

1. **Note segmentation and de-duplication** — discharge summaries are long and sectioned;
   the paper's extractors work on a visit note plus the prior visit for delta context.
2. **Domain extractors** (medication, phenotype, assessment at minimum). Use
   `prompts/phenotype_prompt.txt` as a template: it shows the output schema (`name`, `negated`,
   `severity`, `clinical_status`, `certainty`, `atomic_statement`) and the full-visit vs
   differential prompt pattern. The undefined constants (`TEMPORAL_INSTRUCTIONS`,
   `NEGATION_INSTRUCTIONS`, `DETAILS_INSTRUCTIONS`, `COMMON_FINDING_SCHEMA_BLOCK`) you write
   yourself.
3. **Atomic-statement verification** — second LLM pass scoring each statement 0–3 against the
   source span (`prompts/verification_prompt.txt`). Keep the score on every record; the paper
   reports the distribution (Table 2) and uses it as a filter.
4. **Ontology grounding** — HPO for phenotypes (reuse `experiment3.load_hpo` for label and
   synonym lookup, add an embedding or string-match step), RxNorm for drugs, LOINC for labs.
5. **Timeline assembly** — one record per (patient, day, concept) with provenance (note id,
   span). This is what both the prevalence JSON and the Part II wide CSV are derived from.
6. **Review tooling** — a UI or spreadsheet that shows note + statement + span and captures the
   verdict fields.
7. **Optional**: concept discovery (sentence embeddings → k-means with silhouette sweep → LLM
   labelling → UMAP), which only has its *output* here (`clustering/cluster_hierarchy.json`).

### 3.4 Pitfalls specific to MIMIC

- De-identification tokens (`___`) replace names, dates and some drug names; the authors' notes
  show "REDACTED" medication names caused apparent misses (`adjudication/adjudication_notes.md`).
  Expect the same.
- No outpatient longitudinal narrative: "visit-to-visit delta" becomes "admission-to-admission".
- Negation and family-history sections are dense in discharge summaries; the paper's negation
  handling (polarity kept as attribute) is essential, and PhenoTagger-style tools fail here
  (83 % negation error in Table 1).
- Fine-tuning on MIMIC text requires the PhysioNet credentialed-data agreement and prohibits
  sending text to most hosted LLM APIs unless they are covered (Azure OpenAI under the PhysioNet
  responsible-use terms; check the current policy). Local models (Qwen 2.5 4B as in the paper)
  avoid the question.

## 4. Minimum viable reimplementation checklist

- [ ] Extractor producing `{doc_id: [ {name, hpo_matched_name, negation, status, atomic_statement} ]}` → run `experiment3.py` against GSC+.
- [ ] Verification judge producing a 0–3 score per statement.
- [ ] Review export in `MD*.json` schema → run `experiment2.py` (phenotypes) and a medication analogue.
- [ ] Per-window prevalence JSON → run `experiment4.py` with your indication in `EXCLUDE_CODES`.
- [ ] A wide per-patient-day CSV (see Part II `docs/DATA_SCHEMA.md`) → run the outcome pipeline.
