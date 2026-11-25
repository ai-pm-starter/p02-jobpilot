# Evals

## Eval set
Real digests from 5 working days (P2-07). Every job sent in a digest is appended automatically to `evals/digest-labels.csv` by the workflow (nodes *Label rows* → *Rows to file* → *Append to digest-labels.csv*), one row per job: date, rank, score, market, title, company, link, job id. The PM fills in `relevant` (Y/N) by hand, with an optional note, after reading the ads.

**Relevant (Y)** means the PM would open the ad and consider applying: right level and role, a location and language they can work in.

## Metrics and pass bar
| Metric | Definition | Pass bar |
|---|---|---|
| Delivery | Days the digest arrived by 08:00, out of 5 | 5/5 |
| Precision@5 | Share of rank 1–5 rows marked Y, over all 5 days | ≥ 70% |
| Time | Minutes a day spent reading the digest and opening the good ads | < 10 min |
| Cost | Money spent | €0 |

## Results log
| Date | Prompt/model version | Score | Notes |
|---|---|---|---|
