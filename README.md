# posthog-watch

PostHog, read for you. Hear about what changed, and nothing else.

Every hour, Claude checks PostHog for errors that are **new** or **spiking**. Every week, it checks
where new users **get stuck** in your activation funnel. It scores what it finds and tells the people
who can act on it, in plain language, on Slack or by email.

```
 PostHog ──▶ only what changed ──▶ scored P1 / P2 / P3 ──▶ routed to who can fix it
 (errors,     (new or spiking      (reach, severity,       P1 now, P2 daily digest,
  funnel)      errors, funnel       trend or stage,        P3 weekly, on Slack or email
               steps that moved)    confidence)
```

Read-only: it never changes anything in PostHog, and it never creates tickets.

---

## Why

As a product engineer, I want to find out an error broke sign-up **before** a customer tells me, and
know where new users give up **before** it shows up in churn. PostHog has both answers. It records
every exception and every step every user takes. But having the data and acting on it are different
things, and in between sit three ways it gets lost:

- **Alert fatigue.** An error channel that posts every exception becomes the same twelve known issues
  on repeat. Within a month it's muted, and the new one that matters arrives in a channel nobody reads.
- **Dashboards nobody opens.** The activation funnel exists, but someone has to remember to look at
  it, compare it with last month, and dig into *why* step 3 dropped. That happens once a quarter, if
  someone has time.
- **Findings without an owner.** Even when someone spots a problem, it's a screenshot in a thread. A
  designer never hears about the form that blocks 30% of sign-ups, because it was found by whoever
  happened to be looking at errors.

### What this does about it

- **Only what changed.** Errors alert when they're new or when they start spiking against their own
  baseline, once, not every hour they last. Funnel findings are steps that moved against the previous
  period. Known issues stay quiet unless they get worse. Silence is the normal state.
- **Scored, so the important things come first.** Every finding gets P1, P2 or P3 from how many people
  it hits, how badly, whether it's getting worse, and how sure the evidence is. P1 interrupts. P2
  waits for the morning digest. P3 waits for the week.
- **Sent to the person who can fix it.** Engineers get errors; designers get UX gaps and feature
  opportunities. Everyone sets their own minimum priority.
- **In plain language, with the evidence.** "Saving a project fails for 14 users in 3 tenants, 9× the
  usual rate", with links to the issue and a replay, not a stack trace. Guesses are labelled as
  guesses, and a pattern from 12 users is called a hunch.
- **Real users only, no personal data.** Your team's own clicks are excluded from every count.
  Messages carry counts and tenant names, never a person's email or name.
- **Any product.** Everything specific to your product (project, funnel, events, words) lives in one
  config file that setup writes for you.

---

## What you need

- [ ] **Claude Code**, signed in to claude.ai
- [ ] **PostHog and Slack connected on claude.ai**: open
      [claude.ai/customize/connectors](https://claude.ai/customize/connectors). Add Gmail too if
      anyone wants email
- [ ] **A GitHub account**, with the Claude GitHub app allowed on your copy of this repo
- [ ] **PostHog error tracking turned on**, for the error checks. The activation check only needs events

---

## Install (2 minutes)

**1. Make your own private copy.** On GitHub, click **Use this template → Create a new repository →
Private**. (Your config names your funnel and your internal users.)

**2. Open Claude Code inside it.**

```bash
git clone https://github.com/<you>/posthog-watch.git
cd posthog-watch
claude
```

**3. Type:**

```
/setup-posthog-watch
```

Claude finds most of the answers in PostHog itself and asks you only for the rest.

> **Optional: use it in every project.**
> ```bash
> ln -s "$PWD/.claude/skills/posthog-watch" ~/.claude/skills/posthog-watch
> ln -s "$PWD/.claude/skills/setup-posthog-watch" ~/.claude/skills/setup-posthog-watch
> ```

---

## What setup asks you

| Stop | Claude finds | Claude asks you |
|---|---|---|
| 1. Project | the projects PostHog can see | which one, if several |
| 2. Internal users | your email domain from person data | test accounts, staging users |
| 3. Groups | tenant / org / account property | confirm it |
| 4. Funnel | a proposed funnel from your events | fix the steps; which events mean "blocked" |
| 5. Recipients | that the Slack channels exist | who gets what, at which priority; digest time and timezone |
| 6. Save | | OK to push the config |
| 7. Routines | | OK to create the two routines |
| 8. Dry run | runs both once, sends nothing | is this the right amount of noise? |

It starts in **dry-run mode**: for the first week it shows what it *would* send without sending it.
When you trust it, set `"dry_run": false` in the config and push.

---

## Everyday use

The routines run by themselves. You can also ask in any session:

- `any new errors since this morning?`
- `where are new users getting stuck this month?`
- `what changed in activation since last month, by tenant?`

In a session, Claude answers in the chat and sends nothing unless you ask.

---

## What runs when

| Routine | When | Does |
|---|---|---|
| **errors** | every hour at :15 | checks the previous hour; sends P1 errors now |
| **digest** | daily at your digest time | P2 errors from the last 24 h; on your weekly day, also the activation report and P3s |

Each run is a Claude Code session on your plan: about 24 + 1 a day. To run fewer, change the errors
routine to every 2 or 4 hours and set `errors.window_minutes` to match (120 or 240).

---

## Change settings later

Everything lives in `.posthog-watch/config.json`. Edit it, or ask Claude (*"send UX gaps to
#design too"*), then push.

| To change | In the config |
|---|---|
| Who gets what | `notify.recipients`: channels, `types`, `min_priority` |
| Too many alerts | raise `errors.spike_multiplier`, add to `errors.known_issues`, raise `priority.p1_min` |
| Errors to ignore | `errors.ignore_errors` (matched against the message) |
| The funnel | `activation.funnel_steps`, `time_to_value`, `friction_events` |
| What counts as "enough data" | `activation.min_sample` |
| Where reports go | `reports.save_to`: `slack` or `notion` |
| How factors are weighted | `priority.errors`, `priority.activation` |
| Routine times | [claude.ai/code/routines](https://claude.ai/code/routines) → the routine → Edit |

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| Nothing ever arrives | Is `dry_run` still `true`? Then check the routine's runs on claude.ai and read the log. |
| A message says the config is missing | It isn't pushed to GitHub yet. Push it. |
| A message says a connector is missing | Connect it at claude.ai/customize/connectors, then add it to the routine (Edit → Connectors). |
| Same error every day | Add its issue id to `errors.known_issues`. It stays quiet unless it spikes. |
| Counts look too high | Your team is being counted: add domains, ids or a cohort to `internal_users`. |
| Funnel looks empty | An event name in `funnel_steps` doesn't match PostHog exactly. Check the spelling. |
| Errors never show | Error tracking isn't on, or exceptions aren't captured: check for `$exception` events. |

---

## How it's built

```
.claude/skills/
  posthog-watch/
    SKILL.md                 the rules: only what changed, scoring, privacy, the three modes
    config.example.json      every setting, with defaults
    references/queries.md    the HogQL behind each check
  setup-posthog-watch/
    SKILL.md                 the guided setup, run with /setup-posthog-watch
routines/
  errors.md                  hourly prompt
  digest.md                  daily prompt (weekly on your chosen day)
.posthog-watch/config.json   your config, written by setup
```
