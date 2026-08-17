# RMS Fit Scoring — skill

Scores a G2 vendor product for **Review Managed Services (RMS)** fit and produces a
go / no-go scorecard. Distributed as a Claude plugin from this repo.

## Use it (one-time setup)

You need **read access to this repo** first. It's private, so:
- make sure you've been added as a collaborator (or it's under a G2 org you're in), and
- your GitHub account is authenticated (in Cowork: signed in to GitHub; in Claude Code:
  `gh auth login` or an SSH key on your account).

### In Cowork (desktop app) — most people

1. Open **Customize** in the sidebar → **Plugins**.
2. Click **Add marketplace** and enter `smarimadappa/rms-fit-scoring-marketplace`
   (the `owner/repo` shorthand or the full GitHub URL both work).
3. Find **rms-fit-scoring** in the list and click **Install**.

> Note: `/plugin ...` is **not** a Cowork chat command — use the Plugins screen above.

### In Claude Code (terminal) — if you code

Inside an interactive `claude` session:

```
/plugin marketplace add smarimadappa/rms-fit-scoring-marketplace
/plugin install rms-fit-scoring@g2-skills
```

That's it — the skill is now available. Ask something like *"score Salesforce Sales Cloud
for RMS fit"* and it runs.

> Requires Looker to be connected — 7 of the 11 scoring inputs come from it. The skill
> checks this first and tells you if it's missing.

## Get the latest version

Whenever a new version ships:
- **Cowork:** go to **Customize → Plugins** and update rms-fit-scoring there.
- **Claude Code:** run `/plugin marketplace update`.

You only add + install once. After that, updating is the only step you ever repeat.

## Who maintains this

The RMS team owns the skill logic (`SKILL.md`, `references/`, the scorecard script).
All tunable numbers — weights, recommendation bands, the low-activity industry list —
live in `plugins/rms-fit-scoring/skills/rms-fit-scoring/weights.json`.
To request a change or a retune, ping the team. See `RELEASING.md` for the release process.
