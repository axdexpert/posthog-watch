Run the posthog-watch skill in this repository (.claude/skills/posthog-watch), **Digest** mode, as a
**routine**. The config is `.posthog-watch/config.json`.

Send the daily digest: errors from the last 24 hours, P2 findings in full and one line per P1 already
sent. If today is `notify.weekly_digest_day` in `timezone`, also run the Activation check and send
the weekly digest with it, plus the week's P3 errors.

Never ask questions. If the config or a connector is missing, post what's missing to the first
recipient's channel and stop. Never change anything in PostHog.
