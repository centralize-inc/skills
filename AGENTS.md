# AGENTS.md

## Purpose and layout

This repository distributes public Centralize skills for GTM workflows. These instructions apply to agents maintaining the repository; instructions for running a workflow belong in that skill's `SKILL.md`.

- Store each skill in `skills/<skill-name>/SKILL.md`.
- Use a lowercase, hyphen-separated folder name matching the YAML frontmatter `name`. Include a clear `description` that explains when to use the skill.
- Keep each skill self-contained, including its procedure, example prompts, prerequisites, limitations, and verification steps.
- Update the skill catalog in `README.md` when adding, renaming, or removing a skill.

## Editing skills

- Read the complete existing skill before changing it. Preserve operational context and full prompts; do not replace procedures with card summaries.
- Verify product capabilities against current Centralize tools, schemas, and documentation. Distinguish automatic behavior from manual analysis, team rituals, and plan-dependent features. Do not invent tools or promise unavailable automation.
- Preserve instructions about permissions, data freshness, missing evidence, and authorization for writes or outreach. Editing this repository does not authorize running a skill against live customer data.
- Use fictional examples. Never commit credentials, tokens, private customer data, or internal-only material.
- Compare changes with the [published skill library](https://www.usecentralize.com/skills). Describe any website copy that needs the same update in the PR. Repository edits alone do not authorize publishing website changes.

## Validation and review

This is a Markdown collection with no application build or test suite.

- Check that changed `SKILL.md` files have valid YAML frontmatter with `name` and `description`, and that names match their folders.
- Check local links and the README catalog. Read the resulting instructions end to end for missing steps, conflicting claims, and truncated prompts.
- Run `git diff --check` and inspect the complete diff before opening a PR. Report the checks performed and any validation limits in the PR description.
- During review, flag unsupported product claims, lost workflow context, sensitive data, or instructions that bypass user authorization.

## Git workflow

- Make changes on a branch, using `codex/` for agent-created branches, and open a pull request targeting `main`.
- Respect branch protections. A PR requires independent approval after the latest push and resolution of review conversations; do not bypass protections or approve your own work.
- Do not create release tags, publish packages, or trigger release workflows unless the user explicitly requests the specific release and separately confirms its repository, version, source commit, and release actions.
