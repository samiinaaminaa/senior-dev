# senior-dev

A Claude Code skill: engineering rules for shipping features that stay correct under retries, races, and partial failure. Strictest around payments.

- [`SKILL.md`](SKILL.md): core rules, loaded when the task touches money, accounts, data writes, jobs, webhooks, or deployment
- [`references/payments.md`](references/payments.md): read before writing any payment code
- [`references/preship-checklist.md`](references/preship-checklist.md): run before shipping

Install:

```bash
git clone https://github.com/samiinaaminaa/senior-dev ~/.claude/skills/senior-dev
```
