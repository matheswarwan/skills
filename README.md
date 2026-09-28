# skills

Reference skills for Claude on three Salesforce topics: Data Cloud architecture, Salesforce Personalization, and Salesforce app development and packaging. They are written knowledge bases (Markdown), not code. Claude reads them to answer questions and plan implementations.

The three skills are stored in three different shapes. Only one is ready to drop into Claude Code as-is. See [Installing in Claude Code](#installing-in-claude-code).

## What is here

### `salesforce-data-cloud-skill.md`

A single-file "Data Cloud Architect's Skill" (about 620 lines). It opens with a "When to use this skill" list, then covers:

- platform fundamentals and the data model layers (DSO, DLO, DMO)
- data ingress: web and mobile SDKs, Salesforce and external connectors, Ingestion API, zero-copy (BYOL) federation
- harmonization, fully qualified keys, formula fields and data transforms
- identity resolution: match and reconciliation rules, unified link objects, anonymous to known matching
- calculated and streaming insights, segmentation, activations and data actions
- real-time data, vector search and RAG, Einstein Studio models, consent
- CRM integration, consumption and pricing, sandboxes and deployment, architect methodology, data egress
- a glossary

Format: plain Markdown with no YAML frontmatter, and not named `SKILL.md`.

### `salesforce-personalization/`

A skill for Salesforce Personalization, the Data Cloud-native product (formerly Einstein Personalization). Its description says it does not cover the legacy Marketing Cloud Personalization (Evergage/Interaction Studio).

`skill.md` has YAML frontmatter (`name`, `description` with trigger phrases) and 16 sections: setup, the Interactions SDK, sitemap configuration, DLO-DMO mapping and identity resolution, data graphs, calculated insights, response templates, recommenders, personalization points, decisions and experiments, Web Personalization Manager, SDK and Decisioning API requests, mobile, and Marketing Cloud / Journey Builder activation. It ends with common pitfalls.

The `references/` folder holds the detail that `skill.md` points to:

| File | Content |
| --- | --- |
| `setup.md` | Setup checklist, permission sets, datakit deploy |
| `sitemap.md` | Sitemap patterns, page types, content zones, engagement destinations |
| `ci-patterns.md` | Calculated insight SQL templates (affinity, recency, lifetime value) |
| `api.md` | Decisioning API: auth flow, request and response examples |
| `mobile.md` | iOS and Android SDK setup and fetch patterns |
| `scoping-template.md` | A scoping document template for implementation projects |

Format: a skill folder, but the main file is lowercase `skill.md`.

### `salesforce-packaging-guide/`

A "Salesforce App Development: Best Practices" skill, built from the Salesforce ISVforce (packaging) guide, Spring '26. It covers 2GP vs 1GP packaging, security review requirements (CRUD/FLS, sharing, injection, XSS, secrets), multi-edition design, org and environment management, connected apps and external client apps, the AppExchange security review checklist, and Agentforce/AI security.

| File | What it is |
| --- | --- |
| `salesforce-best-practices.skill` | A zip archive of a complete skill folder: `salesforce-best-practices/SKILL.md` plus `references/packaging.md` and `references/security-deep-dive.md`. |
| `salesforce-best-practices.md` | A readable single-file copy: the main skill body followed by both reference files. It has no frontmatter. |
| `README.md` | Notes the source PDF. |

## Installing in Claude Code

Claude Code loads personal skills from `~/.claude/skills/<name>/SKILL.md` (or project skills from `.claude/skills/<name>/SKILL.md`). The file must be named `SKILL.md` and start with YAML frontmatter that has at least `name` and `description`. Supporting files can sit next to it.

**Packaging guide (ready to use).** Unzip the archive into your skills folder:

```sh
mkdir -p ~/.claude/skills
unzip salesforce-packaging-guide/salesforce-best-practices.skill -d ~/.claude/skills/
# creates ~/.claude/skills/salesforce-best-practices/SKILL.md and references/
```

**Personalization (rename the main file).** Copy the folder and rename `skill.md` to `SKILL.md`. On case-sensitive file systems (Linux) the lowercase name is not picked up.

```sh
cp -r salesforce-personalization ~/.claude/skills/
mv ~/.claude/skills/salesforce-personalization/skill.md ~/.claude/skills/salesforce-personalization/SKILL.md
```

**Data Cloud (add frontmatter).** Create a folder, copy the file in as `SKILL.md`, and add frontmatter at the top. For example:

```sh
mkdir -p ~/.claude/skills/salesforce-data-cloud
cp salesforce-data-cloud-skill.md ~/.claude/skills/salesforce-data-cloud/SKILL.md
```

Then add these lines at the very top of that `SKILL.md`:

```yaml
---
name: salesforce-data-cloud
description: Salesforce Data Cloud architecture reference. Use when designing or reviewing Data Cloud ingestion, harmonization, identity resolution, insights, segmentation, activation, or consumption.
---
```

Start a new Claude Code session afterwards so the skills are discovered.

## Limitations

- The content is a summary of Salesforce documentation at a point in time (early 2026). Product names, limits and pricing change with each release. Check the current Salesforce docs before relying on a number.
- The three skills use three different layouts. Only the `.skill` archive is in the standard layout.
- `salesforce-best-practices.md` refers to `references/packaging.md` and `references/security-deep-dive.md`, which exist only inside the `.skill` archive. Their content is appended to the same `.md` file.
- The API and SDK examples use placeholder values (`{token}`, `customer@example.com`, `YOUR_ACCOUNT_NAME`). They are not tested code.

## Authors

All commits are by Matheswaran Kanagarajan.
