# ADR-001: Build JobPilot v0 in n8n, not as a Python script

- **Date:** Nov 26, 2026 (P2-08)
- **Status:** accepted
- **Decider:** Prashant (PM)

## Context
JobPilot v0 is a fixed daily chain: fetch jobs from two APIs → de-duplicate → score with Gemini → send to Telegram. It has to run every day for free, keep three API keys out of git, and show in a portfolio how an AI product works. The same steps could be built in the workflow tool n8n or as a plain Python script like the P1 eval runner (`run_teardown.py`, 144 lines).

n8n was picked when the problem was scoped (P2-01). This ADR records why, now that v0 is built and running, and what that experience showed. It is written after the build, so the n8n column below uses real evidence, while the Python column is an estimate.

## Options considered

**A. n8n, self-hosted in Docker on the laptop** (what was built)
**B. A Python script on the laptop**, run daily by macOS `launchd` (or cron)
**C. A Python script on GitHub Actions**, run daily by a scheduled workflow in the cloud

| Criterion | A. n8n (Docker, laptop) | B. Python (laptop) | C. Python (GitHub Actions) |
|---|---|---|---|
| **Build speed** | Fast. v0 went from first commit to a working daily digest in about 1.5 hours of commits (P2-02 → P2-06). Schedule, HTTP retries, batching, Telegram node and run history come built in | Similar for one person who knows Python. Retries, rate limiting, logging and the Telegram call have to be written | Like B, plus the Actions setup and secrets |
| **Debugging** | Good to see, weak to test. Each run's data can be inspected node by node in *Executions*. But one-off command-line runs (`n8n execute`) were unreliable for the full workflow, so tests had to go through a schedule set 5 minutes ahead. There are no unit tests | Strong. Unit tests, a debugger, `print`, and reruns from the terminal in seconds | Like B locally; failures in the cloud are read from logs |
| **Logic in code anyway** | 12 of 31 nodes are Code nodes (about 184 lines of JavaScript): de-duplication, mapping, batching, parsing, digest text. n8n doesn't remove the code, it wraps it | All code, in one language | All code |
| **Reviewing changes** | Hard. The workflow is one large JSON file; a small change shows up as a noisy diff | Easy. Normal code diffs | Easy |
| **Secrets** | Stored encrypted in n8n credentials; the workflow file only references them by ID | `.env` file, git-ignored | GitHub Secrets |
| **Hosting and reliability** | Laptop only. No digest if the Mac is asleep or Docker is off; a missed day isn't run later (this happened on Nov 26). n8n Cloud would need a paid plan | Same laptop limit | Runs in the cloud for free, even when the laptop is off. But the anonymised profile and chat ID would have to live in GitHub Secrets, and scheduled runs can start late |
| **Cost** | €0 | €0 | €0 (free minutes for this size) |
| **Portfolio value** | High. A screenshot of the workflow explains the product to a non-engineer in seconds, and "no-code/low-code automation" is a skill PM roles ask for | Medium. Shows coding, but there's nothing to show a non-engineer | Medium-high. Shows coding and a cloud schedule |
| **Path to P12 (JobPilot v1 agent)** | Possible: n8n has AI agent and tool nodes. But an agent that tailors CVs and cover letters needs testable logic and evals, which are weak points here | Strong. Python is where agent SDKs, eval tooling and the P1 runner already are | Strong, and already in the cloud |

## Decision
**Keep n8n (option A) for JobPilot v0.**

v0 is a fixed chain, not an agent, and the main goals are to ship fast, prove the digest is useful (P2-07 eval) and show the product clearly. n8n did that: it was built in an evening, it runs daily for €0, and the keys never touch the repo. Its weak points (testing, diffs, laptop hosting) are acceptable for a one-person v0 that runs once a day.

## Consequences
**Easier**
- Changing a search term, threshold or the digest text is a quick edit in the n8n editor.
- Every run can be inspected step by step in *Executions*, which helps when a digest looks wrong.
- The workflow picture is a ready-made demo for the README and LinkedIn post (P2-10).

**Harder**
- **Reliability depends on the laptop.** Mitigated with a daily 07:25 wake-up (`pmset`), Docker starting at login, and the run moved to UK time on Nov 26 so it lands after the wake-up.
- **No automated tests.** Changes are checked with small test workflows and a scheduled run. Keep the logic in Code nodes small.
- **Workflow JSON is hard to review.** Re-export it after every change (P2-09) and describe changes in the commit message.
- **Hidden state:** the "already seen" list lives in n8n's workflow static data, not in a file. A re-import with empty static data resets it.

## When to revisit
Reopen this decision at the start of **P12 (JobPilot v1)**, or earlier if either of these happens:
- The eval week shows more than one missed digest because the laptop was asleep. Then try option C (GitHub Actions) for the fetch-and-score part.
- The logic grows beyond simple steps (for example an agent that tailors CVs). Then move the scoring and agent logic to Python with tests, and keep n8n only for scheduling and delivery, or drop it.
