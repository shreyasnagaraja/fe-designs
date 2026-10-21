# KVK Extract Editor: frontend spec (v1 + v2)

For a GitHub Copilot session in the **frontend repo only** (Vue + TypeScript + Vite). The backend is specified in a separate file and built in another session. Both sides follow the API contract below. The visual reference is `kvk-editor-mockup.html` (open it in a browser; optional, this file is enough to build from).

## 0. What this builds

One page in the existing MPMA frontend where a developer loads a KVK company extract XML and can:

1. see every parameter MPMA reads from it, with all its xpaths,
2. edit values, fill empty ones, remove nodes, or copy the extract into a new one,
3. save the result as a new XML through the backend,
4. (v2) for extracts that come from production: de-identify personal data first, review what was replaced, and only then edit and save.

Rules for the whole session:

- Vue + TypeScript + Vite, the project's existing structure, router, HTTP client, styling approach and component conventions. **No new libraries.**
- The page and its menu entry are shown only when the project's existing string-based feature flag is on, placeholder name `kvkExtractSandbox` (replace with the real one). Non-prod only.
- No business logic in the browser beyond what is listed here: the backend parses, applies edits, validates and de-identifies.
- Never log XML or extracted values to the console. Do not store the XML in localStorage, sessionStorage or cookies.
- Accessible markup: real buttons, labels on inputs, `aria-label` on icon-only buttons, visible focus, touch targets at least 44 px.
- Build against the contract using a mock client first (section 6), so this session does not need the backend running.

## API contract (identical in the BE and FE spec; do not change it in one session only)

Base path `/kvk-editor`. JSON in and out unless noted. Every endpoint returns **404 when the feature flag is off**. XML travels as a string field. The backend is stateless: the browser sends the XML with every call.

| Method and path | Request body | Success response |
| --- | --- | --- |
| `GET /kvk-editor/parameters` | none | `Parameter[]` |
| `POST /kvk-editor/parse` | `{ xml }` | `{ parameters: ParsedParameter[] }` |
| `POST /kvk-editor/apply` | `ApplyRequest` | `{ xml, errors: ValidationError[] }` |
| `POST /kvk-editor/deidentify` | `{ xml }` | `{ xml, report: DeidReport }` |
| `POST /kvk-editor/save` | `SaveRequest` | `200 { uploadResult }` or `422 { errors: ValidationError[] }` |

Malformed XML in any request: `400 { errors: [{ message }] }`.

```ts
type Parameter = {
  key: string;              // stable id, e.g. "tradeName"
  label: string;
  hint?: string;
  mandatory: boolean;
  type: "text" | "date" | "number";
  pii?: "personName" | "dob" | "address" | "companyName" | "companyAddress" | "idNumber";
  group?: string;           // parameters with the same group are replaced as one unit
  xpaths: string[];         // priority order; first non-empty one is the value MPMA reads
};

type ParsedParameter = {
  key: string;
  paths: { xpath: string; exists: boolean; value: string }[]; // same order as Parameter.xpaths
};

type Edit = { key: string; xpath: string; action: "SET" | "REMOVE"; value?: string };

type ApplyRequest = {
  xml: string;                       // the original (or de-identified) XML, never a partially edited one
  mode: "EDIT" | "COPY";
  newKvkNumber?: string;             // COPY only
  newName?: string;                  // COPY only, optional
  edits: Edit[];                     // the full cumulative list, replayed on every call
};

type SaveRequest = ApplyRequest & {
  sourceType: "TEST" | "PRODUCTION";
  deidentified: boolean;             // must be true for PRODUCTION
};

type ValidationError = { key?: string; xpath?: string; message: string };

type DeidReport = {
  counts: Record<string, number>;    // per pii category
  pairs: { kind: string; before: string; after: string }[];   // for the review screen only
  unlisted: { path: string }[];      // places outside the parameter list that were also replaced
  warnings: { path?: string; message: string }[];
};
```

Rules both sides rely on:

- `apply` always starts from `xml` and replays `edits` in order, so the browser keeps the cumulative edit list and sends it each time.
- After a successful `deidentify` the browser uses the returned `xml` as the new original and clears its edit list.
- `SET` on a path whose node is missing creates the node. `REMOVE` deletes only that node.
- Values are plain strings; dates are `yyyy-MM-dd`.
- The `report` is returned to the browser only. It is never logged or stored.

## 1. Screen layout

One page, a sticky bottom bar. Wide tables scroll horizontally inside their box; at phone width blocks stack.

