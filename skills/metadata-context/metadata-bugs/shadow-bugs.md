# Shadow Metadata Bugs

Bugs related to metadata behavior but fixed OUTSIDE the metadata package.
Found through: JIRA link graph, import/caller analysis, file co-change analysis, symptom keyword search.

## Discovery Methods

| Method | Count | Confidence | What it finds |
|--------|-------|------------|---------------|
| JIRA link graph | 41 confirmed | **High** | Follow Cause/Duplicate/Relation links outward from 290 known metadata bugs |
| Import graph | 22 confirmed | **High** | Git-verified bugs in files importing from `com.signavio.metadata.*` |
| Co-change analysis | 20 files profiled | **High** | Files that statistically change alongside metadata files, with verified bug commits |
| Symptom search | 459 candidates | **Low-Medium** | JIRA keyword matches for metadata symptoms — not individually verified |

**Confirmed shadow bugs: ~63.** The 459 from symptom search are unverified candidates — they show the blast radius but include false positives (generic UI bugs mentioning "attribute" or "dictionary" without a metadata root cause).

---

## 1. Shadow Bugs from JIRA Link Graph (41 bugs)

### Still Unresolved (5 — most actionable)

| Key | Priority | Summary | Linked From | Link Type |
|-----|----------|---------|-------------|-----------|
| SIGSPM-6959 | High | Preserving 'extensionElements' in BPMN-XML files does not work for boundary events | SIGSPM-7205, SIGSPM-7711 | relates to |
| SIGSPM-10512 | Medium | Explorer doesn't show data field with default date format | SIGSPM-17308 | caused by |
| SIGSPM-12263 | Medium | Special character display incorrect in SPM Editor | SIGSPM-19324 | relates to |
| SIGSPM-8628 | Medium | After merging of dictionary item, old one is shown in Hub | SIGSPM-4484 | relates to |
| SIGSPM-3166 | High | 'Advanced Search' button out of order | SIGSPM-2954 | relates to |

### Resolved Shadow Bugs by Theme

#### Dictionary / Glossary Merge (7 bugs)
- **SIGSPM-9642** Fixed — Dictionary name in diagram not updated after merge (→ SIGSPM-4484)
- **SIGSPM-18863** Fixed — Merged dictionary entry removed in Editor after saving model (→ SIGSPM-16377)
- **SIGSPM-11667** Done — Part of merging fails in dictionary (→ SIGSPM-15999)
- **SIGSPM-42** Done — Dictionary link lists cannot be overridden in Editor (→ SIGSPM-4063)
- **SIGSPM-7805** Won't Fix — Glossary: dict item doesn't open (→ SIGSPM-8069)
- **SIGSPM-8077** Done — Content of dictionary item different from preview (→ SIGSPM-8069)
- **SIGSPM-200** Won't Do — Overriding dictionary names in diagrams is broken (→ SIGSPM-4143)

#### Date Format / Timezone (6 bugs)
- **SIGSPM-198** Done — The Zulu time issue (→ SIGSPM-262)
- **SIGSPM-11653** Fixed — Wrong date in the feed/Explorer (→ SIGSPM-11255)
- **SIGSPM-497** Duplicate — Date format changes when clicking in Editor (→ SIGSPM-633)
- **SIGSPM-7793** Cannot Reproduce — Date format changed by system for dictionary entries (→ SIGSPM-7830)
- **SIGSPM-5444** Cannot Reproduce — Chrome displayed date format different than defined (→ SIGSPM-11255)
- **SIGSPM-7814** Won't Fix — No error indicator in Firefox (→ SIGSPM-7755)

#### SGX Import/Export (5 bugs)
- **SIGSPM-14587** Fixed — SGX export loads most dictionary items repeatedly (→ SIGSPM-15771)
- **SIGSPM-15913** Done — Glossary Excel Import takes extremely long (→ SIGSPM-15771)
- **SIGSPM-4329** Won't Do — Suboptimal error message when SGX import fails (→ SIGSPM-5255)
- **SIGSPM-7260** Duplicate — SGX import fails because of dictionary items (→ SIGSPM-4294)
- **SIGSPM-7156** Duplicate — SGX import doesn't check if language already exists (→ SIGSPM-4225)

#### VAL Integration (4 bugs)
- **SIGSPM-14069** Fixed — VAL import fails if entity names near length limit (→ SIGSPM-13778)
- **SIGSPM-14323** Fixed — All VAL imports retried even if not recoverable (→ SIGSPM-15746)
- **SIGSPM-12443** Fixed — VAL export of unofficial diagram types fails (→ SIGSPM-12318)
- **SIGSPM-16053** Fixed — VAL dictionary item export does not use undefined locale (→ SIGSPM-16957)

