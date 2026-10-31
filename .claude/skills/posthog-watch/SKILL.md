---
name: posthog-watch
description: Read-only PostHog watcher for any product. Finds frontend errors that are new or spiking, and gaps in the activation funnel, scores them, and tells the people who can act on them. Use when asked to check for new errors or crashes, review activation or onboarding, find UX gaps, or send the PostHog digest.
---

A product ships, and PostHog quietly records everything: every exception, every step a new user takes
or doesn't. Nobody reads it. The error channel fills with the same twelve known issues until it gets
muted, and the funnel dashboard gets opened once a quarter. Meanwhile the one error that broke
sign-up this morning, and the step where half of last month's new accounts gave up, sit in the data
unnoticed.

This skill reads PostHog so people don't have to, and speaks up only when something is worth a
person's attention: a **finding**, scored, explained in plain language, and sent to whoever can act
on it.

## Read-only, always

The skill watches; it never touches. It does not resolve, assign, suppress or edit anything in
PostHog, and it creates no tickets. Its only output is reports and the notifications `notify`
configures. It messages nobody else.

## Only what changed

An alert that repeats stops being read. A **finding** is something that changed, never something
that merely exists:

- An error is **new** when its issue was first seen inside the window.
- An error is **spiking** when its count in the window is at least `spike_multiplier` × its
  **baseline** (the average for a window of the same length over the previous `baseline_days`), and at
  least `spike_min_count`, and the window before was **not** already spiking. A spike alerts when it
  starts, not every hour it lasts.
- An activation finding is a step, segment or friction event that **moved** between the current
  `window_days` and the previous `window_days`, or a gap big enough to block people today.

Everything above is computed from PostHog at run time. There is no state file: scheduled runs start
from a fresh checkout and would lose it. Issues listed in `errors.known_issues` are silent unless they
spike.

## Config is the product

Nothing about the product is hardcoded. The project, internal users, the group property (tenant,
org, account), the funnel, the friction events and the product's own words all come from
`.posthog-watch/config.json` at the repo root. Missing → copy `config.example.json` from this skill's
folder and run `/setup-posthog-watch`. Never guess a project id.

Use the product's words: `vocabulary.replace` maps terms to avoid onto the ones the team uses.

## Count real users, not us

Exclude everyone in `internal_users` (email domains, distinct ids, or a cohort) from every count.
Ignore errors matching `errors.ignore_errors`. Count unique persons, not events, and use medians, not
means.

**A small sample is a hunch, not a finding.** Below `activation.min_sample` persons, say so, label it a
hunch, and cap it at P3.

## No personal data leaves PostHog

Notifications carry counts, group names (tenant, org), pages, error messages and links. They never
carry a person's email, name or distinct id. Anyone who needs a person follows the link into
PostHog, where access control already applies.

## Plain language, inferences labelled

Say what broke and who it hit, in the product's terms: "Saving a project fails for 14 users in 3
tenants", not a stack frame. When the trace is minified, infer the feature from the URL and the
message, and say it's an inference.

## Score

Each finding gets 1–3 on four factors, combined as a weighted average (so 1.0–3.0, whatever the
weights). Errors and activation findings use different factors, because "stage" means nothing for a
crash and "trend" means little for a funnel step:

| Factor | 1 | 2 | 3 | Used for |
|---|---|---|---|---|
| **reach** | 1 person or group | a few | many, or a key group | both |
| **severity** | cosmetic | slows people down | blocks them | both |
| **trend** | steady | new | spiking | errors |
| **stage** | late in the funnel | middle | first steps | activation |
| **confidence** | inferred | events | events + replays | both |

Weights live in `priority.errors` and `priority.activation`. The average maps to a **priority**:

- **P1** (≥ `priority.p1_min`, default 2.4): sent now.
- **P2** (≥ `priority.p2_min`, default 1.8): in the daily **digest**.
- **P3**: in the weekly digest.

Always show the priority with a one-line reason: "P1: blocks sign-up for 9 tenants, spiking since 08:00".

## Classify

Every finding is one of:

- **error**: something broke.
- **ux_gap**: something works but people get stuck. Fix it now.
- **feature_opportunity**: people try to do something the product doesn't do.
- **small_change**: a ticket-sized fix with clear evidence.

## Invocation

Three modes. "Errors / crashes" → Error check. "Activation / funnel / UX gaps" → Activation check.
"Digest" → Digest. Nothing specified in a session → Error check, then Activation check.

A session run reports in the chat and sends nothing unless asked. A **routine** run (the prompt says
so) never asks questions: anything missing → post what's missing to the first recipient and stop.
Either way, when `notify.dry_run` is true, show what would be sent instead of sending it.

Queries for each step are in [queries.md](references/queries.md).

### Error check

1. **Window:** the last full `errors.window_minutes` ending `errors.ingestion_lag_minutes` ago (default:
   the previous full hour, 15 minutes late so slow events have arrived). Windows don't overlap, so
   nothing alerts twice.
2. Per issue, apply the exclusions and count events, unique users and groups in the window, the window
   before, and the baseline.
3. Keep only **new** and **spiking** issues. Score them.
4. Per finding: what broke, the page, users and groups affected (by `group_property`), first seen (in
   `timezone`), the issue link `{host}/project/{project_id}/error-tracking/{issue_id}`, and one replay
   link if any.
5. Notify P1 now. P2 and P3 wait for the digests, which recompute them.
6. Nothing qualifies → in a session, say `No new or spiking errors between <start> and <end>.` In a
   routine, send nothing.

### Activation check

1. **Windows:** the last `activation.window_days` and the same length before it, external persons only.
2. **Funnel** through `activation.funnel_steps`, at person level, overall and broken down by
   `group_property`. Note each step's conversion and how it moved.
3. **Time to value:** the median time from `time_to_value.from` to `time_to_value.to`, both windows.
4. **Friction:** for each `friction_events` event, its frequency and the property values that drive it.
5. **Why, not just where:** for the biggest drop, look at paths after the step and two or three
   replays of people who stopped. Counts say where people leave; paths and replays say why.
6. Classify and score each finding, with who's affected and the numbers. Respect `min_sample`.
7. Save the report (see `reports`) and notify per recipient.

### Digest

**Daily** (at `notify.daily_digest_time`): rerun the error check over the last 24 hours and collect
P2 findings, plus one line per P1 already sent. **Weekly** (on `notify.weekly_digest_day`): the
activation check, plus P3 errors from the week. Recomputing beats queueing: nothing is lost between
runs, and nothing stale is sent.

## Notify

Each entry in `notify.recipients` chooses `channels` (`slack`, `email`, or both), the `types` it wants,
and a `min_priority`. Send a finding to every recipient whose filter it passes.

- **Slack:** post to `slack_channel` through the Slack connector.
- **Email:** through the mail connector. It sends **as the person who connected it**, so say so the
  first time. No mail connector → write it to `reports/outbox/` and say so.
- Nothing qualifies → send nothing. Silence is the normal state for errors.

Message format:

```
P1 · error · Saving a project fails
14 users in 3 tenants (acme, globex, initech) since 08:12, 9× the usual rate
Why P1: blocks the core action, spiking, many groups
Issue → <link> · Replay → <link>
```

## Reports

Activation reports go where `reports.save_to` says: `slack` (posted to the first matching recipient's
channel, details in the thread), `notion` (a page under `reports.notion_parent_url`), or `repo`
(`reports/activation-YYYY-MM-DD.md`, in a session only, since routines don't push).

Lead with the answer, then the details. Times in `timezone`. Concise.
