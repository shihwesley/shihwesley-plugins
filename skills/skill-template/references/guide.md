# Skill Template Guide

Reference for writing skills that comply with Anthropic's official skill guide (Jan 2026).

## Progressive Disclosure — The Three Levels

Skills load in layers to minimize token usage:

1. **Frontmatter (always loaded)** — name + description appear in Claude's system prompt. This is how Claude decides whether to load your skill. Get this right or your skill never fires.
2. **SKILL.md body (loaded on demand)** — When Claude decides to use your skill, the full instructions load. Keep this focused and under 5,000 words.
3. **references/ files (loaded as needed)** — Detailed documentation, API patterns, examples that Claude discovers and reads only when relevant.

## Frontmatter Checklist

### name (required)
- kebab-case only: `my-cool-skill` not `My Cool Skill`
- Must match the folder name exactly
- No "claude" or "anthropic" in the name (reserved)

### description (required)
Formula: `[What it does] + [When to use it] + [Key capabilities]`

Must include:
- What the skill does (1 sentence)
- Trigger phrases — actual words users would type
- Key capabilities (what it handles)
- Optional: negative triggers (what it should NOT fire for)

Constraints:
- Under 1024 characters
- No XML angle brackets (< or >)

### Good descriptions

```yaml
# Specific, actionable, has triggers
description: "Analyze Figma design files and generate developer handoff docs. Use when user uploads .fig files, asks for design specs, component documentation, or design-to-code handoff."

# Clear scope with negative triggers
description: "Advanced data analysis for CSV files. Use for statistical modeling, regression, clustering. Do NOT use for simple data exploration (use data-viz skill instead)."
```

### Bad descriptions

```yaml
# Too vague — Claude can't route to this
description: "Helps with projects."

# Missing triggers — what would a user say?
description: "Creates sophisticated multi-page documentation systems."

# Too technical, no user language
description: "Implements the Project entity model with hierarchical relationships."
```

## Body Best Practices

1. **Be specific** — "Run `python scripts/validate.py --input {filename}`" not "Validate the data"
2. **Instructions at the top** — most important stuff first, use ## Important or ## Critical headers
3. **Bullet points over prose** — easier for Claude to follow
4. **Include examples** — at least one common scenario showing trigger → actions → result
5. **Include error handling** — common issues with cause and fix
6. **Reference linked files explicitly** — "Before writing queries, consult `references/api-patterns.md` for rate limiting and pagination patterns"

## Folder Structure

```
your-skill-name/
├── SKILL.md                    # Required — main instructions
├── scripts/                    # Optional — executable code
│   ├── process_data.py
│   └── validate.sh
├── references/                 # Optional — detailed docs
│   ├── api-guide.md
│   └── examples/
└── assets/                     # Optional — templates, icons
    └── report-template.md
```

Rules:
- SKILL.md must be exactly `SKILL.md` (case-sensitive)
- Folder name must be kebab-case
- No README.md inside the skill folder (docs go in SKILL.md or references/)
- Keep SKILL.md under 5,000 words — move detailed content to references/

## Testing Your Skill

### 1. Triggering tests
Does it load at the right times?
- Test with obvious phrases ("help me [do the thing]")
- Test with paraphrased requests
- Test that it doesn't fire on unrelated topics

### 2. Functional tests
Does it produce correct output?
- Valid outputs generated
- Error cases handled
- Edge cases covered

### 3. Debug technique
Ask Claude: "When would you use the [skill name] skill?"
Claude will quote the description back. If it misses key triggers, revise the description.
