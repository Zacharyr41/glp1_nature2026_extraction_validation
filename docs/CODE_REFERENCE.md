# Code reference — Part I (extraction validation)

Function-level documentation for everything under `code/`. Line numbers refer to the committed
files. Paper-table attributions are inferred from content; the only in-repo cross-reference is
`code/CHANGES.md`, which ties `experiment3.py` to "Table S1a" (Extended Data Table 5 in the
published paper).

---

## `code/agreement.py` — shared inter-rater agreement estimator (99 lines)

Imported by `experiment2.py` and `experiment3_med.py` so both agreement tables use one
estimator.

| Name | Line | Role |
|---|---|---|
| `N_CLASSES = 3` | 24 | Label encoding: `0 = Present`, `1 = Absent`, `2 = Rejected` |
| `TOL = 1e-10` | 25 | Numerical tolerance for "p_e is 1" |
| `confusion_matrix(items, r1, r2)` | 28 | 3×3 matrix over items both reviewers scored; returns `(matrix, n)` |
| `agreement_stats(matrix, n)` | 37 | p_o (diagonal / n), p_e (Σ row_i·col_i / n²), Cohen's κ `(p_o−p_e)/(1−p_e)` or `None` when p_e saturates, PABAK `2·p_o−1`, and `degenerate` = constant marginal **or** p_e saturated |
| `pairwise_agreement(items)` | 68 | Runs `agreement_stats` for every reviewer pair discovered in `items` (`{item_key: {reviewer: label}}`) |
| `format_kappa(stats)` | 87 | `"undefined"` when κ is `None` |
| `HEADER`, `format_row(stats)` | 91, 95 | Fixed-width table rendering: `Rater Pair, N, Agreement (%), p0, pe, Cohen's K, PABAK, Degenerate` |

Why PABAK: if one rater assigns every item to the same class, p_o = p_e exactly and κ = 0
regardless of observed agreement; if both do, κ is 0/0. The module tests that condition directly
rather than comparing the magnitudes of κ and PABAK (the pre-review heuristic).

---

## `code/experiment2.py` — physician review of phenotype extraction (271 lines)

