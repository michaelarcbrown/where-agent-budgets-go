# Where Agent Budgets Actually Go

### A case study of one production multi-agent system: four failure modes behind token bleed and agent loops, and the rule that stopped each one

**Author:** Michael Brown · **Date:** November 2026 · **Type:** case study (observational, single system)

---

## Why this study

Teams running LLM agents in production keep asking the same questions in public:

- *"What did my provider actually bill me for?"*
- *"Is anyone running always-on agents that don't cost a fortune?"*
- *"My agent went haywire - does this output mean anything?"*

A scan of developer forums (Hacker News, Reddit r/SaaS and r/startups, LangChain / AutoGen / LlamaIndex GitHub issues) over
the 90 days to 17 Nov 2026 found **AI cost / tokens** and **agents unreliable / looping** to be the two most-discussed
operational problems among builders, well ahead of every other category (method below).

This study documents a single production system: a voice assistant, a build agent and a set of outbound and data jobs, coordinated by
an orchestrator and run by one operator. Over three weeks it hit four distinct failure modes. Each one was found from logs, traced to
a cause, and fixed with one rule. The rules are the result. None of them is specific to one framework.

## The point

> **Agents rarely burn money by being dumb. They burn it through decisions nobody owns:
> which model to escalate to, who is allowed to say "go", how often something wakes up, and who decides a job is done.**

Each of the four failures below is one of those unowned decisions.

---

## Failure 1 - Escalation bleed

**Symptom.** The prepaid model balance was not going to last to its planned date.

**Measured** (27 Oct - 4 Nov 2026):

| | Share of spend | Jobs proven successful |
|---|---|---|
| Large model (every follow-up job) | **$13.51 of ~$14.40 (94%)** | 21 / 59 (36%) |
| Small model | the rest | 7 / 14 (50%) at ~1/10 the price |

**Cause.** The routing function sent every follow-up job in a conversation straight to the large model, on the assumption that
a second attempt needs a stronger model. The data said otherwise: the large model was not more successful on these jobs.

**Rule.** *Start on the small model. Escalate only after two real, verified failures. Track success per model, not by feel.*

---

## Failure 2 - The permission loop

**Symptom.** The agent asked the operator to raise a budget, said thank you, and then asked again, indefinitely. Work never
resumed on its own.

**Cause.** "May this job start?" was decided in **three places**: a text check in the job runner, a fallback that blamed
"whichever budget is nearest its limit", and the **chat model's own memory of an earlier refusal**. Raising the budget fixed
one of the three. The model kept repeating the refusal it remembered. A refused job was simply dropped, so nothing could
resume it.

**Rule.** *One gate, computed from live numbers on every check. It names the one budget that is actually exhausted. A refused
job is kept and resumes by itself when the gate clears. The model never remembers a refusal.*

---

## Failure 3 - Always-on bleed

**Symptom.** The hosting platform **paused every deployment** for exceeding its monthly CPU allowance (4 active-CPU hours a
month on the plan in use). Dashboards, queues and outbound sending all stopped.

**Cause.** No single change was expensive. Their sum was:

- a keep-warm ping every 4 minutes (360 a day);
- a status view auto-refreshing every 2 minutes;
- every screen reading live data on every load;
- a remote poster polling every few minutes.

In the same period an unwatched oversized worker machine cost $40 in one month on another host.

**Rule.** *Before anything runs on a timer, polls, or fans out on page load, estimate calls a day × cost per call against the
plan, and write the number down. Prefer a cache and a slower interval. Then put a guard on it: a scheduled check that alerts
when any machine, interval or monthly projection crosses a limit.* (After the fix: keep-warm every 15 minutes, a 3-minute
read cache cleared on any write, and a cost guard running every 2 hours.)

---

## Failure 4 - The agent that grades its own homework

**Symptom.** A fix was announced as done. It wasn't.

**Cause.** The completion gate accepted the agent's own statement as proof. Here that statement was a **prediction** ("after
deploy, the queue will show…") rather than an observation. A related bug did the opposite: when a budget wall stopped a
queued fix, the queue marked it **failed** instead of waiting.

**Rule.** *An agent's own logs and statements are not proof. "Done" means the real thing was re-run and its live output was
checked. A job blocked by a budget waits; it does not fail.*

---

## The checklist

1. Small model first; escalate after two **verified** failures; log success per model.
2. One permission gate, from live numbers, every time. No model memory of refusals.
3. Refused or blocked jobs are **kept** and resume automatically.
4. Every timer, poll and auto-refresh has a written calls-a-day × cost estimate.
5. Cache reads; clear the cache on write; slow every interval you can.
6. A scheduled cost guard alerts on projections, not just on bills.
7. "Done" requires an observed live result, never the agent's own claim.
8. Review spend **by decision** (routing, permission, schedule, completion), not only by model or by day.

---

## Method and limits

- **One system, one operator, three weeks (23 Oct - 11 Nov 2026).** Observational case study, not a controlled experiment. The success rates in
  Failure 1 are small samples (59 and 14 jobs) and show the direction of the effect, not its size.
- Every figure comes from the system's own cost log and incident records, written at the time of each incident.
- **Forum signal:** posts were collected from Hacker News, Reddit (r/SaaS, r/startups) and GitHub issues of LangChain,
  AutoGen and LlamaIndex, then put into problem categories. Product launches ("Show HN", "we built…") and news articles were
  excluded, as were duplicate URLs. The category assignment is keyword-based and loose: a manual spot check suggests roughly
  a third of the posts in each category are squarely on topic. The **ranking** (cost and loops far ahead of other categories) held
  up. The absolute counts should not be quoted on their own.
- Framework- and vendor-neutral by design. Model names are withheld because the rules do not depend on them.

## Citation

Brown, M. (2026). *Where Agent Budgets Actually Go: a case study of four failure modes behind token bleed and agent loops.*
Zenodo. DOI: [10.5281/zenodo.24329602](https://doi.org/10.5281/zenodo.24329602)

Related: Brown, M. (2026). Preprint on how language models invent details about real products. DOI 10.5281/zenodo.22059291
