---
name: setup-posthog-watch
description: Set up posthog-watch for a product. Finds the PostHog project, internal users and activation funnel from the data, writes the config, and schedules the hourly error check and the daily digest.
disable-model-invocation: true
---

Someone wants to hear about the error that broke sign-up this morning without watching a dashboard,
and to know where new users give up without building a funnel. By the end of this, PostHog is
watched every hour, the right people hear about what changed, and nobody hears about what didn't.

Eight **stops**. Show this line at each one, with the current stop in bold:

```
1 Project · 2 Internal users · 3 Groups · 4 Funnel · 5 Recipients · 6 Save · 7 Routines · 8 Dry run
```

## Find, then ask

PostHog already knows most of the answers. Before asking anything, look: the projects the connector
can see, the event definitions, the person properties, the group types. Ask only what the data can't
tell you, and offer your best guess as the default.

## Confirm before it leaves the machine

Drafting the config is free. Pushing it and creating routines are not. Show what's about to happen,
and act on a yes. Never write a person's email or name into anything except `internal_users`.

## The stops

### 1. Project

List the projects the PostHog connector can see. One → confirm it. Several → ask. Record its id and
host (`us.posthog.com` or `eu.posthog.com`). No connector → send the user to
https://claude.ai/customize/connectors.

### 2. Internal users

The team inflates every count. Find the company's email domains from the person properties (the
domain with the most persons that matches the user's own email is usually it), and ask about test
accounts, staging users, or a cohort that already holds them.

### 3. Groups

Does the product have a tenant, org, workspace or account? Check the group types and the event
properties. Record the property name as `group_property`, or leave it out for a single-tenant
product.

### 4. Funnel

The **activation funnel** is the path from signing up to the first moment the product proved its
worth. Propose one from the event definitions (sign-up, the first object created, the first real
outcome), and ask the user to fix it. Then:

- **time to value:** from which step to which.
- **friction events:** events that mean someone got blocked (validation errors, empty searches,
  "upgrade required"). Propose candidates from the event list.
- **vocabulary:** words the team avoids, and what they say instead.

### 5. Recipients

Who should hear about what? For each recipient: the channel (Slack, email or both), which types
(`error`, `ux_gap`, `feature_opportunity`, `small_change`), and the minimum priority. Check each Slack
channel exists. Email sends as whoever connected the mail connector: say so.

Then the **digest** time and the weekly day, in the user's timezone.

### 6. Save

Write `.posthog-watch/config.json` from `config.example.json`, with `notify.dry_run: true`. Show the
diff; commit and push on a yes. Routines read the config from the repo, so an unpushed config doesn't
exist as far as they're concerned.

### 7. Routines

Create two through the `/schedule` skill, after showing the full configuration of each. Both use this
repo, the PostHog connector, Slack, and the mail connector when a recipient wants email.

- **posthog-watch: errors**, prompt `routines/errors.md`, every hour at :15 (`15 * * * *`). The 15
  minutes let slow events arrive before their hour is checked.
- **posthog-watch: digest**, prompt `routines/digest.md`, daily at the digest time. Convert it to UTC,
  say the conversion back, and warn that cron ignores daylight saving, so the local time shifts by an
  hour at the clock change.

End the stop with both routine links.

### 8. Dry run

Run both routines once ("Run now") and read their logs. In dry-run mode they report what they would
send without sending it. Go through it with the user: too noisy → raise `spike_multiplier`, add
`known_issues`, or raise a recipient's `min_priority`; too quiet → the reverse.

Leave `dry_run` on for the first week. Setup is done when the user has seen a dry run they trust. Tell
them that turning it on is one line, `"dry_run": false`, pushed.
