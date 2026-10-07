# Data schema — Part I (extraction validation)

Every file under `data/`, `adjudication/`, `clustering/` and `prompts/`, with the fields each
script consumes. Counts were verified by loading the committed files.

## 1. Physician phenotype reviews — `data/MD1.json`, `MD2.json`, `MD3.json`

JSON list, one record per model-extracted phenotype statement reviewed by one physician
(MD1 n=1,336; MD2 n=1,374; MD3 n=1,025; 24 shared visits, MD3 reviewed 27).

| Field | Type | Meaning | Used by |
|---|---|---|---|
| `visit_key` | str, e.g. `VISIT-01` | De-identified visit identifier | item key (experiment2) |
| `source` | str ∈ {`PhenoTagger`, `Verified`} | Which extractor produced the statement; `Verified` = the authors' GPT pipeline | experiment2 |
| `phenotype_name` | str | Extracted surface string | item key |
| `start_pos`, `end_pos` | int | Character offsets in the (unreleased) note text | item key |
| `reviewer` | str ∈ {`MD1`,`MD2`,`MD3`} | One value per file | experiment2 |
| `original_negation` | bool | Polarity asserted by the extractor (`True` = negated) | negation accounting |
| `corrected_negation` | bool | Polarity according to the physician | negation accounting |
| `is_correct` | bool | Physician verdict on the extraction | precision |
| `is_missing` | bool | `True` for phenotypes the physician found in the note that the extractor missed (reviewer-added rows) | recall; excluded from agreement |
| `label` | int ∈ {0,1,2} or null | Agreement class: `0` Present, `1` Absent (negated), `2` Rejected; null when `is_missing` | `agreement.py` |

Observed joint structure: `label 0` ⇔ `is_correct` and not `corrected_negation`;
`label 1` ⇔ `corrected_negation` (either a correct negated extraction or a negation error);
`label 2` ⇔ not `is_correct` with unchanged polarity; `label null` ⇔ `is_missing`.

## 2. Physician medication reviews — `data/MD1_med.json`, `MD2_med.json`, `MD3_med.json`

JSON list (MD1 n=319; MD2 n=149; MD3 n=217).

| Field | Type | Meaning | Used by |
|---|---|---|---|
| `visit_key` | str | as above | item key (experiment3_med) |
| `source` | str ∈ {`med_ner`, `disease_ner_meds`, `medications`} | BERT Large Med NER / BioNER / authors' GPT pipeline | experiment3_med |
| `finding_name` | str, e.g. `Amlodipine 5 MG / valsartan 160 MG Oral Tablet [Exforge]` | Extracted medication string (RxNorm-style for the GPT pipeline) | item key |
| `finding_index` | int | 0-based position in the visit's extraction list | item key |
| `reviewer`, `original_negation`, `is_correct`, `corrected_negation`, `is_missing`, `label` | as in §1 | | only `label`, `is_missing`, `reviewer`, `source` are used |

## 3. `data/med_aggregate_stats.json`

Pre-computed counts over a wider reviewer set than the three MD files; not recomputable here.

```json
{"med_ner":          {"total_extracted": 344, "correct": 225, "neg_err": 13, "other_err": 106},
 "disease_ner_meds": {"total_extracted": 179, "correct": 146, "neg_err":  6, "other_err":  27},
 "medications":      {"total_extracted": 221, "correct": 211, "neg_err":  5, "other_err":   5}}
```

These are the `N` and precision values in the medication rows of Table 1.

## 4. `data/GSCplus_test_gold.tsv` — BiolarkGSC+ gold standard (third-party, see `NOTICE`)

PubTator-style text blocks separated by blank lines:

```
1003450                                   <- PubMed ID (digits only)
A syndrome of brachydactyly (...)         <- abstract text
14	27	brachydactyly	HP:0001156        <- start \t end \t surface \t HPO ID
...
```

206 documents, 1,949 annotations, 405 distinct HPO IDs. Seven documents carry no annotations
within the phenotypic-abnormality branch and are retained as empty. `experiment3.py` uses columns
3 and 4 only.

## 5. `data/hp.obo` — Human Phenotype Ontology, release 2025-05-06 (third-party)

OBO 1.2, 19,650 `[Term]` stanzas. Fields parsed: `id`, `name`, `alt_id`, `synonym`, `is_a`,
`is_obsolete` (experiment3); `id`, `name` (experiment4).

## 6. Model predictions on GSC+ — `data/base-*.json` (6 files), `data/sft-*.json` (3 files)

`{pmid: [item, ...]}` over 199 PMIDs (the 7 empty gold documents are absent). One item per
extracted phenotype:

