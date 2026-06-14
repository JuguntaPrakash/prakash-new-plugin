# Metadata Package Guide

Custom attribute definitions (e.g., "Risk Level", "Process Owner") that tenants define and attach to BPMN diagrams and glossary items via bindings.

Bug database: see `metadata-bugs/` directory (58 bugs with diffs, 10 regression chains, 63 confirmed shadow bugs).
Git history database: see `metadata-bugs/git-knowledge-db.json` (25 reverts, 183 detailed commits, 117 PR reviews, 175 blame hotlines, 352 coordinated changes, 5 feature flags).

## 4 Key Classes

| Class | Role | Size |
|-------|------|------|
| `MetaDataDefinitionInfo` | Entity. Single-table inheritance, 16 subtypes, composite PK (id + tenantId). | 895 lines |
| `MetaDataManager` | Business logic hub. CRUD, JSON, bindings, config. God class — 86 methods mixing concerns. | 1685 lines |
| `MetaDataService` | Data access. 4 query methods. Stable, simple API. | 106 lines |
| `MetaDataHandler` | REST endpoint at `/meta`. GET/POST/PUT/DELETE. Legacy handler pattern. | 218 lines |
| `MetaDataEntityValue` | Value storage base class. `isValueEqual()` uses HTML unescaping — encoding minefield. | 268 lines |

## Type Hierarchy (16 subtypes, discriminator column "metaType")

Boolean, String, Number, Enum, Date, Url, ModelLink, glossaryLink, iks, MetaDataComplex, risk, DataObject, Decision, user, assetLink, ratings.

Each subtype overrides: `getSingleValue()`, `validateSingleValue()`, `getPropertyType()`.

## Binding System

- `MetaDataStencilSetBinding` (EAGER) — "this attribute appears on Task shapes in BPMN 2.0"
- `MetaDataGlossaryBinding` (EAGER) — "this attribute appears in Glossary Category X"
- Values stored in `MetaDataDiagramValue` (diagrams) and `MetaDataGlossaryItemRevisionValue` (glossary items), both LAZY loaded.

## Attribute Package Dependency

`com.signavio.attribute` wraps this entity — 38 files import MetaDataDefinitionInfo. Changes here affect both systems.

## Hard Rules — Learned from Production Incidents

These rules were each learned the hard way through production outages or multi-week regression chains.
See `metadata-bugs/causal-chains.md` for full incident timelines.

### 1. NEVER change FetchType on MetaDataDefinitionInfo collections
`stencilSetBindings` and `glossaryBindings` MUST stay `FetchType.EAGER`. This has been optimized and reverted **3+ times** (SPM-15744 → SPM-17120 → SIGSPMR-126 → SIGSPM-8634). Every attempt broke AML import, reports, or model saving via `LazyInitializationException`.

### 2. NEVER add/remove date formats without a feature flag
The format list in `MetaDataDate` is global shared state — changes affect ALL tenants and ALL existing data. Regression chain spanning 2+ years: SIGSPM-4300 → SIGSPM-9771 → SIGSPM-12331 → SIGSPM-17308. See class javadoc.

