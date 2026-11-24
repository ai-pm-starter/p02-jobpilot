# Problem statement

PM: Prashant · Drafted Nov 24, 2026 (P2-01)

## User
A product manager looking for a new PM role in **Germany or London**, reading job ads in German or English. Primary persona: a mid-level to senior PM who checks several job boards every day while working or job hunting.

## Pain
Today they open several job boards every morning, rerun the same searches and skim dozens of ads that are mostly irrelevant (wrong seniority, wrong domain, German required at C2, not really a PM role).
- **Time:** about **1 hour a day**, roughly 5 hours a week.
- **Missed roles:** good roles get missed on busy days or sink under duplicates, and early applicants have a better chance.

## Why now / why AI
Keyword alerts already exist, but they match words, not fit. "Product Manager" matches project managers, product owners for hardware, and junior roles. A model can read the whole ad and judge it against a profile: seniority, domain, required languages, location and remote rules. It can then explain the score in one line, which a rule or a saved search can't.

## v0: what it does
A daily **Telegram digest** of new PM jobs, built as an **n8n workflow** (fixed steps, no agent):
1. **Fetch** new PM jobs from the last 24 h: the **Bundesagentur für Arbeit Jobsuche API** for Germany and a free job API for London (**Adzuna or Reed**, picked in P2-04).
2. **De-duplicate** across sources and against jobs already sent.
3. **Score** each job **0–100** against the candidate's profile with **Gemini** (free tier), with a one-line reason.
4. **Send** the top jobs to Telegram as one message: title, company, location, score, reason, link.

**Privacy:** the free tier may let Google use what we send, so Gemini gets an **anonymised skills profile**, not the CV: roles, years, skills, domains, languages and location preferences. No name, contact details, addresses or employer-confidential details.

## Success metrics
| Metric | Target | How measured |
|---|---|---|
| Reliability | Digest arrives every day | 5 of 5 days in the eval week (P2-07) |
| Quality | ≥ 70% of each day's top 5 judged relevant | PM labels each top-5 job relevant / not for 5 days (25 jobs); precision@5 in `docs/04-evals.md` |
| Speed | Digest ready in under 10 min after the trigger | n8n execution log |
| Time saved | Daily job-board check from ~1 h to under 10 min | PM self-report during the eval week |
| Cost per task | €0 (free tier) | No paid plan or card on any service |

## Out of scope (v0)
- Applying, writing cover letters or tailoring the CV (candidates for P12 JobPilot v1).
- Job boards without a free official API. No scraping of LinkedIn, StepStone or similar.
- Markets other than Germany and London.
- Hosting in the cloud: n8n runs in Docker on the PM's laptop, so the digest only goes out when the laptop is on (known limitation).
- Multiple users or a sign-up flow: one profile, one Telegram chat.
