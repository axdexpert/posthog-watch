Run the posthog-watch skill in this repository (.claude/skills/posthog-watch), **Error check** mode, as
a **routine**. The config is `.posthog-watch/config.json`.

Check the previous full window (default: the previous full hour, ending 15 minutes ago). Notify P1
findings now, following `notify`. P2 and P3 wait for the digest. Nothing new or spiking → send nothing.

Never ask questions. If the config or a connector is missing, post what's missing to the first
recipient's channel and stop. Never change anything in PostHog.