### 3. NEVER add/remove HTML escaping on MetaDataEntityValue
`isValueEqual()` uses `StringEscapeUtils.unescapeHtml3()` for comparison. Encoding in this class has been fixed, broken, and fixed again **6+ times** since 2017 (planio #19511 → #19878 → SIGSPM-7319 → revert → SIGSPM-9589 → SIGSPM-10266). See class javadoc.

### 4. NEVER delete MetaDataDefinitionInfo without bulk pre-deletes
Direct `session.delete()` on a definition with many glossary values triggers cascade loading → OOM. Production outage: SIGINCIDENT-573/585 (SIGSPM-9962). Must delete `GlossaryItemToItemLinkInfo2` and `MetaDataGlossaryValue4` first via SQL/HQL. Controlled by `ENABLE_METADATA_DELETE_OPTIMISATION` feature flag.

## Hot Spots — Known Traps

### MetaDataDate — Format list is global shared state
Adding/removing a date format changes parsing for ALL tenants and ALL existing data. Always wrap behind a feature flag. Always test backward compatibility. Contested PR #6384 (SIGSPM-4300) had **45 review comments** — reviewers flagged backward compatibility concerns that were initially dismissed, leading to 3 follow-up regressions. See class javadoc for incident history.

### MetaDataGlossaryLink — 2 independent value paths + divergent override
`getSingleValue()` and `replaceLinks()` are independent paths. `getListValue()` is overridden to return rawValue unchanged (diverges from base class). Fix one, check the others. `replaceLinks()` has been modified **19 times** across 12+ authors — the most-changed method in the package. See class javadoc.

**GlossaryLink ordering trap (SIGSPM-8448)**: A fix for glossary link ordering was reverted **4 times in a single day** — each attempt broke a different consumer. This pattern is not in the bug database because it was caught before release, but demonstrates the fragility of this class.

### MetaDataEntityValue — Encoding minefield
`isValueEqual()` compares values using HTML unescaping. `cleanupValue()` handles list/single normalization. `isTypeOfStringOrText()` guards prevent unwanted array extraction for String/Text types. 12+ bugs trace to encoding in this class. PR #6453 (SIGSPM-7319, special characters) had **29 review comments** — reviewers debated the scope of the encoding fix, and the narrow scope chosen led to follow-up bugs in other types.

### MetaDataManager — Reflection-based field population
`@MetaDataField` annotation + `updateFields()` method uses reflection to populate entity fields from JSON. This is invisible without knowing the annotation exists. Search for `@MetaDataField` in entity classes to see which fields are auto-populated.

### MetaDataManager.deleteMetaData() — Production OOM risk
Deep FK relationship graph. Correct delete order: (1) `GlossaryItemToItemLinkInfo2` via SQL DELETE with JOIN, (2) `MetaDataGlossaryItemRevisionValue` via HQL bulk DELETE, (3) `session.delete(MetaDataDefinitionInfo)`, (4) `removeBindingsAndValues()`. Never use `collection.clear()` without `session.delete()` on each element first.

### getTypedValue() — Return type polymorphism trap
`MetaDataEntityValue.getTypedValue()` can return `Jackson ArrayNode`, `org.json JSONArray`, or `String` depending on subtype and context. Consumers that assume one type crash with `ClassCastException`. 3 confirmed bugs: SPM-1535, SPM-2116, SPM-1536. Always use `.toString()` and re-parse when crossing type boundaries.

### Validation contract mismatch
Validation strategies inconsistently return `false` vs throw `MetaDataValueException`. Also watch for `javax.inject.Provider` vs `javax.ws.rs.ext.Provider` import confusion — these are different classes. Result: 500 instead of 400 on validation errors.

## VAL Import — Highest Bug Density Area

21 bugs in the VAL import path. Each fix for one metadata type exposes the same gap in others. Key rules:
- Entity matching must check ALL properties (name, type, isList, configuration) — not just name+type
- Single invalid metadata value must never abort entire import — log and continue
- Test ALL metadata types (risk, dictionary link, glossary link) in a single import
- Reference extraction must discover both diagram-level and category-level definitions
- `return` vs `continue` in skip loops has caused bugs (SIGSPM-15916)
- See `metadata-bugs/causal-chains.md` Chains 4, 5, 9

## Shadow Bug Blast Radius

63 confirmed bugs exist OUTSIDE the metadata package but are caused by metadata behavior.
308 additional JIRA tickets describe metadata symptoms without mentioning "metadata."

When modifying metadata, always test across these consumer domains:

| Domain | Co-changed files | Key risk |
|--------|-----------------|----------|
| Dictionary/Glossary | 41 (`GlossaryItemRevision`) | equals() by reference not ID; NPE on null categoryId |
| Model/Editor | 29 (`DiagramUtil`) | Dictionary link visualization; diagram reuse |
| Reports/Excel | 26 (`AttributesExcelReportGenerator`) | Dictionary items shown as NULL; only first item returned |
| SGX Import/Export | 20 (`SgxExporter`) | Binding loading reverts; encoding; session clear timing |
| VAL Integration | 4+ (`ValMetaDataDefinitionHandler`) | 21 bugs — see section above |

## Coupling — Git Coordinated Change Evidence

352 commits touch both metadata AND outside packages. This reveals required co-modifications that an agent can't discover from reading code alone.

### Highest-coupling packages (from git history mining)

| Outside package | Co-changes | Bugfix co-changes | Bugfix % | Key signal |
|----------------|------------|-------------------|----------|------------|
| `glossary.business` | 131 | 31 | 24% | **#1 coupling partner.** Equals by reference, NPE on null categoryId, merge link rewriting. |
| `attribute.adapter.output.mysql.data` | 47 | 21 | 45% | Nearly half of all co-changes are bugfixes — high-risk coupling. |
| `functiontoggle.launchdarkly` | 24 | 17 | **71%** | **Highest bugfix rate.** Almost every co-change is adding or modifying a feature flag after an incident. |
| `glossary.data` | 38 | 14 | 37% | Entity layer coupling — changes to MetaDataDefinitionInfo often require GlossaryItemRevision changes. |
| `attribute.adapter.input.rest` | 29 | 8 | 28% | REST adapter layer — API contract changes propagate here. |
| `platform.handler` | 22 | 6 | 27% | Legacy handler changes when metadata REST contract changes. |

**Rule of thumb**: If your metadata change is a bugfix and touches `glossary.business` or `attribute.adapter`, verify you've addressed both sides. The 71% bugfix rate on `functiontoggle.launchdarkly` co-changes means nearly every metadata feature flag was added *after* a production incident — these flags are safety nets, not rollout toggles.

## Blame Hotlines — Most-Changed Methods

Methods changed 10+ times across multiple authors. Each is a code path where fixes keep accumulating because the original abstraction is leaky or the requirements are inherently complex. Approach these methods with extra caution.

| Method | File | Changes | Authors | Signal |
|--------|------|---------|---------|--------|
| `replaceLinks()` | MetaDataGlossaryLink | 19 | 12+ | Independent path from getSingleValue — fixes to one don't fix the other |
| `createMetaData()` | MetaDataManager | 19 | 10+ | SPM/Suite dual path + reflection field population |
| `deleteMetaData()` | MetaDataManager | 17 | 8+ | Production OOM — bulk pre-delete ordering is critical |
| `validateSingleValue()` | MetaDataDate | 15 | 7+ | Format list backward compatibility — each format change triggers fixes |
| `getReadableValue()` | MetaDataGlossaryLink | 13 | 6+ | Encoding and link resolution — 2 independent value paths |
| `getReadableValue()` | MetaDataEntityValue | 12 | 5+ | Base class readable value — overridden inconsistently by subtypes |
| `getTypedValue()` | MetaDataDefinitionInfo | 11 | 5+ | Polymorphic return type — ArrayNode/JSONArray/String |
| `cleanupValue()` | MetaDataEntityValue | 9 | 4+ | List/single normalization — `[]` treated as empty |
| `setValue()` | MetaDataEntityValue | 9 | 4+ | isValueEqual + HTML unescaping + lazy array migration |

## Feature Flags — All Incident-Driven, All Still Active

All 5 feature flags in metadata code were introduced after production incidents. None are rollout toggles — they are safety nets. All remain in the codebase.

| Flag | Introduced | Incident | Guards |
|------|-----------|----------|--------|
| `ENABLE_METADATA_DELETE_OPTIMISATION` | SIGSPM-9962 | SIGINCIDENT-573/585 (OOM) | Bulk pre-delete before session.delete() |
| `isAllowAdditionalFormatsEnabled` | SIGSPM-4300 | Date format regression chain | Additional date format parsing (has 1 revert in its history) |
| `metadataValueExternalLink` | SIGSPM-16059 | Silent data loss on link type | External link resolution path |
| `enableMetadataDefinitionCreationLimit` | SIGSPM-? | Unbounded definition creation | Max 2048 definitions per tenant |
| `suiteAttributes` | SIGSPM-? | SPM/Suite field leakage | Guards Suite-specific fields in createMetaData() |

**Do not remove these flags** without understanding the incident that caused them. See `metadata-bugs/git-knowledge-db.json` → `feature_flags` section for full lifecycle history.

## Debugging Quick Reference

| Symptom | Check First |
|---------|-------------|
| Attribute not showing on diagram | DB table `MetaDataStencilSetBinding3` (class: `MetaDataStencilSetBinding`) — binding exists? + `MetaDataByStencilSetRequestCache` |
| Attribute not showing in glossary | `MetaDataGlossaryBinding` table — binding exists for category? |
| Wrong value displayed | Subtype's `getSingleValue()` + check `isList` flag |
| Validation error on save | Subtype's `validateSingleValue()` |
| 500 instead of 400 on validation | Validation strategy contract — boolean vs exception? `Provider` import correct? |
| Create fails | Max 2048 limit? ID collision in `MetaDataManager.createMetaData()`? |
| Delete OOM / High GC | `ENABLE_METADATA_DELETE_OPTIMISATION` flag. Bulk pre-delete before `session.delete()`. |
| JSON shape wrong | `MetaDataManager.getMetaDataJson()` → `MetaDataDefinitionInfo.getBasicJsonRepresentation()` |
| SGX import/export broken | `MetaDataDefinitionExchange` field names |
| Translations missing | `MDDTranslationInfo` locale + `retrieveTranslationWithFallbackToEnglish()` fallback chain |
| ClassCastException on value | `getTypedValue()` returns ArrayNode/JSONArray/String — use `.toString()` and re-parse |
| VAL import fails for one type | Test ALL metadata types — fix for one exposes gap in others |
| LazyInitializationException | Session cleared while cached MetaDataDefinitionInfo still referenced? FetchType must be EAGER. |
| Value unchanged after save | `isValueEqual()` HTML unescaping — stored `&amp;` matches input `&` |

## Common Bug Clusters (12 patterns)

1. **Encoding/escaping** (12 bugs) — HTML unescaping in `isValueEqual()`, double-encoding through browser→REST→JSON→DB→render chain. Test with: `& " < > \ [ ] &nbsp;` newlines.
2. **Date format** (12 bugs) — Global shared state, 4-bug regression chain. Always feature-flag format changes.
3. **VAL import/export** (21 bugs) — Entity mapping, locale handling, reference extraction. Test all types in one import.
4. **List vs single value** (6 bugs) — `cleanupValue()` treats `[]` as empty. `getSingleValue()` / `getListValue()` / `replaceLinks()` diverge.
5. **Multilanguage/i18n** (13 bugs, 24% of open) — Translation fallback chain, default value per locale, reports show only main language.
6. **Permission/security** (6 bugs) — Use `findPublicUserDto()` not `findUserDto()`. XSS in custom table attributes.
7. **Performance/memory** (6 bugs) — Delete OOM, `/p/meta` slowness, N+1 on bindings. Check `MetaDataByStencilSetRequestCache`.
8. **Dictionary link consistency** (8 bugs) — Merged/deleted items become stale references. `replaceLinks()` must update during merge.
9. **Hibernate lazy loading** (5 bugs) — FetchType change → revert cycle. Initialize collections before session clear.
10. **Delete ordering** (3 bugs) — FK constraint graph. Bulk pre-delete required. Production OOM.
11. **Validation contract** (2 bugs) — Boolean vs exception mismatch. `Provider` import confusion.
12. **SPM vs Suite boundary** (1 bug) — `createMetaData()` serves both; guard Suite fields with `suiteAttributes` flag.

## Implicit Contracts — Rules Not Enforced by Code

These rules exist only in the codebase's collective memory. No interface, annotation, or compiler check enforces them.
Violating any of them compiles fine but causes bugs. Each rule below was learned from at least one production bug.

### 1. getTypedValue() return type is polymorphic
`MetaDataEntityValue.getTypedValue()` returns `Object`, but the actual runtime type varies: Jackson `ArrayNode`, org.json `JSONArray`, or plain `String` depending on the subtype and whether the value is a list. Consumers must never cast directly — use `.toString()` and re-parse.
**Bugs**: SPM-1535 (ClassCastException), SPM-2116 (ArrayNode instead of JSONArray), SPM-1536 (empty list `"[]"` not treated as empty).

### 2. validateSingleValue() must throw, never return false
Some validation strategies return `false` on invalid input, others throw `MetaDataValueException`. The ExceptionMapper only catches the exception — returning `false` causes a 500 response instead of 400.
Also: `javax.inject.Provider` and `javax.ws.rs.ext.Provider` are different classes with the same simple name. Wrong import = ExceptionMapper not Guice-bound = 500.
**Bugs**: DIFF-SCAN-14 (SIGSPM-19479/19535).

### 3. Date validation must accept ALL known formats AND empty strings
`MetaDataDate.validateSingleValue()` must try all format variants (`d/m/y`, `d.m.y`, `d-m-y`, ISO) and must treat empty string as valid "no value." Rejecting any previously-accepted format breaks existing data globally.
**Bugs**: SIGSPM-4300 → 9771 → 12331 → 17308 (regression chain spanning 2+ years).

### 4. replaceLinks() must handle the same value formats as getSingleValue()
`MetaDataGlossaryLink` has independent code paths for reading values (`getSingleValue`) and rewriting them during merge/copy (`replaceLinks`). Both must handle the same JSON formats — single href strings and JSON arrays of hrefs.
**Bugs**: GlossaryLink merge defect requiring 3 separate PRs because each path was fixed independently.

### 5. Entity matching in imports must check ALL properties
VAL import matches metadata definitions by name+type, but `isList`, `configuration`, and subtype-specific properties (like glossary category ID) are also part of the identity. Name+type match with different `isList` silently overwrites the wrong definition.
**Bugs**: SIGSPM-19543 (type changed between imports), SIGSPM-16595 (isList mismatch).

### 6. Import errors must never abort the entire operation
A single invalid metadata value (wrong date format, missing enum item, changed definition type) must be logged and skipped — never thrown as an exception that aborts the whole import.
**Bugs**: SIGSPM-15976 (unknown dropdown items abort import), SIGSPM-12338 (invalid attribute values abort import), SIGSPM-9771 (invalid date aborts Excel import).

### 7. Link resolution must degrade gracefully
When resolving entity links (glossary items, files, models), inaccessible or deleted entities must be skipped — not thrown as exceptions and not silently classified as external URLs.
**Bugs**: SIGSPM-16059 (wrong link type → silent data loss), SIGSPM-13909 (inaccessible file → crash), DIFF-SCAN-11 (permission check → crash on whole list).

### 8. Locale must never be the literal string "undefined"
`getDefaultOrUndefinedLocale()` can return the string `"undefined"` — this is not a valid locale code. Using it as a map key causes metadata values to be keyed under a non-existent locale, making them invisible.
**Bugs**: SIGSPM-16957 (VAL transfer ignores dictionary relations).

### 9. SRI construction must use source tenant ID, not target
During VAL import, Source Resource Identifiers (SRIs) must be constructed with the source workspace's tenant ID. Using the target tenant ID creates invalid references that can't be resolved.
**Bugs**: SIGSPM-11750.

### 10. User DTO lookups must use findPublicUserDto()
In data accessors that resolve user-type metadata values, always use `findPublicUserDto()` instead of `findUserDto()`. The latter throws `InsufficientPrivilegesException` when a non-admin user looks up another user's info.
**Bugs**: SIGSPM-19677 (attributes don't load when toggle is on).

## REST API Test Coverage — Known Gaps

There are **no contract/API tests** (`backend-api-test`) for the core metadata CRUD endpoints. The SAML metadata API tests (`SamlMetadataEndpointApiTest`) test SAML XML metadata, not custom attributes — naming collision.

| Endpoint | Contract test | Handler test | Bugs found without contract tests |
|---|---|---|---|
| `/p/meta` (CRUD) | **None** | MetaDataHandlerRestTest | SIGSPM-10215 (privilege bypass — Bug Bounty), SIGSPM-6623 (invisible definitions) |
| `/p/metaconfig` | **None** | MetaDataConfigurationHandlerRestTest | — |
| `/p/metase` | **None** | MetaDataStencilSetExtensionHandlerRestTest | — |
| `/p/metaexternal` | **None** | MetaDataExternalHandlerTest | — |
| `/attribute-service/.../attribute-definitions` | **None** | **None** | — |
| `/attribute-service/.../attribute-values` | PUT only | **None** | — |

Handler tests (`handlerTest/`) test within the JVM — they verify business logic but not the actual HTTP contract (status codes, response body shapes, error formats, content types). The 500-instead-of-400 bug (DIFF-SCAN-14) is the type of issue that only a contract test would catch, because the handler test calls Java methods directly and never sees the HTTP status code mapping.
