# Releasing rms-fit-scoring

The whole system: one repo, one skill, versioned by `plugin.json`.
You never download or zip anything — edit the files in this repo and push.

## Files that matter
- `plugins/rms-fit-scoring/skills/rms-fit-scoring/SKILL.md` — the logic (your team owns this)
- `.../weights.json` — the tunable config (weights, bands, low-activity list). Retune here only.
- `plugins/rms-fit-scoring/.claude-plugin/plugin.json` — the **`version`** field is the release valve.

## To ship a change (every time)
1. Edit the file(s) — by hand, or just tell Claude "update the skill to ..." while this folder is open.
2. Bump `version` in `plugin.json`:
   - PATCH (0.1.0 -> 0.1.1): wording/typo, no behavior change
   - MINOR (0.1.1 -> 0.2.0): new instruction/capability, weights retuned, backward-compatible
   - MAJOR (0.2.0 -> 1.0.0): scoring behavior changes in a way that moves scores
3. Commit, tag, push:
   ```bash
   git add .
   git commit -m "describe the change"
   git tag v0.2.0
   git push origin main --tags
   ```
4. (Optional pre-flight) `claude plugin validate ./plugins/rms-fit-scoring`

Users get it by running `/plugin marketplace update` inside Claude.
If you push commits WITHOUT bumping `version`, users see nothing — safe for work-in-progress.

## First-time setup for a new user (once each)
```
/plugin marketplace add <your-org>/rms-fit-scoring-marketplace
/plugin install rms-fit-scoring@g2-skills
```

## Note on user tuning vs. updates
`weights.json` ships inside the plugin. If a downstream operator edits their *installed* copy
to tune weights, a `/plugin marketplace update` will overwrite it with the repo's version.
While it's just your team iterating, this is fine. Once outside operators need tuning that
survives updates, switch to a user-level override file (a `weights.local.json` outside the
plugin that SKILL.md reads if present) so their tuning isn't clobbered on update.
