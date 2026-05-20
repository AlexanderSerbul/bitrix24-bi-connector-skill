# Bitrix24 BI-connector — Claude Code skill

🇬🇧 English | [🇷🇺 Русский](README.ru.md)

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that teaches Claude how to talk to the Bitrix24 BI-analytics HTTP API. With this skill active, Claude can list tables, describe their columns, and fetch rows with date-range filtering and predicate filters.

## Getting started

After installing the skill, start the conversation by telling Claude your Bitrix24 portal URL and your bi-token — for example:

> My Bitrix24 portal is `https://example.bitrix24.ru` and my bi-token is `<your-token>`.

Claude will reuse them for every query in the rest of the session, so you only need to provide them once. See [Getting a bi-token](#getting-a-bi-token) below if you don't have one yet.

## What Claude can do once the skill is active

- "How many successful leads do I have on my Bitrix24 portal in last three months?"
- "Show me deals with stage = successful created in the last 3 months."
- "List tasks with TITLE containing 'CRM:' and ID between 21 and 27."
- "Find tasks whose DEADLINE is null."

## What's inside

- `.claude/skills/bitrix-bi-connector/SKILL.md` — the skill itself.
- `BICONNECTOR_PROTOCOL.md` — the underlying API specification (Russian), source of truth that the skill is built from.

## Installing the skill

**Project-local (default).** Claude Code picks up `.claude/skills/<name>/SKILL.md` automatically when invoked inside this project directory. Clone the repo and run `claude` in it — the skill activates on relevant prompts.

```bash
git clone <repo-url>
cd <repo-dir>
claude
```

**User-wide.** Copy the skill folder once so it's available across all your projects.

```bash
cp -r .claude/skills/bitrix-bi-connector ~/.claude/skills/
```

On Windows PowerShell:

```powershell
Copy-Item -Recurse .claude\skills\bitrix-bi-connector $env:USERPROFILE\.claude\skills\
```

## Getting a bi-token

You need a BI-analytics key from your own Bitrix24 portal. Navigate to *CRM → Аналитика → Оперативная аналитика → BI-аналитика → Настройки BI-аналитики → Управление ключами* (this is the path on a Russian-locale portal — Bitrix24's default; menu names may differ in other locales), or open `https://<your-portal>.bitrix24.ru/biconnector/key_list.php` directly. The token is a credential — keep it out of source control and logs.

## Row limits per BI entity

Bitrix24 limits how many rows the BI-connector returns per BI entity depending on your cloud plan. See the [official plan comparison](https://www.bitrix24.ru/prices/compare_cloud_plans.php) for the up-to-date values.

| Plan | Row limit |
|---|---:|
| Free | 0 |
| Basic | 10 000 |
| Standard | 10 000 |
| Professional | 100 000 |
| Enterprise | 250 000 |
