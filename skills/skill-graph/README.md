# Skill Graphs

MOC-based navigation for agent-consumed domain knowledge. When you accumulate enough skills, docs, and agents for a domain, agents need structure to find the right file without scanning everything.

## What's here

- `create-skill-graph.md` — The `/create-skill-graph` command. Works from neo-research output (question tree branches become MOCs) or from existing files (inventory and cluster by topic).
- `traverse-template.md` — Template for the traversal protocol that gets included in every graph. Teaches agents the 3-read progressive disclosure pattern.

## How it works

1. neo-research runs `/research <topic>` and builds a question tree
2. During the question tree phase, it assesses coupling — do sub-topics reference each other?
3. If coupling is high (3+ indicators), the research report recommends `/create-skill-graph`
4. The command reads the question tree, derives 5-10 MOCs, builds the graph, wires enforcement into agents and the skill registry
5. Agents navigate: index → MOC → target file (3 reads, ~3-4k tokens instead of ~25k+)

## When to use it

Use a skill graph when:
- You have 15+ skills/docs/agents for a domain
- Sub-topics reference each other (learning A requires knowing about B)
- Agents keep loading wrong files or spending tokens on irrelevant docs

Skip it when:
- You have a handful of independent skills with no overlap
- The domain is narrow enough that one expertise doc covers everything

## Inspired by

[arscontexta](https://github.com/agenticnotetaking/arscontexta) — a Claude Code plugin that builds Zettelkasten-style knowledge systems using Niklas Luhmann's slip-box methodology. We adapted their MOC structure for agent navigation instead of personal knowledge management.