**Purpose.** Inter-rater agreement among three physicians (MD1–MD3) and precision / negation-error
metrics for two phenotype extractors on 24–27 real clinical visits: `PhenoTagger` (non-LLM
baseline) and `Verified` (the authors' GPT pipeline, displayed as "Ours (GPT)").
Feeds **Table 1** (phenotype rows of the comparison block) and **Extended Data Table 1** (phenotype κ).

**CLI.** None. Reads `../data/MD{1,2,3}.json` relative to the script (lines 135–142); exits 1 if any
is missing.

**Inputs.** Each MD file is a list of review records; fields consumed: `reviewer`, `source`,
`is_missing`, `is_correct`, `original_negation`, `corrected_negation` (metrics), and
`visit_key`, `phenotype_name`, `start_pos`, `end_pos`, `label` (agreement). See `DATA_SCHEMA.md`.

**Outputs** (stdout; `run.sh` redirects to `results/experiment2_results.txt`):

| Table | Columns |
|---|---|
| Table 1: Inter-Rater Agreement | `Phenotypes, Rater Pair, N, Agreement (%), p0, pe, Cohen's K, PABAK, Degenerate` per source × pair |
| Table 2: Phenotype Extraction Performance (aggregated) | `Model, N (extracted), N (missing), Precision (%), Negation Error (%), Other Error (%)` |
| Table 2b: Emitted negation distribution | `Model, N (extracted), Asserted present, Asserted negated` |
| Table 3: Per-Reviewer Performance Detail | per source: Strict P/R/F1, Entity P/R/F1, Negation Error Rate, one column per reviewer |

**Metric definitions** (`calculate_performance_metrics`, lines 14–70), per reviewer × source:

```
is_missing            -> missing_count (FN: phenotype in note, not extracted)
else total_extracted += 1
  is_correct          -> correct_count
  elif corrected_negation != original_negation -> negation_errors
  else                -> other_errors
tp_strict = correct_count;  tp_entity = tp_strict + negation_errors
P_strict = tp_strict / total_extracted
R_strict = tp_strict / (tp_strict + missing + negation_errors)
P_entity = tp_entity / total_extracted
R_entity = tp_entity / (tp_entity + missing)
negation_error_rate = negation_errors / tp_entity
```

Points to note: "strict" treats a negation error as both a wrong extraction and a missed
entity; "entity" ignores polarity; `other_errors` (wrong concept) never enter a recall
denominator; recall is only against reviewer-flagged `is_missing` rows.
`calculate_aggregate_performance` (lines 73–120) pools across reviewers, so one extraction
reviewed by three physicians counts three times.

**Agreement.** `build_items` (lines 123–131) drops `is_missing` rows and keys each item as
`"{visit_key}|{phenotype_name}|{start_pos}|{end_pos}"` → `{reviewer: label}`.

**Hard-coded.** `SOURCE_DISPLAY = {'PhenoTagger': 'PhenoTagger', 'Verified': 'Ours (GPT)'}`
(lines 8–11); file names MD1–MD3; the three-class label encoding.

**Functions.** `calculate_performance_metrics(reviews, reviewer_name, source_name)` :14 ·
`calculate_aggregate_performance(reviews, source_name)` :73 · `build_items(source_reviews)` :123 ·
`get_agreement_stats()` :134 (main; loads, prints all tables, returns exit code).

---

## `code/experiment3_med.py` — physician review of medication extraction (125 lines)

**Purpose.** Same design as experiment2 for medications: agreement on three extractors —
`med_ner` ("Bert Large Med NER"), `disease_ner_meds` ("BioNER"), `medications` ("Ours (GPT)") —
plus an aggregate precision table. Feeds the medication rows of **Table 1** and **Extended Data
Table 1**.

**Inputs.** `data/MD{1,2,3}_med.json` (fields `source, reviewer, is_missing, visit_key,
finding_name, finding_index, label`) and `data/med_aggregate_stats.json`
(`{section: {total_extracted, correct, neg_err, other_err}}`). The aggregate counts come from a
wider reviewer set than the three MD files (docstring lines 16–18) and cannot be recomputed from
them. No recall is reported because the source carries no missing count.

**Algorithm.** Agreement identical to experiment2 via `agreement.py`, item key
`"{visit_key}|{finding_name}|{finding_index}"` (no character offsets). Aggregate:
`precision = correct/total_extracted`, `negation_error_rate = neg_err/(correct+neg_err)`,
`other_error_rate = other_err/total_extracted`.

**Headline (results file).** "Ours (GPT)" pairs are degenerate: MD1–MD2 and MD1–MD3 are 22/22
Present (κ undefined, PABAK 1.000); MD2–MD3 has a constant MD2 marginal (κ 0.000 at 95.8 %
agreement, PABAK 0.917). Baselines are non-degenerate (κ 0.78–1.00).

**Functions.** `calculate_aggregate_performance_from_json(data_dir)` :15 · `build_items` :48 ·
`main()` :59.

---

## `code/experiment3.py` — BiolarkGSC+ HPO benchmark (452 lines)

**Purpose.** Scores nine LLM phenotype-prediction files (six base, three supervised-fine-tuned)
and three two-model unions against the GSC+ test gold standard (206 PubMed abstracts, 1,949 HPO
annotations), raw and after a post-processing validity filter. Feeds **Extended Data Table 5**
(the "LLMs (base)", "Distilled SFT", "Post Proc (PP)" and "Multi-Agent" blocks; the "Encoders &
early LLMs" rows are quoted from the literature).

**CLI.** None. `hp.obo` is parsed at import time (lines 82–83), so importing the module (as the
tests do) loads the ontology.

**Inputs.**

- `data/GSCplus_test_gold.tsv` — PubTator-style blocks: PMID line, abstract line, then
  `start\tend\tsurface\tHPO_ID` lines, blank-line terminated. `parse_gold_standard` (lines 197–253)
  keeps `(surface, hpo_id)` and validates but discards offsets. Seven documents have zero
  annotations and are retained as empty.
- `data/hp.obo` — `load_hpo` (lines 44–75) reads `[Term]` stanzas: `id`, `name`, `alt_id`,
  `synonym` (quoted string), `is_a`, `is_obsolete`. 19,650 terms.
- `data/{base,sft}-*.json` — `{pmid: [item, ...]}`; only `item["name"]` and
  `item["hpo_matched_name"]` are used. `negation` and `status` are present but **not scored**.

**Matching** (`check_match`, lines 154–190). Prediction strings are normalised (lower-case,
punctuation → space) and tokenised to frozensets minus `FUNCTION_WORDS` (line 14). A prediction
matches a gold annotation if any layer fires:

1. **exact** — a prediction token-set equals the gold surface form, the canonical HPO label, or
   any `hp.obo` synonym of the gold ID (`gold_expansion`, lines 130–136).
2. **subset** — strict token-subset in either direction, where the smaller side carries at least
   one token outside `GENERIC_TOKENS` (lines 19–23: abnormality, disorder, disease, syndrome,
   finding, increased, decreased, … and plurals).
3. **hierarchy** — the prediction string resolves through `HPO_LABEL_TO_IDS` to the gold concept
   or a *descendant* of it (`hpo_ancestors`, cached transitive `is_a` closure, lines 93–106).
   Predicted ancestors of the gold term do not match unless layer 2 fires.

`difflib` similarity was removed after review because it scored microcephaly/macrocephaly 0.917
and deafness/hearing loss 0.50 (`CHANGES.md` §2).

**Per-class accounting** (`update_name_class_stats`, lines 256–279). Within a document, each gold
annotation is a class keyed by its normalised surface string: TP if any prediction matches, else
FN. Each prediction that matches nothing in the document is an FP under its own normalised name.
Many-to-one matches are allowed in both directions.

**Aggregation** (`print_macro_row`, lines 363–389). Macro precision = mean of tp/(tp+fp) over
classes with ≥1 prediction; macro recall = mean of tp/(tp+fn) over classes with ≥1 gold
annotation; **macro F1 = harmonic mean of those two means** (not the mean of per-class F1);
micro F1 = 2TP/(2TP+FP+FN) over pooled counts. Printed `TP/FP/FN` are the micro sums.

**Post-processing filter** (`is_valid_phenotype`, lines 297–314). Drops a prediction whose
normalised name matches `\bnormal\b`, `\bunremarkable\b`, contains `no abnormal`, matches any
`GENETIC_KEYWORDS` (`mutation, deletion, duplication, translocation, chromosome, gene, allelic,
mosaicism, trisomy, monosomy, karyotype`, with optional plural, word-bounded), or matches
`\bsyndromes?\b`. These encode the GSC+ protocol (phenotypic-abnormality branch only).

**Multi-model union** (lines 408–447). `combined_models` pairs
`base-claude-4.5 + base-gpt-5.2-thinking-low`, `base-gpt-4.1-mini + base-qwen-4b-instruct`,
`sft-gpt-4.1-mini + sft-qwen-4b-instruct`. Per document the two item lists are unioned and
de-duplicated on `(name, hpo_matched_name)`; no voting. This is the recall-maximising
"multi-agent aggregation" of the paper.

**Output.** One table `Model | Type | Prec | Rec | F1 | microF1 | TP | FP | FN`, two rows per model
(`Base` = raw, `PP` = post-processed).

**Functions.** `normalize` :26 · `tokenize` :30 · `load_hpo` :44 · `resolve_hpo_id` :86 ·
`hpo_ancestors` :93 · `gold_expansion` :130 · `prediction_tokens` :139 · `_is_informative` :150 ·
`check_match` :154 · `parse_gold_standard` :197 · `update_name_class_stats` :256 ·
`is_valid_phenotype` :297 · `main` :321 (inner `process_and_report` :345, `print_macro_row` :363).
Module globals `HPO_NAMES, HPO_SYNONYMS, HPO_PARENTS, HPO_ALT_IDS, HPO_OBSOLETE` :82,
`HPO_ID_TOKENS`, `HPO_LABEL_TO_IDS` :110–111.

---

## `code/experiment4.py` — HPO prevalence delta around GLP-1 initiation (439 lines)

**Purpose.** From a pre-aggregated per-window prevalence table, computes for every HPO code the
cumulative after-minus-before prevalence delta at each day threshold, fits a slope, and plots the
ten most negative and ten most positive. Reproduces `comparisonhpo2vis.py` from the proprietary
pipeline (docstring). Exploratory figure; not a numbered display item in the final paper.

**CLI** (lines 348–388): `--json` (default `../data/hpo_timeline_prevalence.json`), `--top 10`,
`--suffix experiment4`, `--output_dir .`, `--data_dir .` (where `hp.obo` is looked up; lines
398–399 override the module defaults unconditionally, so omit `--data_dir` and the legend shows
raw HPO codes), `--no-normalize`.

**Inputs.** `hpo_timeline_prevalence.json` keys `meta.window_labels_before/after`,
`meta.total_patients`, `meta.overall_event_counts`, `overall_cumulative_by_window[label].percent`.
`overall_by_window` and `.counts` are unused. `hp.obo` for `id`/`name` only;
`phenotype.hpoa` is a fallback that is not shipped.

**Algorithm.**
- Thresholds = sorted distinct `|day|` from labels `prev_N` / `post_N` → `[30, 60, 90, 120, 180, 240, 300, 365]`.
- Normalisation (`get_normalization_factors`): `factor[label] = total_patients / overall_event_counts[label]`,
  i.e. prevalence per unit of encounter volume; post-windows have ~3× more events than
  pre-windows, so normalisation damps the post side. `--no-normalize` is the sensitivity run.
- Delta series: `y[t] = pct(post_t) − pct(prev_t)`, missing codes = 0.
- Ranking: `np.polyfit(thresholds, y, 1)[0]`; top/bottom `--top` by slope.
- `EXCLUDE_CODES = {"HP:0000819"}` (Diabetes mellitus) — excluded because it is the indication,
  not an outcome, and would swamp the delta scale (comment lines 43–47).

**Outputs.** `highlight_delta_cumulative_{down,up}_{suffix}.png` at 150 dpi; a log on stdout.

**Functions.** `is_excluded` :62 · `log` :66 · `load_results` :70 ·
`build_hpo_name_map_from_obo` :79 · `build_hpo_name_map_from_hpoa` :112 · `build_hpo_name_map` :142 ·
`extract_window_order` :154 · `label_to_day` :161 · `get_normalization_factors` :170 ·
`get_delta_cumulative_series` :196 · `make_highlight_delta_cumulative_lines` :236 · `main` :335.

---

## `code/test_experiment3.py` — regression tests (148 lines)

Run: `python3 -m unittest discover -s code -p 'test_*.py' -v` (10 tests, all pass).

| Class | Pins |
|---|---|
| `TestGoldParser` | 206 documents, 1,949 annotations, first annotation of PMID 1003450 is "brachydactyly", every annotation has an HPO ID, exactly 7 empty documents |
| `TestMatcherRejectsOpposites` | 14 clinically opposite / wrong-concept pairs must not match (microcephaly vs macrocephaly, hypotonia vs hypertonia, …) |
| `TestMatcherAcceptsVariants` | 8 naming variants must match (seizures ↔ epilepsy, hearing impairment ↔ deafness, intellectual disability ↔ mental retardation, …); bare "abnormality" must not match "abnormality of the eye" |
| `TestValidityFilter` | Keeps `Abnormal EEG`, `renal agenesis`, `generalized hypotonia`; drops `normal findings`, `FBN1 gene mutation`, `Marfan syndrome`, `trisomy 21` |

---

## `code/run.sh`

Bash orchestrator described in `ARCHITECTURE.md`. `run_step LABEL OUTFILE CMD...` redirects stdout
and stderr to `OUTFILE`, records the exit code, and prints the log tail on failure. Exit code of
the script is non-zero if any step failed.

## `code/CHANGES.md`

Ten numbered corrections made in response to code review, each with the defect, the fix, and the
effect on reported numbers (for example: the gold parser dropped the first annotation of every
document, 199/1,750 → 206/1,949; the `'gene' in name` substring filter was discarding
`agenesis`). Read it before trusting any number in `results/` that predates the fix.
