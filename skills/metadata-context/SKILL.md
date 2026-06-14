---
name: metadata-context
description: >-
  Metadata package regression history (290 JIRA bugs, 10 documented regression chains). ALWAYS invoke this skill when
  editing files in com.signavio.metadata package, MetaDataEntityValue, MetaDataDefinitionInfo,
  MetaDataGlossaryItemRevisionValue, MetaDataManager, or any metadata data/business class.
  Do not edit metadata package files directly — invoke this skill first to load incident
  history and trace regression chains before writing any fix code.
allowed-tools: Read, Glob, Grep
---

# Metadata Package Context

290 JIRA bugs, 58 with detailed root cause analysis, 10 documented regression chains. High regression rate — fixes frequently break other rendering paths weeks later.

## Checklist

Copy this and check off each item before committing:

```
Metadata Fix Safety Checklist:
- [ ] 1. Read incident history (metadata-guide.md + causal-chains.md)
- [ ] 2. Named the most similar past bug and how my fix differs
- [ ] 3. Found ALL callers of the method I'm changing
- [ ] 4. Traced data flow through all rendering paths
- [ ] 5. Tested interaction with isValueEqual() if touching values
- [ ] 6. Tested with already-stored database values (lazy migration)
- [ ] 7. Stated blast radius and what this does NOT fix
```

## Step 1: Read the history

- [metadata-guide.md](metadata-guide.md) — 4 key classes, NEVER rules, co-change partners
- [metadata-bugs/causal-chains.md](metadata-bugs/causal-chains.md) — 10 regression chains (fix A → bug B)

For deeper investigation:
- [metadata-bugs/bug-database.json](metadata-bugs/bug-database.json) — 58 bugs with diffs
- [metadata-bugs/bug-patterns.md](metadata-bugs/bug-patterns.md) — recurring patterns
- [metadata-bugs/shadow-bugs.md](metadata-bugs/shadow-bugs.md) — 63 confirmed shadow bugs

## Step 2: Name the most similar past bug

**You must name** a past bug (JIRA ID) and state:
- What was the original fix?
- Why did it regress?
- How is your fix different?

## Step 3: Trace callers and data flow

Find all callers of the method you are changing. Specifically check these hot methods if your change touches them: `isValueEqual()`, `getTypedValue()`, `cleanupValue()`, `setValue()`, `getReadableValue()`.

List which rendering paths your test covers and which it does not: Editor save, Dictionary save, Export, Import, Report, Hub, Explorer.

## Step 4: Write regression-aware tests

Test not just that your fix works, but also:
- Interaction with `isValueEqual()` (HTML unescaping comparison)
- Values already stored in the database before your fix
- At least one consumer path beyond the one you fixed

## Step 5: State risk assessment

- Blast radius — which consumer domains are affected?
- What this does NOT fix — be honest about limitations
- Can it be rolled back via feature flag?

## Example: Good vs bad metadata fix

**Bad fix** (SIGSPM-15923 pattern — 3 agents produced this independently):
```java
// "Just replace the bad characters" — fixes ONE path, ignores the encoding chain
value = value.replace("&nbsp;", " ").replace("\u00A0", " ");
```
Why bad: Doesn't address the double-encoding root cause (ParseParametersFilter → unescapeHtml), breaks `isValueEqual()` for already-stored values, no feature flag.

**Good fix** (what the 4th agent produced after tracing the data flow):
```java
// Fix at the SOURCE — where the double-encoding happens, not where it's stored
value = value.replace("&nbsp;", "\u00A0"); // in GlossaryItemJsonParser
```
Why good: Fixes the encoding mismatch at the entry point, preserves `isValueEqual()` semantics, doesn't touch stored values.
