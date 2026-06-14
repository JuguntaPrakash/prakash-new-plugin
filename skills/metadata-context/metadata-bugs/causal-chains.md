# Metadata Bug Causal Chains

Bugs that caused other bugs — regression chains. These are the most dangerous patterns
because fixing one bug introduced another.

## Chain 1: Date Format Cascade

```
SIGSPM-4300 (2023) "SPM API: date format in attribute of date type on update of dictionary item"
  ↓ fix introduced
SIGSPM-9771 (2023-11) "Dictionary Excel Import is failing for invalid date"
  ↓ fix introduced regression
SIGSPM-12331 (2024-05) "Dictionary entries skipped during import, if date attribute empty"
  → Labels: regression
```

**Pattern**: Each date format fix tightened validation, which broke previously-accepted edge cases.
**Lesson**: Date format changes need exhaustive backward compatibility testing with ALL format variants AND empty/null values.

## Chain 2: GlossaryLink Multi-PR Fix

Referenced in metadata-guide.md as "GlossaryLink merge defect requiring 3 PRs":
- Fix 1: Addressed getSingleValue() path
- Fix 2: Addressed getListValue() path
- Fix 3: Addressed replaceLinks() path

**Pattern**: MetaDataGlossaryLink has 3 independent value paths that must stay consistent.
**Lesson**: When fixing MetaDataGlossaryLink, ALWAYS check all 3 paths in a single PR.

## Chain 3: Dictionary Entry Saving Special Characters

```
SIGSPM-7319 (2023-07) "[SPM-8837] Dictionary: saving of an entry is not possible if certain characters used"
  ↓ caused by related encoding issues
SIGSPM-8644 "Encoding follow-up"
  ↓ incomplete fix
SIGSPM-9589 (2023-11) "saving of an entry is not correct for certain entries"
  ↓ still incomplete
SIGSPM-10266 (2023-12) "saving of an entry is not correct for certain entries"
  ↓ encoding still surfacing
SIGSPM-10512 → caused SIGSPM-17308 (2025-07) "Explorer List View Attribute Column Empty"
```

**Pattern**: HTML encoding in dictionary values propagated through multiple display paths.
**Lesson**: Character encoding fixes must be tested across ALL rendering paths (Editor, Explorer, Hub, Reports, Export).

## Chain 4: VAL Import Stencil/Category Attachment

```
SIGSPM-11512 (2024-03) "VAL: RiskManagement attributes cannot be imported"
SIGSPM-11513 (2024-03) "VAL: Dictionary Link Attributes don't reference a category after import"
SIGSPM-11514 (2024-03) "VAL: Custom attributes on dictionary categories are not attached properly"
  ↓ partial fixes
SIGSPM-11768 (2024-04) "VAL Import: Existing custom attributes are not added to additional stencils & categories"
  ↓ ID mapping issue
SIGSPM-11750 (2024-04) "Import Handlers use target tenant id to regenerate source-sri for id mappings"
```

**Pattern**: VAL import handler fixes for one entity type (risk, dictionary link, category) exposed the same bug in other entity types.
**Lesson**: When fixing VAL import for one metadata type, test ALL metadata types in the same import.

## Chain 5: VAL Reference Extraction

```
SIGSPM-11263 (2024-02) "Invalid SRIs returned to VAL on custom attribute definition reference extraction"
  ↓ fix tightened extraction
SIGSPM-12277 (2024-05) "Reference extraction returns custom attributes bound to category even if no items have values"
  ↓ over-corrected
SIGSPM-12318 (2024-05) "Custom attribute definitions only used on diagram-level are not discovered by reference extraction"
```

**Pattern**: Reference extraction for VAL must balance between: (a) not including too many definitions and (b) not missing diagram-level definitions.
**Lesson**: Reference extraction changes must test both diagram-level and category-level attribute definitions.

## Chain 6: MetaDataManager deleteMetaData OOM

```
SIGSPM-265 (old) "DELETE /meta endpoint leads to OutOfMemory error"
  ↓ partially fixed, but not fully
SIGSPM-4505 (old, duplicate)
  ↓ production incident
SIGSPM-9962 (2023-11) "MetaDataManager.deleteMetaData and High GC Pause (Pod Down Prod CA)"
  → Incident SIGINCIDENT-573, SIGINCIDENT-585
  → Feature flag: ENABLE_METADATA_DELETE_OPTIMISATION
```

