# wp-plugin-standards-claude-skills

![Claude Code](https://img.shields.io/badge/Claude%20Code-Skill-blueviolet) ![WordPress](https://img.shields.io/badge/WordPress-6.4%2B-blue) ![PHP](https://img.shields.io/badge/PHP-7.4%2B-purple) ![License](https://img.shields.io/badge/License-GPLv2-green)

A Claude Code skill that enforces WordPress.org plugin submission standards, security hardening, code quality, and documentation discipline.

Point Claude at this skill before touching any WordPress plugin and get a structured findings report — grouped by severity — before a single file is changed.

## What it does

Every session begins with a mandatory review report grouped by Critical, High, Medium, and Low severity before any file is touched. You confirm which issues to fix, then the skill works through a structured five-step workflow covering security, code reuse, PCP compliance, version tracking, and changelog discipline.

The skill covers the most common WordPress.org rejection reasons: echoed `<script>` and `<style>` tags, missing nonces, unescaped output, capability gaps, duplicate helpers, version string mismatches, and ownership mismatches.

## Installation

Clone this repository to any path your Claude Code session can read:

```bash
git clone https://github.com/andrewbakercloudscale/wp-plugin-standards-claude-skills.git
```

No dependencies. No build step. The skill is plain Markdown.

## Usage

Point Claude Code at the skill file before working on any WordPress plugin:

```
Read /path/to/wp-plugin-standards-claude-skills/SKILL.md then review my plugin at /path/to/my-plugin
```

Claude will produce a findings report and wait for your confirmation before touching anything.

### Example session

```
Read ~/skills/wp-plugin-standards-claude-skills/SKILL.md then review my plugin at ~/plugins/my-plugin

> WordPress Plugin Standards — Review
>
> Critical — blocks WordPress.org submission
> - admin/class-my-plugin-admin.php:47   echo '<script>...' — echoed script tag
>
> High — security or data integrity risk
> - includes/class-my-plugin.php:112     Missing nonce verification before POST processing
>
> Medium — code quality, documentation, reuse
> - includes/class-my-plugin-utils.php   Missing @return tag on get_settings()
>
> 1 critical · 1 high · 1 medium · 0 low issues found.
> Confirm to proceed with all fixes, or specify which severity levels to address.

Fix critical and high.
```

## Workflow

| Step | Action |
|------|--------|
| 0 | Review report — always first, never skipped. Grouped by Critical / High / Medium / Low. |
| 1 | Check Utils — before writing any new function, confirm it does not already exist in `class-SLUG-utils.php`. |
| 2 | Apply fixes — address the confirmed severity levels using the reference files. |
| 3 | PCP checklist — step through `references/pcp-checklist.md` and confirm zero violations. |
| 4 | Bump versions — update plugin header `Version:`, `VERSION` constant, and `readme.txt` `Stable tag:` in one pass. |
| 5 | Update CHANGELOG.md — add a dated entry for every change made. |

## Files

```
SKILL.md                          Main skill file — load this in Claude Code
references/
├── security.md                   Nonce, sanitisation, escaping, DB, capability, filesystem rules
├── coding-standards.md           Naming, DocBlocks, formatting, Yoda conditions, i18n, error handling
├── performance.md                Asset enqueuing, DB queries, transients, Options API, background tasks
├── accessibility.md              WCAG 2.1 AA, labels, ARIA, keyboard, colour contrast, modals
├── reuse.md                      Utils class, version tracking, CHANGELOG, readme.txt, uninstall
└── pcp-checklist.md              Full WordPress.org Plugin Check compliance checklist
```

## Reference files

| File | Covers |
|------|--------|
| `references/security.md` | Nonce verification, input sanitisation, output escaping, DB queries, capability checks, filesystem access |
| `references/coding-standards.md` | Naming conventions, DocBlocks, file formatting, Yoda conditions, i18n, error handling |
| `references/performance.md` | Asset enqueuing (the most common rejection reason), DB query optimisation, transients, Options API, background processing |
| `references/accessibility.md` | WCAG 2.1 AA, labels, ARIA attributes, keyboard navigation, colour contrast, screen reader support, modals |
| `references/reuse.md` | Utils class pattern, version tracking, CHANGELOG format, readme.txt format, uninstall cleanup |
| `references/pcp-checklist.md` | Full WordPress.org Plugin Check compliance checklist — 60+ line items across security, assets, i18n, documentation, and submission requirements |

## Severity definitions

| Severity | Examples |
|----------|---------|
| Critical | Echoed `<script>` or `<style>` tags, missing nonce, unescaped output, raw SQL, WordPress.org ownership or contributor mismatch |
| High | Missing capability check, unsanitised input, bare `die()`, hardcoded URLs, missing ABSPATH guard |
| Medium | Duplicate helper functions, missing DocBlocks, version string mismatch, missing CHANGELOG entry, global asset enqueue |
| Low | Naming convention violations, missing inline comments, non-autoloaded options, minor i18n issues |

## Requirements

- Claude Code (any current version)
- WordPress 6.4+ target
- PHP 7.4+ target (8.0+ recommended)

## Companion skill

Pair with [wp-plugin-development](https://github.com/WordPress/agent-skills) for full plugin scaffolding and development coverage. This skill is structured to match the WordPress/agent-skills conventions for potential upstream contribution.

## Licence

GPLv2 or later — https://www.gnu.org/licenses/gpl-2.0.html