#### Encoding / XSS (4 bugs)
- **SIGSPM-32** Done — Persistent XSS in Dictionary Upload Part I: Title + Description (→ SIGSPM-35)
- **SIGSPM-8650** Fixed — XSS injection into commenting view (→ SIGSPM-8434)
- **SIGSPM-6644** Won't Fix — & in Tenant name is html encoded (→ SIGSPM-19324)
- **SIGSPM-13838** Fixed — PMDs special characters issue (→ SIGSPM-12654)

#### Visualization / Overlays (2 bugs)
- **SIGSPM-16695** Done — Overlay: Translations not applied in visualization (→ SIGSPM-16267)
- **SIGSPM-15170** Done — Investigate: Notationscheck error (→ SIGSPM-15066)

#### Other Shadow Bugs
- **SIGSPM-18021** Done — Login to SPG via SPM gets 500 error (→ SIGSPM-17972, encoding in redirect)
- **SIGSPM-11910** Fixed — Item translations missing after primary language switch (→ SIGSPM-12258)
- **SIGSPM-17849** Done — Refresh button not appearing in toolbar (→ SIGSPM-15542)
- **SIGSPM-15376** Duplicate — Sub-Categories not visible in explorer (→ SIGSPM-15542)
- **SIGSPM-3149** Done — Process Doc: NullPointer loading linked model (→ SIGSPM-3876)
- **SIGSPM-4187** Done — Irrelevant subsets activated in workspace (→ SIGSPM-1403)
- **SIGSPM-8171** Duplicate — Risk management not selectable as data type (→ SIGSPM-1407)
- **SIGSPM-14384** Fixed Elsewhere — Interface Signavio LeanIX doesn't work (→ SIGSPM-16059)

---

## 2. Shadow Bugs from Import Graph Analysis (22 bugs)

200+ files outside `com.signavio.metadata` import from it. The following 22 confirmed bugs were fixed in those consumer packages.

### Type-Mismatch / Casting (3 bugs)

`MetaDataDiagramValue.getTypedValue()` can return different JSON container types (Jackson `ArrayNode` vs org.json `JSONArray`) depending on context. Consumers that assume one type crash.

| Commit | Bug | Summary | Fix Location |
|--------|-----|---------|--------------|
| `866763b3bbf` | SPM-1535 | Dictionary link lists: `ArrayNode` vs `JSONArray` ClassCastException | Rule.java, AbstractGlossaryRelation.java, MetaDataEntityValueDiagramApplier.java |
| `7898879837e` | SPM-2116 | URL list metadata: `ArrayNode` instead of `JSONArray` | AbstractUrlRelation.java — parse via `new JSONArray(value.toString())` |
| `55b06bc5355` | SPM-1536 | Empty list `"[]"` string not treated as empty | GlossaryIsEmptyRelation.java — added `"[]".equals()` check |

### Glossary Link Resolution (2 bugs)

| Commit | Bug | Summary | Fix Location |
|--------|-----|---------|--------------|
| `6162f53255e` | SIGSPM-16240 | Dictionary links not visually resolved in model explorer | DiagramUtil.java — added `resolveGlossaryLinks()` method |
| `f7cb50f459d` | SIGSPM-14953 | GlossaryItemsInModelLoader optimization broke loading chain | Reverted |

### Locale / Language Handling (3 bugs)

| Commit | Bug | Summary | Fix Location |
|--------|-----|---------|--------------|
| `ffd7883583f` | SIGSPM-16957 | VAL dictionary item handler used wrong locale for metadata title | ValDictionaryItemHandler.java |
| `3324929b5fd` | SPM-12265 | SGX import lost translated names on locale change | SgxImporter.java — use `removeLocalizedValue()` |
| `a917af4cc8d` | SIGSPM-17864 | Language code fix in revision/stencilset | Revision area |

### Null Pointer Exceptions (3 bugs)

| Commit | Bug | Summary | Fix Location |
|--------|-----|---------|--------------|
| `3200448db08` | RAT-271 | NPE in PDF print for risk management relations | IksUncontrolledRisksRelation.java — null guards on MetaDataDefinitionInfo |
| `b72ae033d97` | SIGSPM-13473 | NPE in ChangePropagationService: groups map null when flag disabled | ChangePropagationService.java — init as `new HashMap<>()` |
| `72a7e4a8a61` | SIGSPM-13119 | NPE in WireDiagram: `findMatchingShapeOnDiagram()` returned null | WireDiagram.java — added `.filter(Objects::nonNull)` |

