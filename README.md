# Duplicate-Record Matching with Human Review

![Python](https://img.shields.io/badge/Python-3.10+-blue)
![Platform](https://img.shields.io/badge/Run%20on-Google%20Colab-orange)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen)
![Cost](https://img.shields.io/badge/Cost-Free-success)

A duplicate-finder for a product catalog that suggests matches, sends unsure pairs to a human, and **never merges different sizes of the same product**.

## Table of Contents
1. [Problem](#problem)
2. [What I Built](#what-i-built)
3. [Results](#results)
4. [Sample Output](#sample-output)
5. [Acceptance Tests](#acceptance-tests)
6. [Output Files](#output-files)
7. [Tech Stack](#tech-stack)
8. [How to Run](#how-to-run)
9. [Limitations](#limitations)
10. [About Me](#about-me)

## Problem
A product catalog has near-duplicate records: spelling mistakes, different casing, sizes written in different ways (`1L` vs `1000 ml`). A matching tool should find these, but it must NOT silently merge different products. For example, the same orange juice in 1 L and 2 L is two products, even though the name is identical.

## What I Built
- 500 fictional product records with known answers: **380 base products, 70 true duplicates, 50 size variants** (same name, different size)
- **Cleaning and size normalising**: text is lowercased, and every size is turned into one base unit (ml or g)
- **Blocking** to avoid comparing all 124,750 pairs, with a report of the true duplicates it missed
- **Two similarity scores** (TF-IDF on character 3-grams + difflib) averaged into one score
- A **hard size rule**: if both sizes are known and different, the pair is a `variant` and is never suggested as a match
- **Threshold sensitivity table**: precision, recall and how many pairs a human must review
- A **reversible review log in SQLite**: every decision is saved, source records are never changed, undo is just removing the last human row
- Export of the final mapping (`canonical_mapping.csv`) and of the pairs that blocking missed

## Results
### Blocking
| Rule | Candidate pairs | Pairs removed | True pairs kept | Blocking recall |
|---|---|---|---|---|
| A: same category + first 3 letters of brand | 1,915 | 98.5% | 69 of 76 | 0.908 |
| B: same category + first 4 letters of name | 5,117 | 95.9% | 70 of 76 | 0.921 |
| A or B together | 6,607 | 94.7% | 76 of 76 | 1.000 |

One rule alone missed 7 (rule A) or 6 (rule B) true duplicates. Using both together caught all 76 on this data.

### Size rule
Of the 6,607 candidate pairs, **5,796 had different sizes** and 811 had the same size. Many different-size pairs have identical names (similarity 1.0), so a name-only matcher would have merged them.

### Threshold sensitivity
`low` = send to human review above this score, `high` = suggest a match above this score.

| low | high | Suggested matches | Precision | Recall (auto) | Recall (with review) | Pairs to review |
|---|---|---|---|---|---|---|
| 0.5 | 0.90 | 39 | 0.872 | 0.447 | 1.000 | 613 |
| **0.6** | **0.85** | **53** | **0.906** | **0.632** | **1.000** | **388** |
| 0.6 | 0.80 | 73 | 0.849 | 0.816 | 1.000 | 368 |
| 0.7 | 0.80 | 73 | 0.849 | 0.816 | 0.987 | 114 |
| 0.7 | 0.90 | 39 | 0.872 | 0.447 | 0.987 | 148 |

The bold row (0.6 / 0.85) is the one used in the notebook. The trade-off is clear: a lower `low` threshold finds every true duplicate (recall 1.0) but a human must review 388 pairs. A higher one cuts the review work to 114 pairs but misses about 1% of true duplicates.

### Why humans stay in the loop
I accepted **every** suggested match as a stress test. The tool then mapped 51 records to another record, and 46 of those were truly the same product. So about 5 of 51 auto-suggestions were wrong. Auto-merging is not safe by itself, which is why every decision goes through the review log.

## Sample Output
```
TRUE DUPLICATE (suggested):
   0 | Kiro | Cooking Oil | 2 l
   431 | Kiro | CookingOil | 2L
SIZE VARIANT (blocked by size rule, decision = variant ):
   1 | Kiro | Orange Juice | 2 l
   110 | Kiro | Orange Juice | 1 l
UNCERTAIN (goes to review), score 0.669 :
   0 | Kiro | Cooking Oil | 2 l
   40 | GreenLeaf | Cooking Oil | 2 l
```

## Acceptance Tests
| Test | Result |
|---|---|
| Size-variant pairs wrongly suggested or sent to review | 0 of 5,796, PASS |
| Duplicates missed by blocking are in a separate report | PASS (report is empty because blocking recall is 1.0 on this data) |
| Accept a pair, then undo: mapping goes back to the original, 500 source rows still there | PASS |
| After accepting all suggestions, groups containing different sizes | 0 |

## Output Files
- `canonical_mapping.csv`: record id and the canonical id it maps to (source data is untouched)
- `blocking_missed_pairs.csv`: true duplicates that blocking missed
- `threshold_sensitivity.png`: review workload vs recall

## Tech Stack
| Area | Tools |
|---|---|
| Language | Python |
| Data | pandas, numpy |
| Matching | scikit-learn (TF-IDF, cosine similarity), difflib |
| Review log | SQLite (built into Python) |
| Charts | matplotlib |
| Environment | Google Colab |

## How to Run
1. Open `P25_Duplicate_Record_Matching_Human_Review.ipynb` in Google Colab
2. Click **Runtime > Run all**
3. No dataset or API key is needed, all data is generated inside the notebook (seed 42)

Or locally: `pip install -r requirements.txt` and open the notebook in Jupyter.

## Limitations
- The data is synthetic with known answers. It shows that the logic works, not that the same numbers would appear on a real company catalog
- Blocking caught all true pairs here, but on messy real data a missed pair is never even scored, so the missed report matters
- The size rule only works when sizes are written in ml, l, g or kg. Other units become `unknown`
- Precision of auto-suggested matches is about 0.91, so some wrong suggestions are expected

## About Me
**Akshat Kesharwani** - Data Analyst / Data Scientist (fresher)

- Portfolio: https://akshatkesharwani-info.github.io/
- GitHub: https://github.com/akshatkesharwani-info

Idea source: "Top 50 Projects to Build in 2026" (Yasir Arafat Sharfi). The build, data and code here are my own.