```
Header:  KVK Extract Editor   [ACC · non-prod only]  [De-identified badge, v2]      Feature flag: <name> on

Block 1  Source extract and mode
  [ file box: file name, size, "Upload / paste" button ]      Source: (Test data | Production)
                                                               Mode:   (Edit extract | Copy into new XML)
                                                               COPY only: [New KVK number] [New name (optional)]
  Banner (Production, not yet de-identified): explanation + [De-identify] button
  Note (Production, de-identified): "The original production XML is no longer held in this browser."

Review panel (only while reviewing a de-identification)
  [Cancel] [Accept and use this XML]
  Replaced: counts per category | Before → after list | Also replaced in unlisted places | Warnings

Block 2  Parameters read from the extract         [Filter parameters]   shield icon legend
  Per parameter (one block each):
    label (+ shield icon if pii) + hint        |  one line per xpath, in priority order:
                                               |  (priority badge) (xpath, monospace) [value input] [× remove]

Block 3  Raw XML (collapsed by default)  [Show/Hide]  read-only dark textarea

Sticky bar:  "<n> fields changed · not saved yet"    [Download XML] [Save as new XML to database]
```

Behaviour of each element:

- **Source box:** upload a file or paste XML. On load, call `parse` and `GET /parameters` (the latter once per page load) and build the rows. A new file resets everything: edits, mode, review, badge.
- **Source switch:** Test data or Production. Production is the default for every new upload.
- **Mode switch:** Edit keeps the extract's KVK number. Copy shows the two inputs; the new KVK number is required (8 digits).
- **Production banner and lock:** while source is Production and no de-identification has been accepted, the parameter table is dimmed and not clickable, and both bar buttons are disabled with the note "Save is locked until the production extract is de-identified".
- **Parameter rows:** each xpath entry shows the priority badge (the first entry with a non-empty value is highlighted), the xpath, a value input and a × button. A path whose node does not exist shows an empty input with the placeholder "(empty)"; typing a value creates a `SET` edit (the backend creates the node). Editing an existing value also creates a `SET`. The × button creates a `REMOVE` edit and greys that entry (value cleared, strikethrough xpath); typing a value in it again replaces the `REMOVE` with a `SET`. An entry whose current value differs from the original is tinted. Parameters marked `mandatory` show a small required marker; `pii` parameters show a shield icon.
- **Validation errors** from `apply` appear under the parameter named in `error.key`; errors without a key appear in a banner above the table.
- **Raw XML drawer:** read-only; shows the `xml` from the latest `apply` (or the original before any edit). Refresh it after edits with a debounce of about 400 ms.
- **Sticky bar:** the counter shows the number of changed entries (edits whose value differs from the original, plus removals). **Download XML** saves the latest `xml` as a file. **Save** is enabled when there is at least one change, or a new KVK number in Copy mode, and the source is not locked; it sends `SaveRequest` and shows the id or response of `uploadResult` on success; on `422` it shows the errors as for `apply`.

## 2. State model

Keep it in one composable (for example `useKvkEditor`) so components stay thin.

```ts
type EditorState = {
  originalXml: string;                 // replaced after an accepted de-identification
  workingXml: string;                  // latest xml returned by apply (or originalXml)
  fileName: string;
  sourceType: "TEST" | "PRODUCTION";
  mode: "EDIT" | "COPY";
  newKvkNumber: string;
  newName: string;
  parameters: Parameter[];             // from GET /parameters
  rows: Record<string, { xpath: string; exists: boolean; originalValue: string; value: string; removed: boolean }[]>;
  edits: Edit[];                       // cumulative; sent in full on every apply and save
  errors: ValidationError[];
  deid: { status: "none" | "reviewing" | "accepted"; pending?: { xml: string; report: DeidReport } };
};
```

Rules:

- Changing a value updates `edits` (replace an earlier edit for the same `key` and `xpath`) and triggers a debounced `apply`.
- `deid.status`: `none` after upload; `reviewing` after `deidentify` returns (the result is held in `pending`, the working state is not touched yet); `accepted` after **Accept**.
- **Accept:** set `originalXml` and `workingXml` to `pending.xml`, call `parse` again to rebuild the rows, clear `edits`, drop `pending`, set status `accepted`. The original production XML must no longer be referenced anywhere afterwards (clear the old string).
- **Cancel:** drop `pending`, status back to `none`; the source stays locked.
- Switching the source to Test data sets `deid.status` to `none` and unlocks the screen. Switching back to Production does not reuse an earlier acceptance.

## 3. Components

| Component | Responsibility |
| --- | --- |
| `KvkExtractEditorPage` | Layout, feature-flag guard, owns `useKvkEditor` |
| `SourcePanel` | File box, source switch, mode switch, copy inputs, production banner and note |
| `DeidReviewPanel` | Review panel with counts, before/after list, unlisted places, warnings, Cancel and Accept |
| `ParameterTable` | Header, filter, lock state, list of `ParameterRow` |
| `ParameterRow` | One parameter: label, hint, shield, required marker, its path entries |
| `PathEntry` | Badge, xpath, value input, remove button |
| `RawXmlDrawer` | Collapsible read-only XML |
| `SaveBar` | Sticky bar with counter, Download and Save, status and errors |
| `kvkEditorApi.ts` | The five calls from the contract; a mock implementation (section 6) behind the same interface |