### SQL / Persistence Layer (3 bugs)

| Commit | Bug | Summary | Fix Location |
|--------|-----|---------|--------------|
| `98a7c4da15b` | SIGSPM-19280 | SQL alias missing `as tenantId` in MetaDataAssetValueRepository | MetaDataAssetValueRepository.java — one-char fix |
| `fc6e15b5087` | SIGSPMR-126 | MetaDataDefinition loading optimization broke GlossaryCategory and ExcelImport | Reverted twice |
| `e70981f47d8` | SIGSPM-10921 | Session clear during model revision caused LazyInitializationException | SgxImporter.java — moved session clear |

### Import/Export Value Handling (4 bugs)

| Commit | Bug | Summary | Fix Location |
|--------|-----|---------|--------------|
| `16ef7a70221` | SIGSPM-9819 | SGX import duplicated custom SVGs on re-import | Check existing SVG IDs before creating |
| `8c6f1e34236` | SIGSPM-11963 | VAL import: duplicate attribute names silently overwritten | CustomAttributeCreator.java — `addNumberSuffix()` |
| `59ce3fd365e` | SIGSPM-15916 | VAL: `return` instead of `continue` in skip dictionary loop | Changed `return` → `continue` |
| `336b4918cc9` | SIGSPM-17524 | VAL skip dictionary flag: empty ID mapping not handled | Skip logic restructured |

### Security / XSS (2 bugs)

| Commit | Bug | Summary | Fix Location |
|--------|-----|---------|--------------|
| `2d5d5087e33` | SIGDICTIONARY-965 | Stored XSS via `javascript:`, `data:` URL protocols in glossary | GlossaryItemJsonParser.java, MetaDataUrl.java, new XssProtectionUtil |
| `e4cbddefd2c` | SIGSPM-19623 | Null PersistedAttributeValue in attribute service pipeline | Introduced `PersistedAttributeValue.empty()` sentinel |

### Recurring Patterns in Consumer Packages

| Pattern | Count | Affected Packages |
|---------|-------|-------------------|
| Type polymorphism in `getTypedValue()` (JSONArray vs ArrayNode vs String) | 3 | customlayers, model.util |
| Locale/language mismatch in metadata values | 3 | integration.val, platform.util.sgx |
| Missing null guards on MetaDataDefinitionInfo or typed values | 4 | customlayers, comparator, attribute |
| SQL alias / Hibernate session mismanagement with metadata entities | 3 | attribute, platform.util.sgx, glossary |
| Import/export name collisions and skipping logic | 3 | integration.val, platform.util.sgx |
| Glossary link resolution not applied in all code paths | 2 | warehouse.model.util, glossary.business |
| Security: unvalidated metadata URL values | 2 | glossary.handler, metadata.data |

---

## 3. File Co-Change Analysis (Top 20 Files)

873 commits touched the metadata package. The following files change alongside metadata most frequently and have their own metadata-related bugs.

### Top 20 Co-Changed Files

