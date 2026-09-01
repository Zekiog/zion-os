# Contributing to ZIONOS

ZIONOS is the strategic reference for the Zeki ecosystem. It defines the
contracts between layers, not the implementations. Implementations live in
their respective repos (`zion-aid-v2`, `memory-core-mcp`, etc.).

## When to contribute here

- A new **connection spec** between two layers
- A new **entity** in the semantic layer catalog
- A new **cross-cutting integration** (e.g., new compliance standard)
- A correction to the **roadmap** or **architecture**
- A new **vertical application** that maps to Layer 4

## When NOT to contribute here

- Feature work in a specific component repo → file the issue there
- Bug reports in an existing component → file the issue there
- Documentation for a specific component → that component's README

## Pull request template

When opening a PR against this repo, please include:

1. **What layer(s) does this affect?** (L1–L6)
2. **What components does this touch?**
3. **Does this add a new connection spec?** If so, fill in Motivation, Contract, Reference Implementation, Adoption Checklist.
4. **Does this change the roadmap?** If so, link the relevant issue.
5. **Evidence:** Cite at least one source (research paper, production postmortem, RFC).

## Style

- Concise, technical, direct. No marketing language.
- Markdown only.
- Diagrams in fenced code blocks (Mermaid preferred for live rendering).
