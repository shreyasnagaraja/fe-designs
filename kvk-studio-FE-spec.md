# KVK Studio: frontend spec (registry-driven edit, copy and de-identify)

For one GitHub Copilot session in the **frontend repo** (Vue + TypeScript + Pinia + Alova, deployed to an Azure storage account, desktop and intranet only). No backend work is needed here: the upload endpoint and the existing "KvK extract upload" page already exist, and the registry file is produced by a separate backend-repo session. The visual reference is `kvk-editor-mockup.html` (optional; this file is enough to build from). Everything the studio adds goes into **one new folder**, with a very small change to the existing page.

## 0. What this builds

On the existing **KvK extract upload** page (Administration menu), a developer can load a KVK extract XML and then:

1. see every field in the registry, grouped in sections, with all its xpaths in priority order,
2. edit values, fill empty ones, remove nodes, or copy the extract into a new one (new KVK number), to create test data for many scenarios,
3. for extracts that come from production: de-identify personal data first, review what was replaced, and only then edit and upload,
4. upload the final XML through the **existing** upload endpoint.

**The screen and the de-identification are generic.** The frontend contains no field names, xpaths or replacement-file names: they all come from the registry file (schema below) and the replacement files it points to. All XML parsing, editing, validation and de-identification happens in the browser. Production XML is never sent anywhere before it has been de-identified.

Facts about the existing backend (already built; **use it as it is, change nothing**):

- `POST /api/datasets/kvkExtract/{kvkNumber}`, body `multipart/form-data` with one required binary field `file` (the XML). `kvkNumber` is the 8-digit path parameter.
- **An existing extract for the same KVK number is replaced.** The 200 response is `application/json` and contains a string.
- Auth, base URL and the Alova instance are whatever the existing upload page already uses. Import and reuse its upload function; do not write a second one. If it cannot accept a `File` built in code, make the smallest possible change and tell me.

## Registry v2 schema (shared by the backend and frontend specs; both sides must follow it exactly)

One JSON file, `kvk-parameters.json`, drives the whole studio screen and the de-identification. The frontend contains no field names, xpaths or file names of its own: everything comes from this file and the replacement files it points to.

```json
{
  "schemaVersion": 2,
  "namespaces": { "k": "urn:example:kvk" },
  "copyMode": { "kvkNumberKey": "company.kvkNumber", "nameKey": "company.tradeName" },
  "replacementFiles": {
    "addresses":   { "path": "replacements/addresses.json",   "kind": "records", "fields": ["street", "houseNumber", "postcode", "city"] },
    "firstNames":  { "path": "replacements/first-names.json", "kind": "records", "fields": ["name", "gender"] },
    "lastNames":   { "path": "replacements/last-names.json",  "kind": "strings" },
    "companyNames":{ "path": "replacements/company-names.json","kind": "strings" }
  },
  "groups": [
    { "id": "company",           "label": "Company" },
    { "id": "company.address",   "label": "Company · visiting address" },
    { "id": "director1",         "label": "Director 1", "collapsedWhenEmpty": true },
    { "id": "director1.address", "label": "Director 1 · home address", "collapsedWhenEmpty": true }
  ],
  "parameters": [
    {
      "key": "company.address.street",
      "label": "Street",
      "hint": "2 paths",
      "group": "company.address",
      "type": "text",
      "mandatory": true,
      "maxLength": 80,
      "xpaths": ["/k:kvkExtract/k:bezoekadres/k:straatnaam", "/k:kvkExtract/k:postadres/k:straatnaam"],
      "entity": "company.address",
      "deid": { "strategy": "pickFromFile", "file": "addresses", "field": "street", "identity": true }
    }
  ]
}
```

**Top level**

| Field | Meaning |
| --- | --- |
| `schemaVersion` | Always `2`. The frontend refuses any other value with a clear message |
| `namespaces` | Prefix to URI map used by the xpaths; may be empty |
| `copyMode` | The parameter keys that hold the KVK number and the trade name; Copy mode writes the new values there |
| `replacementFiles` | Id to file description. `path` is relative to the registry folder. `kind` is `strings` (a JSON array of strings) or `records` (a JSON array of objects); for `records`, `fields` lists the object properties |
| `groups` | Display sections in screen order. `collapsedWhenEmpty: true` collapses a group whose parameters have no node in the XML, with a button to expand it |

