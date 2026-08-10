# RMS Fit Scoring — skill

Scores a G2 vendor product for **Review Managed Services (RMS)** fit and produces a
go / no-go scorecard. Distributed as a Claude plugin from this repo.

## Use it (one-time setup)

You need **read access to this repo** first. It's private, so:
- make sure you've been added as a collaborator (or it's under a G2 org you're in), and
- your local git is authenticated to GitHub (`gh auth login`, or an SSH key on your account).

Then, inside Claude (Claude Code, or Cowork / desktop with the `/plugin` interface), run:

```
/plugin marketplace add smarimadappa/rms-fit-scoring-marketplace
/plugin install rms-fit-scoring@g2-skills
```

That's it — the skill is now available. Ask something like *"score Salesforce Sales Cloud
for RMS fit"* and it runs.

> Requires Looker to be connected — 7 of the 11 scoring inputs come from it. The skill
> checks this first and tells you if it's missing.

## Get the latest version

Whenever a new version ships, pull it with:

```
/plugin marketplace update
```

You only ever run `add` and `install` once. `update` is the only command you need after that.

## Who maintains this

The RMS team owns the skill logic (`SKILL.md`, `references/`, the scorecard script).
All tunable numbers — weights, recommendation bands, the low-activity industry list —
live in `plugins/rms-fit-scoring/skills/rms-fit-scoring/weights.json`.
To request a change or a retune, ping the team. See `RELEASING.md` for the release process.