| Field | Type | Meaning | Scored? |
|---|---|---|---|
| `name` | str | Extracted phenotype string | yes |
| `hpo_matched_name` | str | The pipeline's downstream ontology match (HPO label) | used only as an extra surface string |
| `negation` | bool | Polarity | **no** |
| `status` | str (`unknown`, `mild`, `moderate`, `severe`, …) | Severity | no |
| `is_new_information` | bool | Delta-extraction flag | no |
| `details` | str | Supporting sentence from the abstract | no |
| `atomic_statement` | str | Single-proposition restatement | no |

Files: `base-claude-4.5`, `base-gpt-4.1-mini`, `base-gpt-4.1-nano`, `base-gpt-5-mini-thinking-low`,
`base-gpt-5.2-thinking-low`, `base-qwen-4b-instruct`, `sft-gpt-4.1-mini`, `sft-gpt-4.1-nano`,
`sft-qwen-4b-instruct`. Item counts range from 516 (GPT-4.1 Nano) to 1,425 (SFT GPT-4.1 Nano).
Note this schema uses `negation`/`status`, whereas `prompts/phenotype_prompt.txt` asks for
`negated`/`severity`/`clinical_status`/`certainty`; the prediction files are post-standardisation
outputs of the proprietary pipeline, not raw LLM JSON.

## 7. `data/hpo_timeline_prevalence.json` — cohort-level HPO prevalence around GLP-1 initiation

```
meta:
  generated_at: "2025-12-31T16:06:09Z"
  total_patients: 15000
  windows_days: [30, 60, 90, 120, 180, 240, 300, 365]
  window_labels_before: ["prev_30", ..., "prev_365"]
  window_labels_after:  ["post_30", ..., "post_365"]
  overall_event_counts: {label: int}        # encounters in that window (post_30: 129,246; prev_30: 42,979)
overall_by_window:            {label: {"counts": {HP_id: int}, "percent": {HP_id: float}}}   # unused
overall_cumulative_by_window: {label: {"counts": {...},        "percent": {...}}}            # used
```

`percent = round(counts / total_patients * 100, 4)` holds for every entry. Cumulative windows
contain 2,623 (prev_30) to 5,010 (post_365) HPO codes. This is the only patient-level-derived
GLP-1 cohort artefact in Part I; it is an aggregate (no patient identifiers).

## 8. `clustering/cluster_hierarchy.json` — concept-discovery taxonomy (not read by code)

Seven event domains, each `{cluster_id: {"count": int, "label": str, "parent": str}}`:
`adherence_barriers` (48 clusters), `adherence_recommendations` (47), `dose_change_event` (48),
`hospitalizations` (49), `med_stop_event` (48), `med_switch_event` (50), `med_symptom_event` (49).
`label` is the LLM-generated fine cluster name, `parent` the LLM-generated coarse category.
This is the serialised result of Fig. 2a's two-level taxonomy (k-means over OpenAI embeddings,
LLM labelling), but the embeddings, the clustering code and the note spans are not released.

## 9. `adjudication/` — blind third-physician adjudication (not read by code)

- `adjudication_decisions_blind.csv`: `visit_num, section, finding_index, finding, reviewerA,
  A_verdict, A_reason, reviewerB, B_verdict, B_reason, adjudication, timestamp` — one row per
  disagreement on the 20-visit subset (sections `assessment_status`, `medications`, …).
- `recall_adjudications_blind.csv`: `visit_num, section, finding, flagged_by, term_in_note,
  adjudication, timestamp` — reviewer-flagged misses with the adjudicator's `ok_to_omit` /
  miss verdict.
- `adjudication_notes.md`: the adjudicating physician's verbatim rationale.

These document the "adjudicated reference (20 visits)" column of Table 1 but are not joined to the
MD JSON files by any script (the visit numbering differs: `visit_num` integers vs `VISIT-NN`).

## 10. `prompts/` — fragments of the proprietary extraction prompts (not read by code)

- `phenotype_prompt.txt`: Python source excerpt of the phenotype agent: system message, HPO
  domain instructions, the full-visit prompt and the differential (delta) prompt returning
  `added / modified / removed`, plus `_standardize` which coerces LLM output into the common
  finding schema. References undefined constants (`TEMPORAL_INSTRUCTIONS`,
  `NEGATION_INSTRUCTIONS`, `COMMON_FINDING_SCHEMA_BLOCK`, …).
- `verification_prompt.txt`: the LLM-judge prompt scoring each atomic statement 0–3 for support
  by the source text and 0–3 for ontology name-match quality.

## 11. Results — `results/`

Plain-text logs of each experiment's stdout (the tables documented in `CODE_REFERENCE.md`) and
four PNGs from experiment4. All are regenerated by `code/run.sh`.
