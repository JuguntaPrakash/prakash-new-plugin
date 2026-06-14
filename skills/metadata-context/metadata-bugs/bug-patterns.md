# Metadata Bug Patterns

Categorized patterns extracted from the metadata bug database.
When modifying metadata-related code, check if your change falls into one of these known bug clusters.

## Pattern 1: Encoding / Escaping (HIGH FREQUENCY)

**Bugs**: SIGSPM-18377, SIGSPM-17972, SIGSPM-19324, SIGSPM-16027, SIGSPM-15923, SIGSPM-14561, SIGSPM-13898, SIGSPM-8759, SIGSPM-7905, SIGSPM-7842, SIGSPM-6940, SIGSPM-4393

**Root cause pattern**: Mixing HTML escaping and JSON/URL escaping. Data passes through multiple encoding layers (browser → REST handler → JSON parser → database → render) and each layer may double-encode or incorrectly unescape.

**Key lessons**:
- NEVER call `StringEscapeUtils.unescapeHtml()` on JSON data — JSON has its own escape rules
- URL redirect chains must handle HTML entity escaping at each hop; validate unescaped URLs with `RedirectValidator`
- `&amp;`, `&quot;`, `&nbsp;` in attribute values are persistent encoding bugs that surface across multiple systems
- Backslashes in URL attributes need special handling