**Parameter**

| Field | Meaning |
| --- | --- |
| `key` | Unique, stable, dot-separated id |
| `label`, `hint` | Text shown on the screen |
| `group` | A group id; the screen shows the parameter under that heading |
| `type` | `text`, `date`, `number`, `enum` or `boolean` |
| `dateFormat` | For `date`: `yyyy-MM-dd` (default) or `yyyyMMdd` |
| `options` | For `enum`: `[{ "value": "...", "label": "..." }]`; for `boolean` the values are `true` and `false` |
| `mandatory` | At least one of its xpaths needs a non-empty value |
| `maxLength`, `pattern` | Optional extra checks; `pattern` is a regular expression the whole value must match |
| `xpaths` | Leaf xpaths in priority order; the first non-empty one is the value MPMA reads. One parameter per XML value element: addresses are split into their own parameters (street, house number, postcode, city, country, ...) wherever the XML splits them |
| `entity` | Optional id of the real-world thing the parameter describes (a person, an address, the company). Parameters with the same `entity` are replaced together |
| `deid` | Optional. Absent means `keep` (never changed). See below |

**`deid`: how a parameter is de-identified**

| Field | Meaning |
| --- | --- |
| `strategy` | One of `keep`, `pickFromFile`, `dateShift`, `bsn`, `patternPreserving`, `compose`, `initials` |
| `file`, `field` | For `pickFromFile`: the replacement file id and, for `records` files, the property to use |
| `filter` | Optional, for `pickFromFile` on a `records` file: `{ "param": "<key of a parameter>", "map": { "M": "male", "V": "female" }, "recordField": "gender" }`. The value of that parameter is mapped and only records whose `recordField` equals the mapped value are used |
| `from`, `join` | For `compose`: keys of other parameters whose replacement values are joined (empty ones skipped) with `join` (default a space). For `initials`: keys whose replacement values give the initials, e.g. `J.` per first letter |
| `identity` | `true` on the parameters that identify the entity (a person's last and first name; an address's postcode and house number). Entities whose identity values are equal get the same replacement, also across different groups. If no parameter of an entity has `identity`, all its non-`keep` parameters count |
| `sweep` | Whether other occurrences of the original value elsewhere in the XML are replaced too (default `true` for text strategies, `false` for `dateShift`) |
| `sweepMinLength` | Shortest original value (after trimming) that may be swept; default 4. Prevents replacing "1" or "NL" everywhere |
| `sweepLastToken` | Also sweep the last whitespace-separated word of the original value (if long enough), mapped to the last word of the replacement; useful for full-name fields |

Strategies:

| Strategy | Replacement |
| --- | --- |
| `keep` | Not changed. Used for KVK number, RSIN, country, legal form and anything not personal |
| `pickFromFile` | A value from the replacement file. The file's record (or string) is chosen once per entity identity, so all parameters of one entity that point to the same file take their fields from the same record (a coherent address) |
| `dateShift` | The date moved by a random offset of 61 to 365 days (never 0), one offset per entity identity; an adult stays at least 18 and the date never lies in the future |
| `bsn` | A random 9-digit number that passes the Dutch 11-test |
| `patternPreserving` | Random characters with the same length and pattern (digits stay digits, letters stay letters, separators stay); for TIN and other identifiers |
| `compose` | Joined from the replacement values of other parameters (for full-name or full-address style elements that exist in the XML besides the split ones) |
| `initials` | Initials built from the replacement values of the parameters in `from` |

**Load-time checks (both the generator's test and the frontend's loader must run them)**

- `schemaVersion` is 2; keys unique; every `group` exists; `copyMode` keys exist.
- Every xpath compiles with the namespace map.
- Every `deid.file` exists in `replacementFiles`; for `records` files, `field` and `filter.recordField` are listed in `fields`; `field` is absent for `strings` files.
- `compose`/`initials` `from` keys and `filter.param` exist; the dependencies contain no cycle.
- `enum` has `options`; `pattern` is a valid regular expression.
- The KVK number and RSIN parameters have no `deid` (or `keep`).

## 1. Ground rules