| Rank | Co-Changes | File | Key Bugs Found |
|------|-----------|------|----------------|
| 1 | **41** | `GlossaryItemRevision.java` | NPE when `info.getItemId()` null (83896f1); **equals() comparing MetaDataDefinitionInfo by reference not ID** (fffde43) |
| 2 | **40** | `GlossaryItem.java` | NPE when `categoryId` null/garbage — fallback to BusinessHierarchy (997dba4, SIGSPM-7517); StackOverflow (82f83fa) |
| 3 | **37** | `GlossaryItemDataAccessor.java` | Merge defect for list dict link attribute (75871d1); broken process level pyramid (fcf488f) |
| 4 | **36** | `GlossaryCategory.java` | Wrong MetaDataService accessor (93728f1, SIGSPMR-32); **reverted MetaDataDefinition loading optimization** (fc6e15b, SIGSPMR-126); NPE on `getOldCategory()` (ee4bb36); deadlock in deletion (7bb2319) |
| 5 | **35** | `GlossaryService.java` | Removes non-existing metadata (0e24b5c, MSR-801); XSS URL validation (44327a9); duplicate check on publish (350afcd) |
| 6 | **33** | `PlatformModule.java` | Guice injection fixes for metadata services (0bacd90, 1bbe795, d52070b) |
| 7 | **29** | `DiagramUtil.java` | Dictionary link visualization fix (6162f53); diagram reuse fix (c11d27a) |
| 8 | **26** | `AttributesExcelReportGenerator.java` | **Dictionary items shown as NULL** — `getLinkedGlossaryItems()` returned only first item (8e4929a) |
| 9 | **25** | `GlossaryItemJsonParser.java` | XSS URL protocol validation |
| 10 | **25** | `GlossaryHandler.java` | Category deletion hotfix (95f1259); pagination >20 items (c0ce03b); RevModeException (023a993) |
| 11 | **24** | `GlossaryUtil.java` | Caching fix (2c79724) |
| 12 | **24** | `GlossaryItemJsonBuilder.java` | **NPE on oldCategory** — `getOldCategory()` returns null (d4c5928, SPM-19271) |
| 13 | **21** | `LinkHandler.java` | **Reverted MetaDataDefinition loading optimization** — wrong accessor `SignavioInjector.get()` vs `SecurityManager.loadTenantSingletonObject()` (fc6e15b, SIGSPMR-126) |
| 14 | **21** | `ModelRevisionInfo.java` | Entity `equals()`/`hashCode()` fix (08fde45) |
| 15 | **21** | `GlossaryExcelImporter.java` | String attribute escaping (96bc284); throttle fix (c3ea368); ID column retrieval (a3b7feb) |
| 16 | **21** | `GlossaryInfoHandler.java` | Prevent null exception logging (c27005965) |
| 17 | **20** | `SgxExporter.java` | **Reverted MetaDataDefinition binding loading** — `getAllMetaDataDefinitionsBindingsInitialised()` back to `getAllMetaDataDefinitions()` (aa194028); encoding fix (5c81c66) |
| 18 | **20** | `GlossaryItemInfo.java` | `@Nullable` on `getCategoryId()` (997dba40, SIGSPM-7517) |
| 19 | **19** | `SgxImporter.java` | Detached entity on template import (7ed6166); multi-language name fix (3324929) |
| 20 | **19** | `Glossary.java` | Title translation fixes (8ae99e1, 19efdb0) |

### Notable Files Just Outside Top 20

| Co-Changes | File | Bug |
|-----------|------|-----|
| 18 | `IndexUpdateCallback.java` | Session opened but only closed in error path, not success path (7b5448a, SPM-10650) |
| 18 | `DiagramAttributeTransformer.java` | Regex `"\\n"` (literal) instead of `"\n"` (newline) in dictionary-linked shape handling (a3643586, SIGSPM-17013) |
| 17 | `MetaDataExtension.java` | Category ID overwritten — `mappedCategoryId` set then immediately nulled on next line (c54811e, RAT-906) |

### Key Co-Change Patterns

1. **MetaDataDefinition loading optimizations reverted 3+ times** (SIGSPMR-126): Optimized `JOIN FETCH` queries in MetaDataService broke GlossaryCategory, LinkHandler, MetaDataHandler, and SgxExporter
2. **MetaDataGlossaryLink equals() by reference not ID** (fffde43): Custom attribute values not found in GlossaryItemRevision
3. **MetaDataGlossaryBinding category mapping**: SGX import `MetaDataExtension` overwrote mapped category ID (c54811e)
4. **Cascading null safety**: Multiple NPEs across glossary classes for nullable metadata fields (`categoryId`, `info`, `oldCategory`, `itemId`)

---

## 4. JIRA Symptom Keyword Search (459 candidates — LOW-MEDIUM confidence)

Users almost never say "metadata" — they describe symptoms using "custom attribute", "dictionary", "date format", etc. This search casts a wide net for bugs that **may** be caused by metadata internals but are described purely as consumer-side symptoms.

**Caveat**: These are keyword matches, not individually verified. Many will be genuine metadata-caused bugs, but some are generic UI/import bugs that happen to mention "attribute" or "dictionary". Treat as a candidate list for further triage, not as confirmed shadow bugs.

### Summary by Category

| Category | Total Found | Status Breakdown |
|----------|-------------|-----------------|
| Custom attribute symptoms | **308** | Mix of Done, Backlog, In Progress |
| Dictionary/glossary link issues | **58** | Mostly Done |
| Import/export issues | **60** | Mostly Done |
| Date format in non-metadata context | **16** | ~50% Backlog |
| Report issues with custom columns | **16** | Mostly Done |
| Editor/Explorer mentioning "MetaData" | **1** | Done |

### Notable Currently-Open/Backlog Bugs (from symptom search)