**Prevention checklist**:
- [ ] Does your change introduce or remove any escaping/unescaping?
- [ ] Test with values containing: `&`, `"`, `<`, `>`, `\`, `[`, `]`, `&nbsp;`, newlines
- [ ] Test the full render chain: save → load → display → export → import

---

## Pattern 2: Date Format Handling (HIGH IMPACT)

**Bugs**: SIGSPM-17308, SIGSPM-12331, SIGSPM-11844, SIGSPM-11255, SIGSPM-10200, SIGSPM-9771, SIGSPM-9299, SIGSPM-7081, SIGSPM-4300, SIGSPM-633, SIGSPM-1589, SIGSPM-15411

**Root cause pattern**: `MetaDataDate` format list is GLOBAL shared state. Changes affect ALL tenants and ALL existing data. Multiple format strings exist (`d/m/y`, `d.m.y`, `d-m-y`, `m/d/y`, `y-m-d`) and validation must accept all of them.

**Key lessons**:
- MetaDataDate.validateSingleValue() must try ALL known formats, not just the configured one
- Date formats in dictionary imports may differ from diagram date formats
- Empty dates in Excel imports must not cause the entire import to fail (SIGSPM-12331 regression from SIGSPM-9771/4300)
- Date display in Explorer table view depends on format stored at creation time

**Regression chain**: SIGSPM-4300 (date format in API) → SIGSPM-9771 (Excel import fails for invalid date) → SIGSPM-12331 (dictionary entries skipped if date empty)

**Prevention checklist**:
- [ ] Always wrap date format changes behind a feature flag
- [ ] Test backward compatibility with ALL existing format strings
- [ ] Test empty/null date values in import paths
- [ ] Test d/m/y vs m/d/y ambiguity (dates like 03/04/2025)

---

## Pattern 3: VAL Import / Export (HIGH FREQUENCY)

**Bugs**: SIGSPM-19543, SIGSPM-16957, SIGSPM-16595, SIGSPM-15976, SIGSPM-15746, SIGSPM-14067, SIGSPM-13921, SIGSPM-13909, SIGSPM-13778, SIGSPM-12381, SIGSPM-12338, SIGSPM-12318, SIGSPM-12277, SIGSPM-11917, SIGSPM-11768, SIGSPM-11750, SIGSPM-11592, SIGSPM-11514, SIGSPM-11513, SIGSPM-11512, SIGSPM-11263

**Root cause pattern**: VAL (Value Accelerator Library) import/export involves complex entity mapping between workspaces. Metadata definitions, dictionary categories, and values must all be mapped consistently. Failures in one mapping cascade to dependent entities.

**Key lessons**:
- Entity matching must consider ALL properties (name, type, isList, configuration) — not just name+type
- ID mappings can become stale when definitions are deleted and recreated with same name but different type
- Locale handling: never use "undefined" as a locale key; always resolve to actual locale
- Single invalid metadata value should never abort entire import — log warnings and continue
- Missing enum/dropdown items in target workspace should be handled gracefully
- Custom attribute descriptions near length limits cause import failures (SIGSPM-13778)
- Reference extraction must discover ALL custom attribute definitions, including diagram-level ones

**Prevention checklist**:
- [ ] Test import with: different types, missing definitions, changed types, deleted+recreated definitions
- [ ] Test with list vs non-list attributes of same name
- [ ] Test with multilanguage workspaces
- [ ] Test with max-length strings for all fields
- [ ] Verify ID mapping consistency across full import chain

---

## Pattern 4: List vs Single Value Confusion

**Bugs**: SIGSPM-16595, SIGSPM-11592, SIGSPM-6827, SIGSPM-5196, SIGSPM-3039, SIGSPM-4320

**Root cause pattern**: MetaDataDefinitionInfo has `isList` flag that changes how values are stored, retrieved, and validated. Code paths for list values and single values often diverge and bugs appear when only one path is fixed.

**Key lessons**:
- MetaDataGlossaryLink has 3 value paths: getSingleValue(), getListValue(), replaceLinks() — fix one, check the other two
- isEmpty() edge cases: `[]` may be treated as "no change" by cleanupValue()
- List attributes are exported even when empty (SIGSPM-11592)
- Single/list mismatch causes VAL import failures (SIGSPM-16595)

**Prevention checklist**:
- [ ] When fixing a value retrieval bug, check getSingleValue(), getListValue(), AND replaceLinks()
- [ ] Test with: empty lists `[]`, single-element lists `["x"]`, null values, and actual multi-value lists
- [ ] Verify import/export handles both list and single value correctly

---

## Pattern 5: Multilanguage / i18n (24% of open bugs)

**Bugs**: SIGSPM-15757, SIGSPM-13766, SIGSPM-13515, SIGSPM-12914, SIGSPM-12721, SIGSPM-12334, SIGSPM-7833, SIGSPM-7195, SIGSPM-6423, SIGSPM-5993, SIGSPM-4888, SIGSPM-4193, SIGSPM-2954

**Root cause pattern**: Attribute values and labels must be stored and retrieved per-locale. Fallback chains (user locale → workspace default → English → any) are complex and bugs appear when any link in the chain is missing.

**Key lessons**:
- Default values are always populated in English regardless of workspace language
- Translation fallback: `retrieveTranslationWithFallbackToEnglish()` has specific precedence rules
- Merge operations must preserve per-locale values
- Reports often show only main language for multilanguage attributes

**Prevention checklist**:
- [ ] Test with multilanguage workspace (at least 3 languages)
- [ ] Test fallback chain: value exists in requested locale, only in default locale, only in English, in no locale
- [ ] Verify reports render correct locale

---

## Pattern 6: Permission / Security

**Bugs**: SIGSPM-19677, SIGSPM-10215, SIGSPM-8432, SIGSPM-8434, SIGSPM-6623, SIGSPM-35

**Root cause pattern**: Metadata endpoints lack proper privilege checks, or use overly-restrictive lookups that fail for non-admin users.

**Key lessons**:
- Use findPublicUserDto() not findUserDto() in data accessors (avoids InsufficientPrivilegesException)
- MetaData creation/deletion requires proper modeling language privileges
- Custom table attributes had XSS vulnerability (SIGSPM-35, SIGSPM-8434)
- Stencilset modification must check admin privileges

**Prevention checklist**:
- [ ] Test all metadata CRUD operations as non-admin user
- [ ] Test cross-tenant access scenarios
- [ ] Sanitize ALL user input in attribute values (XSS prevention)

---

## Pattern 7: Performance / Memory

**Bugs**: SIGSPM-10890, SIGSPM-9962, SIGSPM-15771, SIGSPM-2172, SIGSPM-6519, SIGSPM-265

**Root cause pattern**: MetaDataManager methods load excessive data. deleteMetaData() caused OOM/High GC in production. MetaDataBinding EAGER loading slows model saves.

**Key lessons**:
- MetaDataManager.deleteMetaData() triggered High GC Pause / Pod Down in production (SIGSPM-9962)
- ENABLE_METADATA_DELETE_OPTIMISATION feature flag exists to control delete behavior
- /p/meta endpoint was very slow (SIGSPM-10890), fixed with caching
- SGX and Dictionary Excel export extremely slow for large custom attribute value sets
- MetaDataBinding loading on every model save is wasteful

**Prevention checklist**:
- [ ] Profile any new metadata queries for N+1 patterns
- [ ] Test delete operations with large datasets (>1000 values)
- [ ] Check MetaDataByStencilSetRequestCache usage for new query paths

---

## Pattern 8: Dictionary Link / GlossaryLink Consistency

**Bugs**: SIGSPM-19331, SIGSPM-16948, SIGSPM-15999, SIGSPM-15662, SIGSPM-11288, SIGSPM-8069, SIGSPM-7062, SIGSPM-7021

**Root cause pattern**: Dictionary/glossary links stored in attributes reference entity IDs that can become stale when entries are merged, deleted, or moved. The link resolution must handle all these lifecycle events.

**Key lessons**:
- Merged dictionary items: API returns old item ID instead of merged target ID
- Deleted dictionary items still show in "Show Usages"
- Cross-tenant validation for dictionary links is missing
- Restricted dictionary items can still be linked from backend
- Broken glossary attribute links cause NavMap navigation failures

**Prevention checklist**:
- [ ] Test with: merged items, deleted items, cross-tenant items, restricted items
- [ ] Verify replaceLinks() updates references during merge
- [ ] Check both diagram-level and element-level dictionary links

---

## Pattern 9: Hibernate Lazy Loading + Request Cache (RECURRING — 3+ REVERTS)

**Bugs**: DIFF-SCAN-1, DIFF-SCAN-2, DIFF-SCAN-3, SIGSPM-8634, SIGSPM-8251

**Root cause pattern**: `MetaDataDefinitionInfo` is cached in `MetaDataByStencilSetRequestCache` (request-scoped). When the Hibernate session is cleared during the same request, lazy-loaded collections (`stencilSetBindings`, `glossaryBindings`, `translations`) become inaccessible, throwing `LazyInitializationException`.

**Key lessons**:
- `stencilSetBindings` and `glossaryBindings` MUST be `FetchType.EAGER` — this has been fixed and reverted at least 3 times
- `translations` collection must be explicitly initialized before session clear in report generators
- Manual HQL queries that bypass Hibernate's initialization tracking are dangerous
- Optimizations to MetaDataDefinition binding loading have been attempted and reverted 3+ times

**Prevention checklist**:
- [ ] Never change FetchType on MetaDataDefinitionInfo collections without extensive testing
- [ ] Before clearing Hibernate session, iterate any lazy collections you'll need later
- [ ] Test AML import, risk reports, and Excel export after any metadata loading changes

---

## Pattern 10: Delete Ordering with FK Constraints (PRODUCTION OUTAGE)

**Bugs**: SIGSPM-9962, DIFF-SCAN-10, DIFF-SCAN-12

**Root cause pattern**: MetaDataDefinitionInfo has deep FK relationship graph. Deleting in wrong order causes either FK violations or Hibernate cascade loading OOM.

**Correct delete order**:
1. `GlossaryItemToItemLinkInfo2` (SQL DELETE with JOIN — references MetaDataGlossaryValue4)
2. `MetaDataGlossaryItemRevisionValue` (HQL bulk DELETE)
3. `session.delete(MetaDataDefinitionInfo)` — THEN removeBindingsAndValues() AFTER
4. When clearing item-to-item links: session.delete(each link) BEFORE collection.clear()

**Prevention checklist**:
- [ ] Never use session.delete() alone on MetaDataDefinitionInfo — use bulk pre-deletes
- [ ] Test deleting attributes with >1000 glossary values
- [ ] collection.clear() does NOT delete from DB — always session.delete() each element first

---

## Pattern 11: Validation Strategy Contract Violations

**Bugs**: DIFF-SCAN-14, DIFF-SCAN-13

**Root cause pattern**: Validation strategies inconsistently return `false` vs throw `MetaDataValueException`. Exception mappers may not be Guice-bound. Result: 500 instead of 400 on validation errors.

**Prevention checklist**:
- [ ] All validation strategies must follow the same contract (return boolean OR throw)
- [ ] Verify ExceptionMapper is Guice-bound in the relevant module
- [ ] Check `javax.inject.Provider` vs `javax.ws.rs.ext.Provider` import — these are different classes

---

## Pattern 12: SPM vs Suite Boundary

**Bugs**: DIFF-SCAN-16

**Root cause pattern**: `MetaDataManager.createMetaData()` serves both SPM and Suite code paths. Suite-specific fields (`typeProperties`, `typeModifiers`) leak into SPM-created attributes.

**Prevention checklist**:
- [ ] When adding Suite-specific fields to metadata, guard with `suiteAttributes` flag check
- [ ] Test creation from both SPM UI and Suite API
