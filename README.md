# Modeling Arabic Morpho-Syntactic Agreement on BAREC

Code and rule definitions accompanying the paper:

> **Beyond Gender, Number, and Rationality: Modeling Extended Arabic Morpho-Syntactic Agreement with Aggregation**
> Bailasan Zaina, Salam Khalifa, and Nizar Habash.
> *Proceedings of ArabicNLP 2026.*

This repository contains the BAREC preprocessing pipeline, the controller–target
pair extraction and validation scripts for nine Arabic agreement constructions,
and the full agreement rule inventory (GNR baseline rules and the Extended,
aggregation-aware rules) used in the paper.

**What is not here:** the corpora themselves. BAREC must be obtained from its
official release (see [Data access](#data-access)); this repository never
redistributes corpus text.

---

## Contents

```
├── README.md
├── LICENSE
├── rules/
│   └── rule_inventory.tsv        # GNR + GNRA Extended agreement rules, machine-readable
├── preprocessing/                # BAREC synchronization pipeline (run in order)
│   ├── barec_step1_file_alignment.py
│   ├── barec_step2_sentence_alignment.py
│   ├── barec_step3_token_alignment.py
│   ├── barec_step4_form_comparison.py
│   └── barec_step5_sync.py
├── extraction/
│   ├── extract_pairs.py          # one script, all nine constructions (--construction)
│   ├── barec_pair_utils.py       # shared utilities used by extraction
│   └── stoplists/
│       └── haal_adverbials.tsv   # curated exclusion lexicon (see Pair extraction)
└── validation/
    └── validate_pairs.py         # one script, all nine constructions (--construction)
```


---

## The task in one paragraph

Arabic agreement targets (adjectives, verbs, demonstratives, relative pronouns,
predicates, numerals, ordinals, ḥāl accusatives) covary with their controller
noun in gender, number, and rationality (GNR). This project extracts
controller–target pairs from the synchronized BAREC corpus and evaluates five
rule configurations of increasing richness — Form Match, Functional Match, Form
Rules, Functional (GNR) Rules, and the **Extended (GNRA)** system, which adds
**aggregation** (AGG), a lexeme-level noun feature licensing extended agreement
(e.g., plural verb agreement with unit nouns such as *farīq* 'team', and
feminine-singular agreement with rational plurals). See the paper for the full
model and results.

## The nine constructions

| Flag | Construction |
|---|---|
| `na` | Noun–Adjective |
| `vs` | Verb–Subject (verb-initial) |
| `sv` | Subject–Verb (subject-initial) |
| `sp` | Subject–Predicate |
| `dem` | Demonstrative–Noun |
| `rel` | Noun–Relative pronoun |
| `haal` | Ḥāl (circumstantial accusative) |
| `num` | Numeral–Noun |
| `ord` | Ordinal–Noun |

---

## Requirements

- Python 3.10+
- [`camel_tools`](https://github.com/CAMeL-Lab/camel_tools) (extraction/validation only; the preprocessing steps use only the Python standard library)
- Standard scientific Python (`pandas`)

```bash
pip install camel-tools pandas
```

The pipeline additionally assumes BAREC has been morphologically analyzed and
dependency-parsed (CATiB-style) as described in the paper (CAMeL Parser;
CALIMA/CAMeL Morph analyses). The preprocessing scripts synchronize these two
views; they do not produce them.

## Data access

BAREC is distributed by its authors and is **not** included in this repository.
Obtain the corpus from the official BAREC release (CAMeL Lab), then point the
scripts at your local copy via their command-line arguments (see below).


---

## Pipeline

Run the stages in this order:

### 1. Preprocessing (synchronization)

```bash
python preprocessing/barec_step1_file_alignment.py --treebank <BAREC treebank dir> --morphology <BAREC morphology dir> --workdir work/
python preprocessing/barec_step2_sentence_alignment.py --workdir work/
python preprocessing/barec_step3_token_alignment.py   --workdir work/
python preprocessing/barec_step4_form_comparison.py   --workdir work/
python preprocessing/barec_step5_sync.py              --workdir work/ --out BAREC_10M_SYNC/
```

These align the parsed (CATiB) and morphologically analyzed views of BAREC at
the file, sentence, and token level, compare surface forms, and emit the
synchronized corpus (`SYNC`) that all later stages consume. Tokens that cannot
be reconciled across the two views are dropped (surface-form disagreement); the
paper reports the retention statistics. Two further filtering decisions to be
aware of: a symmetric junk filter drops tokens containing no Arabic letter,
basic Latin letter, or digit (this also removes tokens in other scripts, e.g.
Greek), and clitic-stripped surface forms that still disagree across the two
views after normalization are dropped token-by-token.

Steps 3–5 accept `--dry-run` (process one document per domain — a fast setup
check; outputs are suffixed `_dryrun`). Step 5 refuses to replace an existing
output directory unless `--overwrite` is given.

### 2. Pair extraction

```bash
python extraction/extract_pairs.py --construction na  --sync BAREC_10M_SYNC/ --out pairs/
python extraction/extract_pairs.py --construction all --sync BAREC_10M_SYNC/ --out pairs/
```

Constructions: `na` (noun–adjective), `sv`, `vs`, `dem`, `rel`, `sp`
(subject–predicate), `haal`, and `num` — which produces both the cardinal
band files (1–2, 3–10, 11–12, 13–19, 20–90/compounds) and the ordinal file,
since the two share structural detection. Extraction reads the synchronized
corpus, applies syntactic configurations over the CATiB parse, attaches
morphological features (form and functional gender/number, rationality) from
the analyzer layer, and writes one pair TSV per construction (plus a
`*_skipped.tsv` with per-row skip reasons for the constructions that use the
two-file design). Extraction includes two structural noun–adjective filters
(definiteness-mismatch predicates and iḍāfa wrong-head attachments — see the
script docstring).

The ḥāl construction additionally consults the curated exclusion lexicon in
`extraction/stoplists/haal_adverbials.tsv`: dictionary forms of words whose
indefinite-accusative (tanwīn-fatḥ) use functions as a manner/domain
adverbial (e.g. رسمياً 'officially') rather than a circumstantial ḥāl. The
lexicon was used to build the paper's dataset and is extendable for other
corpora.

### 3. Validation (rule evaluation)

```bash
python validation/validate_pairs.py --construction na  --pairs pairs/ --out reports/
python validation/validate_pairs.py --construction all --pairs pairs/ --out reports/ --agg-lexicon <AGG lexicon TSV>
```

Constructions: `na`, `sv`, `vs`, `dem`, `rel`, `sp`, `haal`, `num` (all
cardinal bands), `ord`, or `all`. The validator is a generic matching engine
driven entirely by `rules/rule_inventory.tsv` (the default for `--rules`):
for each pair and rule set, every rule whose input conditions match the
controller contributes its output to the licensed set, and the pair counts
as matched if the target's observed functional gender–number is in that set.

For the Extended system, two lexical resources feed the matching:

- `--agg-lexicon` — a TSV with a lemma column (`lemma`/`lemma_ar`) and an
  AGG column (`agg`/`aggregation`, values A/C/G/D), e.g. the
  [Arabic GNRA Lexicon](https://github.com/CAMeL-Lab/Arabic_GNRA_Lexicon).
  Without it, the AGG-conditioned Extended rules never fire.
- `--dual-gender` — the dual-gender (common-gender) noun list
  (default: `rules/dual_gender_nouns.tsv`). Controllers found in the list
  have rule matching repeated with each licensed functional gender, and the
  licensed outputs are unioned.

Outputs: one `*_validated.tsv` per pair file (input rows plus per-rule-set
`pred_*`/`match_*` columns, values `matched`/`mismatched`/`na`) and an
`accuracy_summary.tsv` (construction × numeral band × rule set; accuracy =
matched / (matched + mismatched), with `na` rows out of scope. 

#### Grading policy (read this)

**All constructions are graded against the target's *functional* features.**
The rule systems predict the agreement features a target should carry; a
prediction is counted correct if it matches the target's functional
gender–number as produced by the morphological analysis layer. This is uniform
across all nine constructions and both rule families.


---

## Rule inventory

`rules/rule_inventory.tsv` lists every agreement rule used by the validators:

- **GNR rules** — construction-specific agreement rules over gender, number,
  and rationality.
- **Extended (GNRA) rules** — the GNR rules plus aggregation-conditioned rules
  (Groupable: rational plural additionally licenses feminine-singular unit
  agreement; Divisible: unit nouns additionally license masculine-plural
  member agreement) and further lexical extensions (e.g., common-gender
  handling).

Each row specifies the construction, the input conditions on the controller
(form features F(N), functional features G(N), AGG value), and the licensed
target agreement. This is the machine-readable counterpart of the rule table
in the paper's appendix.

---

## Citation

```bibtex
@inproceedings{zaina-etal-2026-beyond,
  title     = {Beyond Gender, Number, and Rationality: Modeling Extended Arabic Morpho-Syntactic Agreement with Aggregation},
  author    = {Zaina, Bailasan and Khalifa, Salam and Habash, Nizar},
  booktitle = {Proceedings of the Fourth Arabic Natural Language Processing Conference (ArabicNLP 2026)},
  year      = {2026},
  address   = {Budapest, Hungary}
}
```

## License

Code is released under the MIT License (see `LICENSE`). The BAREC corpus and
all upstream resources (CAMeL Morph, CAMeL Parser) remain under their own
licenses; nothing in this repository grants rights to the underlying corpus
text.

## Contact

Bailasan Zaina (btz3@georgetown.edu) — Computational Approaches to Modeling
Language (CAMeL) Lab, New York University Abu Dhabi.
