
# FOMC Sentiment Analysis: Does Fed Communication Predict Market Reactions?

For this project, I conducted a textual analysis of Federal Reserve communications (FOMC statements and meeting minutes, 1994–2026), scored using the Loughran-McDonald financial sentiment dictionary, tested against same day Treasury yield and equity market reactions.

## Motivation

Central bank communication is itself a policy tool. The Fed doesn't only move markets through rate decisions, but through how it describes the economy and its outlook. This project asks the question: **does the sentiment of FOMC statement language and communication have measurable explanatory power over how markets react on announcement day?**

This project was built as practice in the kind of textual analysis used in Federal Reserve research, and in quantitative/text based signal construction relevant to systematic investing.

## Data

- **FOMC statements** (1994–2026, 194 documents) and **FOMC minutes** (1995–2026, 201 documents), scraped via the [`FedTools`](https://github.com/David-Woroniuk/FedTools) Python library, which pulls directly from federalreserve.gov.
- **Sentiment dictionary**: the [Loughran-McDonald Master Dictionary](https://sraf.nd.edu/textual-analysis/resources/), the standard finance specific sentiment lexicon, used instead of a generic sentiment dictionary because financial/policy vocabulary is frequently misclassified by general purpose sentiment tools.
- **Market data**: daily 10-year Treasury yield (`^TNX`), 5-year Treasury yield (`^FVX`), and S&P 500 (`^GSPC`), pulled via `yfinance`.

## Methodology

1. **Scrape** raw statement and minutes text via `FedTools`.
2. **Clean** scraped text: strip website navigation/boilerplate contamination, normalize across multiple historical document formats (the Fed's page structure and document phrasing changed several times between 1994 and 2026).
3. **Score sentiment** using the LM dictionary: tokenize each document, count positive/negative word matches, normalize by document length to produce `net_sentiment = (positive_count - negative_count) / total_words`.
4. **Construct market reactions**: for each statement date, compute the change in each market series from the prior trading day's close to the announcement day's close.
5. **Regress** market reaction on sentiment score (OLS), with robustness checks (heteroskedasticity-robust standard errors, outlier sensitivity analysis).

## Data Quality Issues Found and Fixed

This dataset required substantially more cleaning than initially expected, and catching these issues rather than building analysis on top of silently bad data was a core part of the project.

**1. Date misalignment in the scraping library.** The `FedTools` library's returned index did not reliably match the dates of its own document content. Cross-checking the date *mentioned inside* each document's text against its assigned index date revealed 14+ misaligned rows in the minutes dataset alone, including several pairs of meetings that appeared to be swapped with each other. The fix: rebuilt the date index directly from text extracted from each document, rather than trusting the library's metadata. This also surfaced several true duplicate date collisions, which were resolved by identifying and dropping malformed scraped copies.

**2. Inconsistent document header formats across eras.** FOMC minutes have used at least four distinct opening formats since 1994 (narrative "A meeting... was held," "A joint meeting... was held" post-2021, header-style "[Date] PRESENT:" during 2008–2011, and case-inconsistent "Present:" variants). Boilerplate stripping logic had to handle all four, validated by checking the marker-match rate iteratively rather than assuming a single regex pattern would generalize across 30 years of website history.

**3. Content duplication in older scraped documents.** Several 1996–2003 minutes documents were found to be 10–80x longer than normal due to repeated sentence level content from the scrape, diagnosed by tracing a recurring "Secretary" signature phrase, confirming via direct chunk-by-chunk text comparison that substantive paragraphs were duplicating, and resolved using sentence boundary based deduplication with fingerprint matching, validated by checking that zero sentences still repeated post fix.

Each of these was caught through verification rather than assumed correct, a discipline applied throughout, given how much these bugs would have silently distorted results if undetected.

## Results

OLS regression of same-day market reaction on `net_sentiment`, statement-level (n=194):

| Dependent variable | Coefficient | p-value | R² |
|---|---|---|---|
| 10-year yield change | 0.554 | 0.293 | 0.006 |
| 5-year yield change | 0.624 | 0.321 | 0.005 |
| S&P 500 change | -57.42 | 0.795 | 0.0004 |

**No statistically significant relationship was found** between same day FOMC statement sentiment and same-day market reaction, across any of the three market series tested. This null result was checked for robustness and held up:
- **Heteroskedasticity-robust standard errors (HC3)** did not change the conclusion.
- **Excluding the single most extreme observation** (March 18, 2009 — the QE1 announcement, the largest yield reaction in the sample) left the coefficient and significance level essentially unchanged, though it did normalize the residual distribution, meaning the outlier was distorting the *shape* of the regression's errors, but was not the reason the relationship is weak.

## Interpretation and Limitations

A few reasons this null result is plausible rather than a sign of a flawed method:

- **Same-day close-to-close returns are a noisy proxy for "reaction to the statement."** FOMC statements release mid-day (2:00pm ET), so a full day return mixes pre-announcement and post-announcement trading. A tighter intraday window (e.g., 30 minutes around the release) is the standard in the academic event-study literature but requires data not available through free sources like yfinance.
- **Markets likely price in much of the expected statement content in advance.** What moves markets on the day itself is the *surprise* relative to expectations, not the sentiment level, a variable this analysis doesn't yet construct.
- **The actual policy decision (rate change) is a confound not yet controlled for.** A 25bp hike paired with hawkish language is a different event than a hold paired with the same language; the current regression doesn't separate sentiment's effect from the decision's mechanical effect on yields.
- **Dictionary-based sentiment measures tone, not necessarily policy stance.** Exploratory analysis of the time series found that net sentiment turned positive through 2009–2010, a period of continued economic weakness, apparently driven by the Fed's deliberate use of reassurance-oriented forward guidance language ("information received... suggests stabilization") even while describing a still troubled economy. This is a meaningful distinction: a positive/negative word count captures communicative tone, which can diverge from the underlying economic or policy reality it's describing.

## Future Work/Revisions

- Add the FOMC's actual rate decision (available via FRED) as a control variable, and test whether sentiment explains the *residual* market reaction after the mechanical decision effect is accounted for.
- Construct a market-expectations baseline (e.g., from fed funds futures pricing) to isolate the "surprise" component of each statement, rather than its absolute sentiment level.
- Repeat the analysis on minutes (released ~3 weeks after the decision) to test whether sentiment in minutes predicts subsequent market drift rather than same day reaction.
- Separate sentiment into "current assessment" vs. "forward guidance" language, since these likely have different market implications.

## Tech Stack

Python (pandas, re, statsmodels, yfinance), Loughran-McDonald financial sentiment dictionary, Git/GitHub for version control.

## Repository Structure

```
fomc-sentiment-analysis/
├── data/
│   ├── raw/          # unmodified scraped text and source dictionary
│   └── processed/    # cleaned, date-validated, sentiment-scored datasets
├── src/
│   ├── scrape.py
│   ├── preprocess.py
│   ├── sentiment.py
│   └── analysis.py
├── notebooks/
│   └── exploration.ipynb
└── README.md
```
