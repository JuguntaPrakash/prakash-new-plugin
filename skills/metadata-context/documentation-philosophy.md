# Documentation Philosophy for Memory Banks

Guiding principles for what to document, where to put it, and what to leave out.
These rules keep memory banks useful over time and prevent them from becoming stale noise.

## What goes where

### Class-level javadocs — "I'm about to touch this file"
Rules that apply to ONE specific class. The developer/agent sees them exactly when they open the file.
- Production incident history tied to specific methods
- "NEVER do X" rules for that class (e.g., FetchType must stay EAGER)
- Fix/revert timelines for specific methods
- Non-obvious return type behavior (e.g., getTypedValue() polymorphism)

**Test**: Would someone modifying THIS file need to know this? → javadoc.

### Guide files (e.g., metadata-guide.md) — "I'm working in this area"
Cross-cutting knowledge that spans multiple classes.
- Hard rules learned from production incidents
- Implicit contracts (compiles fine, causes bugs)
- Debugging quick reference (symptom → where to look)
- Shadow bug blast radius (which consumer domains to test)
- Test coverage gaps
- Bug cluster taxonomy

**Test**: Does this involve 2+ classes or cross-cutting concerns? → guide.

### Raw database files (e.g., bug-database.json) — "I need to do deep analysis"
Machine-readable data too detailed for a guide but valuable for systematic analysis.
- Full bug entries with diffs, root causes, lessons
- Regression chain timelines
- Shadow bug inventories
- Co-change statistics

**Test**: Is this raw data that supports the guide's claims? → database.

## What NOT to document

These things are either discoverable from code, change too fast, or are opinions not facts:

| Skip | Why |
|------|-----|
| Type hierarchy, method signatures | Agent reads the code directly |
| Hibernate mappings, DB schema | Discoverable via annotations and migrations |
| What tests cover (happy path) | Agent runs the tests directly |
| Current feature flag states | Changes over time → immediately stale |
| Architectural opinions ("should be split") | That's a decision, not a fact — belongs in ADRs |
| Team process/workflow | Belongs in team docs, not code |
| Anything duplicating CLAUDE.md or AGENTS.md | One source of truth, not two |

## What TO document (things agents can't discover alone)

| Document | Why |
|----------|-----|
| Fix/revert cycles | Agent would need to correlate 6+ commits across years |
| Implicit contracts | Compiles fine, no interface enforces them |
| Regression chains | Pattern spans multiple JIRA tickets over months |
| Shadow bugs in other packages | Agent won't search 200+ consumer files for related bugs |
| Test coverage GAPS (not coverage) | Absence is invisible — agent sees tests that exist, not tests that don't |
| "We tried X and it failed" history | Agent might try the same optimization that was reverted 3 times |

## Keeping memory banks clean

1. **Every claim needs a source** — JIRA ticket, commit SHA, or incident ID. No unsourced opinions.
2. **Date-stamp volatile facts** — If something might change (line counts, bug counts), note when it was measured.
3. **Delete when proven wrong** — If a "NEVER do X" rule turns out to be wrong, remove it immediately. Wrong rules are worse than no rules.
4. **Guide summarizes, database proves** — The guide should be readable in 5 minutes. The database backs it up with evidence.
5. **Update on every significant metadata change** — If a PR touches metadata, check if the guide needs updating.
6. **Prefer specificity over generality** — "SIGSPM-9962 caused production OOM" is better than "delete can be slow."