| Key | Priority | Summary | Category |
|-----|----------|---------|----------|
| SIGSPM-19477 | Medium | Overlays for new created attributes show no values | Custom attributes (In Progress) |
| SIGSPM-10512 | Medium | Explorer doesn't show data field with default date format | Date format (Backlog) |
| SIGSPM-4790 | High | Not all table columns are shown in the explorer | Custom attributes (Backlog) |
| SIGSPM-12958 | Medium | Modeling conventions report wrong "Consistency" errors when >50 models | Reports (Backlog) |
| SIGSPM-13967 | Medium | Inability to set glossary link (list) to empty list | Dictionary link (Backlog) |
| SIGSPM-4571 | Medium | Process has 2 links on same Dictionary Item after merging 2 items | Dictionary merge (Backlog) |
| SIGSPM-4014 | Medium | Date has different view in Editor for dictionary for different countries | Date format (Backlog) |
| SIGSPM-10129 | Medium | Faulty date picker and annotation in Editor | Date format (Backlog) |
| SIGSPM-7069 | High | Dictionary import creates empty rows if language not declared | Import (Backlog) |
| SIGSPM-18365 | Medium | Process Governance: Date format and language not displayed in German | Date/i18n (Backlog) |

### Top 50 Custom Attribute Symptom Bugs (of 308)

| Key | Summary | Priority | Status |
|-----|---------|----------|--------|
| SIGSPM-10748 | Explorer shows [object Object] in column instead of dictionary entry | Medium | Done |
| SIGSPM-16977 | Link overflows column space in Process Documentation PDF | High | Deployed |
| SIGSPM-12958 | Modeling conventions report wrong "Consistency" errors >50 models | Medium | Backlog |
| SIGSPM-7682 | Fix display and limitation for overlay visualization | High | Done |
| SIGSPM-7079 | Risk & Control names missing in Hub with multi-language | High | Done |
| SIGSPM-4790 | Not all table columns shown in explorer | High | Backlog |
| SIGSPM-1401 | Language mix for risk management in Editor and Reports | High | Done |
| SIGSPM-5797 | BPMN XML import fails on missing data object references | Medium | Done |
| SIGSPM-4432 | Process documentation: attached documents from Dictionary missing | Medium | Done |
| SIGSPM-16670 | Report for risk/controls shows links instead of dictionary entries | Medium | Done |
| SIGSPM-15375 | Modeling Convention report filters require value for "is empty" | Low | Done |
| SIGSPM-19477 | Overlays for new created attributes show no values | Medium | In Progress |
| SIGSPM-18863 | Merged dictionary entry removed in Editor after saving model | Medium | Done |
| SIGSPM-15976 | VAL Imports fail for unknown dropdown items | Medium | Merged |
| SIGSPM-7258 | Comparator view not displaying changes correctly | Medium | Done |
| SIGSPM-7380 | Using dictionary item, "revert" does not revert value in Editor | High | Done |
| SIGSPM-16501 | Explorer is not showing column information | Very High | Done |
| SIGSPM-543 | Dictionary import: if "Relevant documents" not mapped they are deleted | Very High | Done |
| SIGSPM-7022 | Merging dictionary items fails in special case | Very High | Done |
| SIGSPM-6827 | SGX Import: Distinguish between single value and list for dictionary links | Medium | Done |

### Key Insight

**308 candidate bugs describe custom attribute symptoms without ever using the word "metadata."** While not all are confirmed metadata-caused, the pattern strongly suggests the metadata package's surface area is far larger than its JIRA ticket count suggests. Any search for "metadata bugs" that only looks at tickets mentioning "metadata" will miss a significant portion of actual impact.

---

## Blast Radius Diagram

```
                        ┌─────────────┐
                        │  MetaData   │
                        │   Package   │
                        │  (290 JIRA) │
                        └──────┬──────┘
           ┌───────────────┬───┴───┬───────────────┐
           │               │       │               │
    ┌──────▼──────┐ ┌──────▼──────┐│ ┌──────▼──────┐
    │  Dictionary │ │    Model    ││ │   Reports   │
    │  /Glossary  │ │  /Editor    ││ │  /Excel     │
    │  (58 JIRA)  │ │  (308 JIRA) ││ │  (16 JIRA)  │
    │  (41 co-chg)│ │  (29 co-chg)││ │  (26 co-chg)│
    └──────┬──────┘ └─────────────┘│ └─────────────┘
           │               ┌───────┘
    ┌──────▼──────┐ ┌──────▼──────┐
    │ SGX Import  │ │    VAL      │
    │ /Export     │ │ Integration │
    │  (60 JIRA)  │ │  (21 bugs)  │
    │  (20 co-chg)│ │  (4 co-chg) │
    └─────────────┘ └─────────────┘

    co-chg = top co-changed file count in that domain
    JIRA   = symptom keyword search results
```

When modifying metadata, test across ALL these consumer domains.
