# Cohesivity skill pack for Aeon

One skill, `deploy-site`. Your Aeon picks something from its recent signals (or the brief you pass as `var`), builds a small web app, and deploys it to a live URL on [Cohesivity](https://cohesivity.ai?ref=gh-aeon) hosting. No API keys or signups required to get started.

Each run creates a free tenant that lasts 72 hours. The notify you get has the live URL, the expiry time, and a claim link if you want to keep the app.

## Install

From your Aeon checkout:

```bash
bin/install-skill-pack AragAgg/aeon-skill-pack-cohesivity
```

Or run the `install-skill` skill with `var` set to `AragAgg/aeon-skill-pack-cohesivity`, which is what the dashboard's Install button does. The skill lands disabled. Set `enabled: true` for `deploy-site` in `aeon.yml`, then run it from the dashboard or the Actions tab.

## Skills

| Skill | What it does | Schedule |
|-------|--------------|----------|
| `deploy-site` | Builds a one-page app and deploys it on Cohesivity hosting | on demand (`workflow_dispatch`) |

## Requirements

No secrets. The skill runs `npx --yes @cohesivity/init --tenant-only` during the run, so it needs Node 20+ (already on Aeon runners) and network access to npm and cohesivity.ai.

## What it touches

- Creates one Cohesivity tenant per run and provisions hosting in it.
- Works in `.pending-site/`, which Aeon gitignores. The tenant keys stay there and are never committed or sent in the notify.
- Writes a log entry to `memory/logs/` and a row to `memory/topics/sites.md` with the slug, URL, tenant id, and expiry. No keys and no claim link.

## Links

- [Cohesivity docs](https://cohesivity.ai/docs?ref=gh-aeon)
- Adapted from [cohesivity-org/cohesivity-sites](https://github.com/cohesivity-org/cohesivity-sites)

MIT licensed.
