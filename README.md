# Uptime Weekly Report (shareable skill)

Generates a weekly report from a self-hosted Uptime Kuma instance, published
as a Claude Artifact, with a "primary category" section (default label
"Work") kept separate from a "Personal" section, plus a 24-week trend
appendix.

## Install

1. Copy this whole folder into your skills directory (e.g.
   `~/.claude/skills/uptime-weekly-report/`), keeping `SKILL.md` at the top
   level of that folder.
2. Do **not** rename `config.json.example` to `config.json` yourself —
   leave it as-is. The first time you ask Claude to run the report, the
   skill will notice `config.json` is missing and ask you for:
   - Your Uptime Kuma instance URL
   - What to call your non-personal category (defaults to "Work")
   - The tag name you use in Uptime Kuma for personal monitors (defaults
     to "Personal")

   It then writes those into `config.json` next to `SKILL.md`, and fills in
   the `artifact_url` field itself the first time it publishes the report.
3. Your Uptime Kuma API key is never written to disk by this skill — you'll
   be asked for it each session (or Claude will use it if you've already
   given it earlier in that same conversation).

## Use

Just ask Claude something like "run my uptime report" or "refresh the
uptime report." After the first run it will keep republishing to the same
artifact link.

## Notes

- `config.json` (once created) contains your instance URL and category
  labels — nothing secret. If you ever want to point the skill at a
  different instance, or start a fresh report artifact, edit or delete the
  relevant field in `config.json`.
- The 24-week trend history lives inside the published artifact's own
  database, not in this folder, so it survives independently of this skill
  install.