- Vue 3 + TypeScript + Pinia + Alova, the project's existing structure, router, styling and component conventions. **No new runtime libraries** (Nexus IQ scans dependencies). Use the browser's `DOMParser`, `XMLSerializer`, `document.evaluate` and `crypto.getRandomValues`.
- The studio is shown only when the project's existing string-based feature flag **`kvkExtractStudio`** is on. Non-prod environments only.
- **Never log XML content or extracted values** (no `console.*`, no error text containing values). **Never put the XML in localStorage, sessionStorage, cookies or the URL.**
- Accessible markup: real buttons, labels on inputs, `aria-label` on icon-only buttons, visible focus. Desktop only; no phone layout work.
- UI components stay thin. All XML and de-identification logic lives in plain TypeScript modules with **no Vue, Pinia or Alova imports**, so it can be unit-tested alone.
- I cannot attach a real extract in this chat. Build and test against **synthetic fixtures** (section 11), and the registry file I provide.

## 2. Folder isolation

Everything new goes under one folder (use the project's feature-folder convention; `src/kvk-studio/` is a placeholder):

```
src/kvk-studio/
  index.ts                  // the only file other code imports: KvkStudioPanel and isKvkStudioEnabled()
  types.ts                  // Registry, Parameter, Group, DeidConfig, Edit, ValidationError, DeidReport
  registry/
    kvk-parameters.json     // provided input; do not edit by hand
    replacements/*.json     // provided input; loaded on demand
    loadRegistry.ts         // loads and validates the registry (load-time checks above)
    loadReplacements.ts     // lazy-loads a replacement file by id (dynamic import)
  xml/
    xmlDocument.ts  xpath.ts  applyEdits.ts  validate.ts
  deid/
    deidentify.ts  entities.ts  pick.ts  dates.ts  bsn.ts  pattern.ts  sweep.ts  normalise.ts
  stores/useKvkStudioStore.ts
  components/
    KvkStudioPanel.vue  SourcePanel.vue  DeidReviewPanel.vue  ParameterGroup.vue
    ParameterRow.vue  FieldInput.vue  PathEntry.vue  RawXmlDrawer.vue  SaveBar.vue
  api/uploadExtract.ts      // thin wrapper around the existing upload function
  testing/fixtures.ts       // builds synthetic XML from the registry
  __tests__/
```

Changes outside this folder (list them in your summary):

1. The existing upload page renders `<KvkStudioPanel />` below its current form when `isKvkStudioEnabled()` is true.
2. Nothing else: no global store, router, type or style changes.

## 3. Registry loading

- `loadRegistry.ts` imports `kvk-parameters.json`, runs **all load-time checks** from the schema, and exposes `{ namespaces, copyMode, groups, parameters, replacementFiles }` plus lookups by key and by group. On failure the panel shows one clear error banner (listing the failed checks by parameter key, never values) instead of the screen.
- `loadReplacements.ts` loads a replacement file only when the first de-identification needs it (dynamic import, Vite code splitting), and caches it for the page session. Missing or malformed file: fail the de-identification with a clear error naming the file id.
- If the registry's actual shape differs from the schema above, tell me before building on it.

## 4. XML module

- **`xmlDocument.ts`:** `parse(text)`: reject any input containing `<!DOCTYPE` or `<!ENTITY` before parsing; parse with `DOMParser` as `application/xml`; a `parsererror` result becomes a typed error with no content in the message. `serialise(doc)`: `XMLSerializer`, keeping the XML declaration and encoding of the source (add the declaration back if the serializer drops it).
- **`xpath.ts`:** a namespace resolver built from the registry. `read(doc, xpath)` returns `{exists, value}` (the node's trimmed text, or the attribute value). `set(doc, xpath, value)`: if the node exists, set its text; if not, create the elements along the path and then set the text. Support plain child-element paths, indexes like `bestuurder[2]` (create the missing siblings), attributes (`/@attr`), and predicates of the form `[@attr="value"]` (create the element and attribute). A path too complex to create returns an error naming the parameter key and xpath, never the value. `remove(doc, xpath)` deletes that node only; the parameter's other xpaths stay as they are.
- **`applyEdits.ts`:** `applyEdits(originalXml, mode, newKvkNumber, newName, edits)` parses the original, handles Copy (writes the new KVK number and name to the xpaths of the parameters named in `registry.copyMode`), replays the edits in order, serialises, and returns `{xml, errors}`. An edit is `{key, xpath, action: "SET" | "REMOVE", value?}`. Always start from the original XML; never patch the previous output.

## 5. Validation (`validate.ts`)

Runs on every `applyEdits` result and before upload; collects all errors, not just the first. An error is `{key?, xpath?, message}`.

| Rule | Check |
| --- | --- |
| Well-formed | The XML parses; otherwise one error for the whole file |
| Mandatory | A parameter with `mandatory: true` needs a non-empty value on at least one of its xpaths |
| Type | `date` matches its `dateFormat` and is a real date; `number` is an integer; `enum` is one of `options` (or empty); `boolean` is `true` or `false` |
| Limits | `maxLength` and `pattern` when present |
| Copy mode | New KVK number required, 8 digits, different from the source's KVK number |

## 6. De-identification (`deid/`)

Pure functions. `deidentify(xml, registry, replacements): { xml, report }`, where `report = { counts: Record<string, number> (per strategy), pairs: {label, before, after}[], unlisted: {path}[], warnings: {path?, message}[] }`. The report is shown on screen only; never log or store it. Each run uses fresh randomness (`crypto.getRandomValues`), so a second run gives different results and nothing links two files.

Algorithm, driven only by the registry:

1. **Collect** (`entities.ts`). For every parameter whose `deid.strategy` is not `keep`, read all its xpaths and keep the distinct non-empty original values. Group parameters by `entity`. For each entity occurrence compute its **identity**: the normalised (`normalise.ts`: trim, lowercase, strip accents, collapse whitespace) original values of its parameters flagged `identity: true`, joined; if none are flagged, of all its non-`keep` parameters. An entity with no original values at all is skipped.
2. **Order** the parameters so `compose` and `initials` come after the parameters in their `from`; a cycle is a load error (already checked).
3. **Generate** (`pick.ts`, `dates.ts`, `bsn.ts`, `pattern.ts`):
   - `pickFromFile`: for a `records` file, choose one record per (file id, identity) the first time it is needed, and take `field` from it, so all parameters of one entity that point to the same file get a coherent record (street, house number, postcode and city belong together). For a `strings` file choose one string per (file id, identity). With a `filter`, only records whose `recordField` equals the mapped value of the filter parameter's original value are eligible (no eligible record: any record). Never reuse a record already given to a different identity in this run; if the pool is exhausted, add a warning and allow reuse.
   - `dateShift`: one random offset of 61 to 365 days (never 0, random direction) per entity identity, applied to every date of that entity; support `yyyy-MM-dd` and `yyyyMMdd`. An adult stays at least 18 and no date lies in the future. A value that does not match its `dateFormat` is left alone with a warning.
   - `bsn`: random 9-digit number passing the 11-test; the same original gives the same fake in this run.
   - `patternPreserving`: same length and pattern; the same original gives the same fake in this run.
   - `compose`: join the replacement values of the `from` parameters with `join` (default a space), skipping empty ones. `initials`: first letter of each replacement value in `from`, followed by a full stop, joined without spaces (`J.P.`).
4. **Replace listed xpaths** (`xpath.ts`): in every xpath of the parameter, including lower-priority ones, but only where the node exists or the original value was non-empty; do not create nodes that were not there.
5. **Sweep unlisted places** (`sweep.ts`). Build a table from every original value to its replacement for parameters with `sweep` not false, skipping originals shorter than `sweepMinLength` (default 4); for `sweepLastToken` also add the last whitespace-separated word of the original, mapped to the last word of the replacement. Build **one** combined regular expression from all originals, longest first, case-insensitive, with Unicode-aware boundaries (`(?<![\p{L}\p{N}])` and `(?![\p{L}\p{N}])`, flag `u`). Walk every text node and attribute value once, replace each match with the table's replacement, and record the changed node's path (path only, never the value) in `unlisted`. A node already replaced in step 4 is not counted again.
6. **Final check.** Search the serialised result for every swept original; survivors go into `warnings` (path only). Also list any 9-digit number that passes the 11-test but was not replaced (it may be an RSIN) as a warning; do not replace it.
7. **Report.** Fill `counts`, `pairs` (label from the parameter, before and after, one pair per distinct original), `unlisted`, `warnings`.

Performance: extracts are about 2 MB, rarely more. Run on the main thread (`DOMParser` and `document.evaluate` do not exist in Web Workers, and we add no XML library). Before starting, `await` one animation frame so the "De-identifying..." spinner paints. Build the combined regex once, walk the DOM once, and read each replacement file once. If a 2 MB sample takes more than about 1 second, optimise the sweep first.

## 7. State (Pinia, `useKvkStudioStore`)

```ts
type StudioState = {
  originalXml: string;              // replaced after an accepted de-identification
  workingXml: string;               // latest applyEdits output (or originalXml)
  fileName: string;
  sourceType: "TEST" | "PRODUCTION";
  mode: "EDIT" | "COPY";
  newKvkNumber: string; newName: string;
  rows: Record<string, { xpath: string; exists: boolean; originalValue: string; value: string; removed: boolean }[]>; // by parameter key
  edits: Edit[];                    // cumulative; applyEdits replays all of them every time
  errors: ValidationError[];
  deid: { status: "none" | "reviewing" | "accepted"; pending?: { xml: string; report: DeidReport } };
  expandedGroups: string[];
  busy: boolean;
};
```

- **Keep the XML out of deep reactivity:** store XML as strings; never put a `Document` or DOM node in reactive state. Use `shallowRef`/`markRaw` for large objects such as `pending` and the registry.
- Changing a value replaces any earlier edit for the same `key` and `xpath`, then triggers `applyEdits` after a 400 ms debounce.
- `deid.status`: `none` after load; `reviewing` after `deidentify` returns (result in `pending`, working state untouched); `accepted` after Accept.
- **Accept:** set `originalXml` and `workingXml` to `pending.xml`, rebuild `rows` by reading all xpaths, clear `edits`, drop `pending`, set `accepted`. The original production XML must no longer be referenced anywhere afterwards (overwrite the old strings).
- **Cancel:** drop `pending`, status back to `none`; the studio stays locked.
- Switching the source to Test data sets `deid.status` to `none` and unlocks. Switching back to Production never reuses an earlier acceptance. Loading a new file resets everything.

## 8. UI (inside the existing page)

```
Block 1  KVK Studio: source and mode
  [file chooser / paste XML]      Source: (Test data | Production)
                                  Mode:   (Edit extract | Copy into new XML)
                                  COPY only: [New KVK number] [New name (optional)]
  Production banner (not yet de-identified): explanation + [De-identify] button
  Note (production, de-identified): "The original production XML is no longer held in this browser."

Review panel (only while reviewing): [Cancel] [Accept and use this XML]
  Replaced (counts) | Before → after | Also replaced in places not in the parameter list | Warnings

Block 2  Fields from the registry     [Filter]   shield icon = replaced on de-identification
  one section per registry group, in registry order, each collapsible
    one row per parameter: label (+ shield, required marker) + hint
                         | one line per xpath: (priority badge) (xpath, monospace) [typed input] [× remove]

Block 3  Raw XML (collapsed by default, read-only)

Sticky bar: "<n> fields changed · not uploaded yet"   [Download XML] [Upload to database]
```

Behaviour:

- **Source:** file upload or paste. Production is the default for every new load. A "De-identified" badge appears next to the heading after Accept.
- **Production lock:** while source is Production and nothing has been accepted, the field area is dimmed and not clickable, and both bar buttons are disabled with the note "Upload is locked until the production extract is de-identified".
- **Edit mode:** the KVK number is read from the XML and shown read-only. **Copy mode:** the new KVK number is required (8 digits), is written into the XML and used as the upload path parameter.
- **Groups:** shown in registry order. A group with `collapsedWhenEmpty` whose parameters have no node in the XML is collapsed with an "Add" button that expands it; expanding makes its fields editable so values can be filled in (nodes are created by `set`). Groups with data start expanded.
- **Typed inputs (`FieldInput.vue`):** `text` and `number` use a text input; `date` uses a date input matching `dateFormat`; `enum` uses a select with the options (plus an empty choice if not mandatory); `boolean` uses a select with `true` and `false`. Show a required marker for mandatory parameters and a shield icon for parameters whose `deid.strategy` is not `keep`; the shield's tooltip names the replacement file id.
- **Path entries:** the first xpath entry with a non-empty value gets the highlighted badge. A path whose node does not exist shows an empty input with the placeholder "(empty)"; typing a value creates a `SET` (the node gets created). The × button creates a `REMOVE` and greys the entry; typing a value there replaces the `REMOVE` with a `SET`. An entry whose value differs from the original is tinted.
- **Validation errors** appear under the parameter named by `error.key`; errors without a key appear in a banner above the fields.
- **Filter:** matches label or key, and hides groups with no match.
- **Raw XML drawer:** shows `workingXml`; render the text area only while open.
- **Sticky bar:** the counter shows the number of changed entries. **Download XML** saves `workingXml` as a file. **Upload to database** is enabled when there is at least one change (or a new KVK number in Copy mode), validation passes and the source is not locked.
- **Replace warning:** before every upload show a confirmation: "This replaces the existing extract for KVK number <n>."
- **Upload** (`api/uploadExtract.ts`): turn `workingXml` into a `File` (`application/xml`, name `<kvkNumber>.xml`), call the existing upload function with the KVK number, and show the response string on success or the error on failure. Never include XML or values in an error message.

Visual style: match the existing page and project components first. Where nothing exists, use the mock-up's values (page background `#f4f6fa`, white cards with 10 px radius and `#dde2ec` border, primary blue `#1f4fd8`, changed-value tint `#eef3ff`, production banner `#fff4de`, de-identified badge `#dcf3e4`, raw XML area `#101828`, xpaths in a monospace font).

## 9. Tests (Vitest)

Use jsdom for DOM-based tests and verify early that `document.evaluate` works there; if it does not, tell me before choosing a workaround.

- **registry:** every load-time check passes for the provided file, and each check fails on a deliberately broken copy (unknown file id, missing field, cycle, duplicate key, bad xpath).
- **xml:** read, SET existing, SET missing (indexes, attributes, `[@attr="value"]`), REMOVE, namespaces, malformed XML, DOCTYPE rejected, a parameter whose first xpath is empty.
- **validate:** each rule in section 5, including enum, boolean, pattern, maxLength and Copy mode.
- **deid** (using fixtures from section 11 and small purpose-built replacement files): the same person in three places gets the same replacements everywhere; all parameters of one address come from one record (postcode matches city); a `filter` picks a record of the matching gender; no swept original value remains in the output; output is well-formed and passes validation; shifted dates keep their format and an adult stays an adult; KVK number and RSIN are unchanged; two runs differ; BSN fakes pass the 11-test; `compose` and `initials` follow their sources; a 2 MB generated sample finishes within about 1 second.
- **store:** edit replay, Accept clears the original, Cancel keeps the lock, collapsed group expands.

## 10. Build order (one paste-able request per step; check each before the next)

1. **Skeleton and registry:** folder, `types.ts`, registry and replacement loaders with all checks and tests, feature-flag guard (`kvkExtractStudio`), `KvkStudioPanel` mounted on the existing page with a placeholder.
2. **XML module and validation** with tests (sections 4 and 5), plus `testing/fixtures.ts`.
3. **Store and screen for Test data:** source and mode block, groups and typed fields, raw XML drawer, sticky bar, no upload yet; click through with a fixture.
4. **Upload:** wire `uploadExtract.ts` to the existing function, the replace confirmation and the result display.
5. **De-identification module** with tests (section 6); then the production banner, review panel, lock, and Accept/Cancel flow.
6. **Polish and checks:** the performance test, an accessibility pass, and a check that nothing logs or stores XML.

## 11. Synthetic fixtures

`testing/fixtures.ts` builds a minimal valid XML from the registry: for every parameter, create nodes for its first xpath with a plausible value of its `type` (enum: first option; date: a valid date; number: digits; text: a word of the right length; BSN-like fields: a valid 11-test number), and repeat person and address values across several parameters so consistency can be tested. It also builds a large variant (about 2 MB) by repeating a "history" element with person and address text. Do not commit any real company or person data anywhere.

## 12. Done when

- Test data flow: load, edit a value in a split address field, pick an enum value, fill an empty path, remove a node, expand a collapsed director group and fill it, switch to Copy, upload, see the response.
- Production flow: load with Production selected: locked; De-identify; review; Cancel keeps the lock; De-identify again and Accept: badge shown, editing and upload work on the replaced values.
- All new code is under the studio folder; the only other change is the line that renders the panel on the existing page.
- Changing the registry file (a new field, a different replacement file) changes the screen and the de-identification with **no code change**.
- No XML or values appear in console output; nothing is kept in browser storage; the studio is absent when the flag is off.

## 13. Open items to confirm with me

- How this frontend reads feature flags (the flag name is `kvkExtractStudio`).
- Whether the existing upload function can be imported as is.
- Whether a design system exists that should replace the reference style values.