## 4. Flows to implement and check by hand

1. **Test data, edit:** upload, change a value, fill an empty path, remove a node, open the raw XML, save.
2. **Copy:** switch to Copy, enter a new KVK number (try an invalid one: error shown), save.
3. **Production:** upload with Production selected: banner visible, table locked, Save disabled. Click De-identify: review panel shows. Cancel: still locked. De-identify again, Accept: badge appears, original dropped, rows show the replaced values, editing and saving work.
4. **Errors:** a mandatory field emptied shows its error under the parameter; a `422` on save shows errors; malformed XML on upload shows a banner.

## 5. Visual style

Match the mock-up and the project's existing look. Reference values from the mock-up: page background `#f4f6fa`, white cards with 10 px radius and a `#dde2ec` border, primary blue `#1f4fd8`, changed-value tint `#eef3ff`, production banner amber `#fff4de`, de-identified badge green `#dcf3e4`, raw XML area dark `#101828`, xpaths in a monospace font. If the project has a design system, use its components and tokens instead.

## 6. Mock client and fixtures

Implement `kvkEditorApi.ts` as an interface with a real HTTP implementation and a mock one, chosen by an environment variable, so the page can be built and clicked through without the backend. The mock returns these shapes (illustrative data, all fictional):

```json
// GET /kvk-editor/parameters
[
  { "key": "kvkNumber", "label": "KVK number", "mandatory": true, "type": "number", "xpaths": ["/kvkExtract/kvkNummer"] },
  { "key": "tradeName", "label": "Trade name", "hint": "3 paths", "mandatory": true, "type": "text", "pii": "companyName",
    "xpaths": ["/kvkExtract/handelsnaam/volledigeNaam", "/kvkExtract/eersteHandelsnaam", "/kvkExtract/statutaireNaam"] },
  { "key": "director1Name", "label": "Director 1 name", "mandatory": false, "type": "text", "pii": "personName", "group": "director1",
    "xpaths": ["/kvkExtract/bestuurder[1]/volledigeNaam", "/kvkExtract/bestuurder[1]/handtekening/naam"] },
  { "key": "director1Dob", "label": "Director 1 date of birth", "mandatory": false, "type": "date", "pii": "dob", "group": "director1",
    "xpaths": ["/kvkExtract/bestuurder[1]/geboortedatum"] }
]

// POST /kvk-editor/parse
{ "parameters": [
  { "key": "tradeName", "paths": [
    { "xpath": "/kvkExtract/handelsnaam/volledigeNaam", "exists": true, "value": "Voorbeeld Holding B.V." },
    { "xpath": "/kvkExtract/eersteHandelsnaam", "exists": false, "value": "" },
    { "xpath": "/kvkExtract/statutaireNaam", "exists": true, "value": "Voorbeeld Holding BV" } ] }
] }

// POST /kvk-editor/deidentify
{ "xml": "<kvkExtract>…</kvkExtract>",
  "report": {
    "counts": { "personName": 3, "dob": 1, "address": 2, "companyName": 2, "idNumber": 1 },
    "pairs": [ { "kind": "Name", "before": "A. Voorbeeld", "after": "M. Bakker" } ],
    "unlisted": [ { "path": "/kvkExtract/historie/mutatie[3]/omschrijving" } ],
    "warnings": [ { "path": "/kvkExtract/registratie/rsin", "message": "Looks like a BSN but was not replaced; it may be an RSIN." } ] } }

// POST /kvk-editor/apply  and  POST /kvk-editor/save (422)
{ "xml": "<kvkExtract>…</kvkExtract>", "errors": [ { "key": "tradeName", "message": "Required" } ] }
```

Mock behaviour: `apply` returns the posted `xml` with the edits listed in a comment at the top, and one error if a `mandatory` parameter's value is empty; `deidentify` returns the fixture above; `save` returns `{ "uploadResult": { "id": "mock-1" } }`.

## 7. Done when

- All flows in section 4 work against the mock client.
- Switching `VITE_` environment configuration to the real client needs no component changes.
- No XML or values appear in console output; nothing is stored in browser storage.
- The page is absent (no route, no menu entry) when the feature flag is off.

## 8. Open items to confirm with me

- The real feature flag name and how the frontend reads flags.
- The project's HTTP client and error-handling conventions.
- Whether a design system exists that should replace the reference style values.
