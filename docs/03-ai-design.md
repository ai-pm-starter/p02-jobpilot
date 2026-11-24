# AI design

- **Complexity level:** chain. A fixed n8n workflow: fetch → de-duplicate → score → send. No agent.
- **Model(s) and why:** `gemini-3.6-flash`, free tier. It is the same model as P1, it's fast enough (about 15–20 s for 25 jobs), and it costs €0. ADR in P2-08.
- **Prompt(s):** `prompts/v1-fit-score.md` (P2-05)
- **Tools the model can call:** none. The model only scores text it is given.
- **Guardrails:** the model gets an anonymised profile and job ads only. It can't act or send anything; n8n decides what goes to Telegram. The prompt tells it to ignore instructions inside job ads. Temperature is 0 and the output must be JSON.
- **Human-in-the-loop:** the PM reads the digest and decides whether to apply.
- **Failure modes:** a bad score means a good job ranks low or a poor one ranks high. This is measured in the P2-07 eval (precision@5). If a batch fails, its jobs show as unscored (score null, flag `unscored`) instead of silently disappearing.
- **Privacy:** what leaves the laptop: search terms to the job APIs; job ads plus the anonymised profile to Gemini; the digest to Telegram. The CV never leaves the laptop.

## Data sources

### Germany: Bundesagentur für Arbeit Jobsuche API (P2-03)
Checked Nov 24, 2026. **Unofficial:** the Bundesagentur has no official public API. This is the API behind its own job search, documented by the community at [bundesAPI/jobsuche-api](https://github.com/bundesAPI/jobsuche-api). It may change without notice.

| | |
|---|---|
| Search | `GET https://rest.arbeitsagentur.de/jobboerse/jobsuche-service/pc/v6/jobs` (v4 search now returns 403) |
| Details | `GET …/pc/v4/jobdetails/{base64(referenznummer)}` for the full description (v6 details returns 403) |
| Auth | Header `X-API-Key: jobboerse-jobsuche`. This is a public client id published in the docs, not a secret |
| Link for users | `https://www.arbeitsagentur.de/jobsuche/jobdetail/{referenznummer}` |
| Politeness | Search: 1 request per second. Details: 5 requests per second at most. One run a day, about 85 requests |

**Search terms** (node *DE search terms*): `Product Manager`, `Produktmanager`, `Product Owner`, `Head of Product`. `Produktmanagerin` returns the same jobs as `Produktmanager`, so it is left out.

**Filters** (node *BA search*):
- `angebotsart=1`: jobs only, no internships, training or self-employment.
- `veroeffentlichtseit=1`: published in the last day.
- `size=100`, `page=1`: the largest term returned 42 jobs a day, so one page is enough. Paging can be added if a term gets close to 100.
- Germany-wide, with no `wo`/`umkreis` location filter. Location and remote fit are left to scoring, because remote-friendly roles can be anywhere.

**De-duplication:** by `referenznummer` across search terms (node *Flatten + de-duplicate*). De-duplication against jobs already sent in earlier digests comes in P2-05.

**First run (Nov 24, 2026):** 4 searches gave 93 results, which were 80 unique jobs, all with descriptions (about 3,700 characters on average). The run took 21 s. About a third are not PM roles (e.g. quality manager, design engineer, brand manager) because the BA search matches loosely. Scoring has to filter these out.

### London: Adzuna API (P2-04)
Checked Nov 24, 2026. Free developer key, no payment details. Personal job search is allowed by the [terms](https://developer.adzuna.com/docs/terms_of_service). Commercial use or reselling the data would need a licence.

| | |
|---|---|
| Search | `GET https://api.adzuna.com/v1/api/jobs/gb/search/{page}` |
| Auth | `app_id` + `app_key` query parameters, held in the n8n Custom Auth credential (`{"qs": {"app_id": …, "app_key": …}}`). The workflow file only holds the credential's id, never the keys |
| Link for users | `redirect_url` (an Adzuna page) |
| Politeness | 1 request every 1.5 s, 3 requests a day |
| **Attribution** | Every London job shown must be labelled **"Adzuna"** with a link to adzuna.co.uk. Jobs carry `attribution: "Adzuna"`, and the P2-06 digest must show it |

**Search terms** (node *UK search terms*): `product manager`, `product owner`, `head of product`, matched in the **title only** (`title_only`).

**Filters** (node *Adzuna search*): `where=London`, `distance=25` (km), `max_days_old=1`, `results_per_page=50`, `sort_by=date`.

**Why title-only:** `what_phrase` also matches the description. On the first run it returned 86 jobs, many not PM (business development, buying admin, data architect). Title-only returned 27, almost all PM or PO roles. Title-only matches the words anywhere in the title, so a few near-misses get through, such as "Product Trainer / Training Manager" and "Product Marketing Manager". Scoring handles those.

**Limits:**
- Adzuna returns only a **snippet of the description** (about 500 characters, against about 3,700 for German jobs), so London jobs are scored on less text. We don't scrape the full ads.
- Remote/hybrid is only detected from words in the title and snippet.
- Salary is kept only when the employer gave it, not when Adzuna predicted it.

**First run (Nov 24, 2026):** 3 searches gave 27 unique London jobs. Together with Germany: **107 jobs a day** go into *All jobs*.

## Common job shape
Every source maps to the same fields (node *Map to job schema*), so scoring and the digest don't care where a job came from:

| Field | Example | Notes |
|---|---|---|
| `id` | `ba:19301-986202680-S` | source prefix + source id |
| `source`, `market` | `bundesagentur`, `DE` | |
| `title`, `company`, `location` | `Senior Product Manager (m/w/d)`, …, `Berlin` | |
| `remote` | BA: `no`, `NACH_VEREINBARUNG`, `ANGABE_IN_PROZENT`; Adzuna: `mentioned`, `not mentioned` | normalised in P2-05 if needed |
| `published` | `2026-11-24` | |
| `salary` | `60000–80000 (JAHR)` or null | given for about 1 in 5 jobs |
| `description` | text, max 6,000 characters | markdown bold removed |
| `link`, `external_link` | public BA page or Adzuna page; employer page if given | |
| `attribution` | `Adzuna` or absent | must be shown next to the job in the digest |

## Scoring (P2-05)

```
All jobs → Title pre-filter → Drop already seen → Make batches → Gemini score → Parse scores
```

| Step | What it does | Why |
|---|---|---|
| Read profile / prompt | Reads `profile/profile.md` and `prompts/v1-fit-score.md` from read-only mounts. File nodes can only read `/home/node/jobpilot` | Profile stays out of git and out of the workflow JSON. Prompt is versioned |
| Title pre-filter | Keeps titles with "product" or "Produkt". Drops marketing, training, control, approval, Produktion, sales (whole word), Vertrieb, designer, Konstrukteur, quality, internships and apprenticeships | Saves quota: 107 → 73 jobs on Nov 24. Borderline titles go to the model |
| Drop already seen | Skips job ids scored in the last 30 days (workflow static data) | The same job isn't scored or sent twice |
| Make batches | 25 jobs per request, description cut to 1,500 characters, **short ids 1..n per batch** | The free tier ran out after roughly 20–25 requests a day on Nov 15. This needs about 3 requests a day |
| Gemini score | `generateContent`, JSON output, temperature 0, 5 s between batches, 3 tries | |
| Parse scores | Maps scores back by batch position, sorts best first, records ids as seen | |

**Profile:** `profile/profile.md` (git-ignored) is the only candidate text sent to Gemini. It has no name, contact details, employer or school names, or exact identifying figures. `profile/profile.example.md` shows the format. The PM confirmed the target roles, languages, work rights and locations on Nov 24.

**Prompt v1** (`prompts/v1-fit-score.md`): weighs role first, then skills and domain, then constraints. Hard caps: not PM max 20, German as working language max 35, junior/intern max 40, location mismatch max 40. Returns score, a reason of up to 20 words, and flags (`german_required`, `not_pm`, `too_junior`, `too_senior`, `location_mismatch`, `visa_unclear`, `ai_product`, `short_description`).

**First run (Nov 24, 2026):** 75 jobs in 3 requests: about 12k input and about 1.7k output tokens per batch of 25, 55 s in total.
- **Scores:** 10 strong (85+), 4 good (70–84), 7 partial (50–69), 53 poor (<50).
- **Most common flags:** `german_required` 49, `visa_unclear` 45, `location_mismatch` 37, `not_pm` 22. Most German ads ask for fluent German, so with this profile the strong matches are mostly London roles plus a few English-speaking ones in Germany (e.g. Munich, Berlin).
- **Top 3:** Senior PM Data (London, 92), AI Product Owner (Munich, 88), Senior PM B2B SaaS (London, 88).
- **Bug found and fixed:** the model shortened one long Bundesagentur id (`…-S` dropped), so that job came back unscored. Batches now use ids 1..n, which the model copies reliably.

## Digest (P2-06)

```
Daily 07:30 → … → Make batches → Has jobs? ─yes→ Gemini score → Parse scores → Build digest → Send to Telegram
                                           └─no──────────────────────────────→ Build digest
```

- **Trigger:** schedule trigger at 07:30 Berlin time. The workflow is published (n8n 2.x `publish:workflow`). A missed run (laptop asleep or off) is not caught up.
- **Content:** one Telegram message (HTML) with the **top 5 always**, plus any other job scoring **70+**, up to 10. Each job shows the score, a linked title, company · location · market, and the model's reason. Link previews are off.
- **Attribution:** London jobs show "via Adzuna" linked to adzuna.co.uk (Adzuna terms).
- **Header line:** date, jobs fetched per market, new jobs scored, and how many scored 70+.
- **Bad days are visible, not silent:** "No new PM jobs today" when nothing new came in. A ⚠️ line names any source that failed or returned nothing. A ⚠️ line counts jobs the model failed to score. Empty sources and failed calls pass an empty result on (n8n `alwaysOutputData` / continue on error), so the message is still sent.
- **Secrets and personal data:** bot token in the n8n Telegram credential. Chat id in `profile/telegram-chat-id.txt` (git-ignored, read-only mount). Neither is in the workflow JSON.
- **Tested Nov 24, 2026:** a scheduled production run (trigger temporarily set 5 minutes ahead) delivered the digest to Telegram. The PM confirmed receipt. One-off command-line runs (`n8n execute`) proved unreliable for the full workflow because they compete with the running instance's task runner, so tests now go through the schedule.