**Pattern**: deleteMetaData() loads all related entities eagerly, causing memory explosion for attributes with many values.
**Lesson**: Any changes to MetaDataManager delete logic must be tested with large datasets and monitored for memory usage. Use the feature flag.

## Chain 7: Glossary Performance → Feature Flag → Regression

```
SIGSPM-6519 "Dictionary Performance: update API too many database queries"
  ↓ optimization added behind flag
SIGSPM-8897 "Enable Dictionary Performance Changes as default"
  ↓ flag removal exposed issues
SIGSPM-10890 "[3lvl][performance] /p/meta very slow"
  ↓ new optimization with flag
SIGSPM-11222 "Follow up /p/meta performance: remove LD flag"
  ↓ need for more optimization
SIGSPM-15364 "MVs not visible with perf flag enabled"
```

**Pattern**: Performance optimizations behind feature flags work, but removing the flags (going to default-on) reveals edge cases.
**Lesson**: When removing a metadata performance feature flag, test ALL metadata display paths (Editor, Explorer, Hub, Reports, Dictionary, Glossary).

## Chain 8: Hibernate Binding Loading — 3+ Optimization/Revert Cycles

```
SPM-15744 (2022-07) "LazyInitializationException when loading MetaDataDefinition bindings"
  ↓ fix: use definition.getStencilIds() instead of HQL
SPM-17120 (2022-07) "Regression in model saving from stencil ID optimization"
  ↓ reverted optimization
SIGSPMR-126 (2023-10) "MetaDataDefinition binding loading optimizations"
  ↓ two optimization attempts, both reverted
SIGSPM-8634 (2023-09) "AML upload not possible — LazyInitializationException"
  ↓ fix: changed stencilSetBindings to FetchType.EAGER
```

**Pattern**: Every attempt to optimize MetaDataDefinition binding loading has been reverted. The EAGER fetch solution works but hurts performance.
**Lesson**: DO NOT attempt to optimize MetaDataDefinition binding loading without extensive regression testing across AML import, reports, and model saving.

## Chain 9: VAL Import Entity Ordering and Cross-References

```
SIGSPM-11512 (2024-03) "RiskManagement attributes cannot be imported"
  → Fix: reorder imports (DICTIONARYCATEGORY before METADATADEFINITION)
SIGSPM-11513 (2024-03) "Dictionary Link Attributes don't reference category after import"
  → Fix: read category from S3 source, not target entity
SIGSPM-11514 (2024-03) "Custom attributes on categories not attached during import"
  → Fix: add updateGlossaryBindings() for category ID mapping
SIGSPM-11750 (2024-04) "Source SRI constructed with target tenant ID"
  → Fix: introduce ValEntityData DTO for source SRI tracking
SIGSPM-11768 (2024-04) "Existing custom attributes not added to additional stencils"
  → Fix: update stencil bindings during import
```

**Pattern**: All 5 bugs were in the same VAL import flow within ~6 weeks. Each fix exposed the next gap.
**Lesson**: VAL import must be tested end-to-end with ALL metadata types (risk, dictionary link, glossary link) in a single import. Fixing one type exposes the same bug in others.

## Chain 10: Special Character Handling in MetaDataEntityValue

```
planio #19511 (2017) "Added HTML unescaping to stored values"
  ↓ caused XSS vulnerability
planio #19878 (2017-06) "Reverted HTML unescaping — XSS fix"
  ↓ encoding inconsistencies remained
SIGSPM-7319 (2023-07) "Saving dictionary entry impossible with certain characters"
  ↓ broadened isTypeOfStringOrText() check
(2023-10) "Revert: broadened check broke other attribute types"
  ↓ narrowed back to meta- prefix only
SIGSPM-9589 (2023-11) "Saving entry incorrect for certain entries — nested brackets"
  ↓ fixed substringBetween for nested brackets
SIGSPM-10266 (2023-12) "Saving still incorrect for certain entries"
```

**Pattern**: HTML/special character handling in MetaDataEntityValue has been fixed, broken, and fixed again at least 6 times.
**Lesson**: MetaDataEntityValue encoding is a minefield. NEVER add/remove escaping without testing: `& < > " ' [ ] { } \ &nbsp;` across Editor save, Dictionary save, Export, Import, and Report display.
