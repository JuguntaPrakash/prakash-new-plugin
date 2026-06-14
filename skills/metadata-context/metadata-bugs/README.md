# Metadata Bug Database

Historical database of all metadata-related bugs, their root causes, fix diffs, and patterns.
Used by AI agents to prevent recurring bugs when working on the metadata component.

## Structure

- `bug-database.json` — Machine-readable database of 58 bugs with diffs, timestamps, root causes
- `bug-patterns.md` — 12 categorized bug patterns with prevention checklists
- `causal-chains.md` — 10 regression chains (bugs that caused other bugs)
- `shadow-bugs.md` — ~63 confirmed + 459 candidate bugs found OUTSIDE the metadata package

## Sources

- **JIRA**: 290 total metadata-related bugs found (101 fixed with code changes)
- **JIRA symptom search**: 459 additional bugs describing metadata symptoms without mentioning "metadata"
- **JIRA link graph**: 41 shadow bugs linked from known metadata bugs
- **Git**: 873 commits touching `com.signavio.metadata.*` package
- **Git**: 1224 commits touching `com.signavio.attribute.*` package
- **Import graph**: 200+ consumer files, 22 confirmed shadow bugs in 7 categories
- **Co-change analysis**: Top 20 files that statistically change with metadata, with specific bug patterns
- **Diff analysis**: Bug fixes identified from commit messages + diff patterns (DIFF-SCAN-1 through 18)

## How to Use

When modifying metadata code, search this database for:
1. Similar past bugs in the same file/method (check `bug-database.json`)
2. Known regression chains — fix A caused bug B (check `causal-chains.md`)
3. Common bug patterns with prevention checklists (check `bug-patterns.md`)
4. Shadow bugs in consumer packages (check `shadow-bugs.md`)

## Quick Reference: Most Dangerous Areas

| Area | Why | File to Check |
|------|-----|---------------|
| Encoding/escaping | 6+ fix/revert cycles on MetaDataEntityValue | `bug-patterns.md` Pattern 1 |
| Date formats | Regression chain 4300→9771→12331→17308 | `causal-chains.md` Chain 1 |
| VAL import | 5 bugs in 6 weeks, each fix exposed next gap | `causal-chains.md` Chain 9 |
| Hibernate loading | 3+ optimization/revert cycles | `causal-chains.md` Chain 8 |
| Delete ordering | Production OOM (SIGINCIDENT-573) | `bug-patterns.md` Pattern 10 |
| List vs single value | 3 value paths in MetaDataGlossaryLink | `bug-patterns.md` Pattern 4 |
