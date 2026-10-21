# KVK Extract Editor v1+v2 – build spec in parts

Oct 21, 2026 · @Sn

## How to use this spec

The work is split into six parts. Each part lists what it needs, what to build and when it is done, so you can give Copilot one part at a time and check it before the next. Part 1 to 3 give the v1 tool (edit and copy). Part 4 to 6 add de-identification of production extracts.

| Part | What it builds | Needs |
| --- | --- | --- |
| 1 | Reference JSON endpoint and XML service (parse, SET, REMOVE) | Shared reference JSON |
| 2 | Validation, `apply` and `save` endpoints | Part 1 |
| 3 | Vue screen: upload, edit/copy, xpath rows, raw XML, save bar | Part 2 |
| 4 | De-identification engine (pure Java class, no endpoint) | Part 1 |
| 5 | `deidentify` endpoint and the save guard for production | Parts 2 and 4 |
| 6 | Vue source switch, shield icons and review panel | Parts 3 and 5 |

Part 4 only needs Part 1, so it can be built in parallel with Part 2 and 3.

General rules for every part:

- Stack: Quarkus REST backend and a Vue + TypeScript + Vite frontend in the existing MPMA app; no new libraries.
- Everything is behind the project's existing string-based feature flag and runs in non-prod only; every endpoint returns 404 when the flag is off.
- The backend is stateless: the browser sends the XML with every call.
- Never log XML content or values; extracts can hold real company data.
- Parse XML with DTDs and external entities disabled.
- Give Copilot the shared reference JSON section plus only the part you are building. Each part ends with a line you can paste as the request.

## Shared: reference JSON

One JSON file in the backend repo, `src/main/resources/kvk-parameters.json`, lists every parameter read from the KVK extract. It rarely changes, so it is not a DB table. The backend serves it from one GET endpoint and the frontend keeps no copy.

Top level: an optional `namespaces` map (prefix to URI) and a `parameters` array. Each parameter has:

| Field | Meaning | Used in |
| --- | --- | --- |
| `key` | Stable id, e.g. `tradeName` | Parts 1 to 6 |
| `label`, `hint` | Text shown on the screen | Part 3 |
| `mandatory` | true or false; checked before saving | Part 2 |
| `type` | `text`, `date` or `number` | Part 2 |
| `xpaths` | Array in priority order; the first non-empty one is the value MPMA reads | Parts 1 to 6 |
| `pii` | Optional: `personName`, `dob`, `address`, `companyName`, `companyAddress` or `idNumber` (BSN and TIN) | Parts 4 to 6 |
| `group` | Optional id; parameters with the same group are replaced as one unit (a person's name and date of birth, the parts of an address) | Part 4 |
| `genderXpath` | Optional, on a `personName` parameter: where the person's gender sits | Part 4 |

KVK number and RSIN carry no `pii` value, so they are never replaced.

Sample (illustrative paths; replace with your real list):

```json
{
  "namespaces": {},
  "parameters": [
    {
      "key": "tradeName",
      "label": "Trade name",
      "mandatory": true,
      "type": "text",
      "pii": "companyName",
      "xpaths": [
        "/kvkExtract/handelsnaam/volledigeNaam",
        "/kvkExtract/eersteHandelsnaam",
        "/kvkExtract/statutaireNaam"
      ]
    },
    {
      "key": "director1Name",
      "label": "Director 1 name",
      "mandatory": false,
      "type": "text",
      "pii": "personName",
      "group": "director1",
      "xpaths": ["/kvkExtract/bestuurder[1]/volledigeNaam"]
    }
  ]
}
```

## Part 1: backend XML service and reference endpoint

**Needs:** the shared reference JSON. **Goal:** read and change values in a KVK extract XML by xpath, and serve the parameter list.

Build:

1. **DTOs and loader** for the reference JSON, loaded once at startup.
2. **`GET /kvk-editor/parameters`** returning the reference JSON.
3. **`POST /kvk-editor/parse`** with XML text as body. It returns, per parameter and per xpath, `{xpath, exists, value}`. A path whose node is missing has `exists: false` and an empty value, so the screen can show it as an empty, editable row.
4. **XML service class** with the core operations, free of REST code so other parts can reuse it:
   - parse text to a DOM document (hardened settings, namespaces from the JSON);
   - read the value at an xpath;
   - `SET`: replace the text of an existing node; if the node is missing, create the elements along the path and then set the text;
   - `REMOVE`: delete that node only, leaving the parameter's other xpaths untouched;
   - serialise back to XML text.
5. **Creating nodes:** supported for plain child-element paths and for predicates of the form `[@attr="value"]` (create the element and the attribute). For any path too complex to create, return a clear error naming the parameter.

**Done when:** unit tests with a sample extract pass for read, SET existing, SET missing, REMOVE, a priority parameter with an empty first xpath, and malformed XML.

**Request for Copilot:** "Implement Part 1 of the KVK Extract Editor spec using the shared reference JSON section."

## Part 2: validation and save endpoints

**Needs:** Part 1. **Goal:** apply a list of edits, validate the result, and store it through the existing upload endpoint.

Build:

1. **`POST /kvk-editor/apply`.** Input: XML, mode (`EDIT` or `COPY`), optional new KVK number and new name (copy mode), and a list of edits `{key, xpath, action, value}` where action is `SET` or `REMOVE`. Output: the updated XML plus a list of validation errors `{key, xpath?, message}`.
2. **Copy mode:** write the new KVK number (and the new name if given) to the xpaths of their parameters before applying the other edits.
3. **Validation rules,** run on the result of every `apply`:

| Rule | Check |
| --- | --- |
| Well-formed | The XML parses; otherwise one error for the whole file |
| Mandatory | A parameter with `mandatory: true` needs a non-empty value on at least one xpath |
| Type | `date` is `yyyy-MM-dd`; `number` is an integer; `text` has no check |
| Copy mode | New KVK number required, 8 digits, different from the source |

4. **`POST /kvk-editor/save`.** Same input as `apply`. It runs `apply`, refuses with the errors if there are any, otherwise calls the existing upload endpoint with the final XML and returns that endpoint's result. Reuse the existing client or REST call and the pipeline's usual auth. This code never touches the SQL table.

**Done when:** tests cover each validation rule, copy mode, a refused save, and a successful save with the upload call mocked.

**Request for Copilot:** "Implement Part 2 of the KVK Extract Editor spec on top of the Part 1 XML service."

## Part 3: Vue screen for editing and copying

**Needs:** Part 2 endpoints. **Goal:** one page, shown only when the feature flag is on, that does everything v1 needs. UI reference: the v1+v2 mock-up (Part 3 is the screen with the source set to test data).

Build:

1. **State:** one reactive object with the original XML, the mode, the optional new KVK number and name, and the parameter rows. Each row has a list of path entries `{xpath, exists, originalValue, value, removed}`.
2. **Source box:** file upload or paste for the XML; an **Edit extract** / **Copy into new XML** switch. In copy mode show inputs for the new KVK number and an optional new name.
3. **Parameter table:** one block per parameter with a line per xpath in priority order: priority badge, xpath, value input, and a × button. Paths whose node does not exist show an empty input; typing a value sends a SET, which creates the node. The × button marks the entry removed and sends REMOVE; typing a value again undoes it.
4. **Highlighting and counts:** an entry that differs from its original is tinted; the save bar counts changed entries. Mandatory parameters show a required marker; validation errors appear next to the parameter they belong to.
5. **Raw XML drawer:** collapsed by default, read-only, refreshed by calling `apply` after changes (debounced). Editing the raw XML directly is out of scope.
6. **Save bar:** sticky at the bottom with **Download XML** and **Save as new XML to database**. Save is disabled until there is at least one change, or a new KVK number in copy mode. On success show the id returned by the upload endpoint.

**Done when:** you can upload a sample, edit a value, fill an empty path, remove a node, switch to copy mode, and save with the backend running.

**Request for Copilot:** "Implement Part 3 of the KVK Extract Editor spec following the mock-up."

## Part 4: de-identification engine

**Needs:** Part 1 (XML service and reference JSON). **Goal:** a plain Java class, no REST code, that takes XML text and returns de-identified XML plus a report. Build it and test it on its own first.

Decisions:

- **Scope:** bestuurder names, dates of birth, person addresses, company names and addresses, and BSN and TIN values anywhere in the file. KVK number and RSIN are kept.
- **Consistency:** the same real value gets the same fake everywhere in one file. A fresh random key is created for each run and discarded afterwards, so nothing links two files and a second run gives different fakes.

Steps:

1. **Collect.** For every parameter with a `pii` value, read all its xpaths. Normalise each value (trim, lowercase, strip accents, collapse spaces). Group by `group` id: each distinct person or address becomes one entity; a parameter without a group is its own entity.
2. **Generate.** Take HMAC-SHA256 of the run key, the category and the normalised value, and use it to pick an entry from the fake lists in `src/main/resources` (first names, surnames with tussenvoegsels, company names, streets, and postcode with matching city). If two different originals pick the same fake, probe to the next entry so fakes stay unique.
3. **Replace listed xpaths** in every xpath of the parameter, including lower-priority ones. Rules per category:

| `pii` | Replacement |
| --- | --- |
| `personName` | Fake Dutch name keeping initials and tussenvoegsel format; first names from the matching list if `genderXpath` gives a gender, else a neutral list. The gender value is not changed |
| `dob` | Original date shifted by the person's offset, same format |
| `address` and `companyAddress` | One coherent fake address (street, number, postcode, city together) |
| `companyName` | Fake name keeping the legal form, e.g. B.V. |
| `idNumber` | BSN: fake 9-digit number that passes the 11-test. TIN: same length and character pattern (digits stay digits, letters stay letters) |

4. **Date offsets.** Each person gets one offset between 61 and 365 days, never zero, derived from the same HMAC. Clamp if the shifted date would make the person under 18 or fall after the company's foundation date.
5. **Sweep unlisted places.** Build a table of every original value and its variants: full name, surname alone (only if 4 or more characters), initials plus surname, street name, postcode, and each BSN or TIN value. Walk all text nodes and attribute values, replace matches case-insensitively on word boundaries, longest match first, and record each place changed (path only).
6. **Final check.** Search the result for every original value and variant. Survivors go into the report as warnings. Also list any 9-digit number that looks like a BSN but was not replaced (it may be an RSIN) as a warning; do not replace it.
7. **Report.** Counts per category, the unlisted places changed, warnings, and before/after pairs for the review screen.

**Done when:** unit tests cover the acceptance checks at the end of this doc, using a sample extract with repeated names and addresses.

**Request for Copilot:** "Implement Part 4 of the KVK Extract Editor spec as a standalone class with unit tests; no endpoint yet."

## Part 5: de-identify endpoint and save guard

**Needs:** Part 2 and Part 4. **Goal:** expose the engine and stop production extracts from being saved without de-identification.

Build:

1. **`POST /kvk-editor/deidentify`** with the XML as body. It calls the Part 4 engine and returns the de-identified XML plus the report. The backend stays stateless; the browser replaces its working XML with the returned one, so the random key never has to be sent back. The report is returned to the browser only and never logged or stored.
2. **Source type on save.** `apply` and `save` accept `sourceType` (`TEST` or `PRODUCTION`) and a `deidentified` flag. For `PRODUCTION`, `save` refuses a request where `deidentified` is not true. This guards against mistakes only: the stateless backend cannot prove the data was changed, so it does not protect against someone bypassing the UI.
3. **Audit log** per de-identify run: user, time, source file name, counts per category and number of warnings; never values.
4. The original production XML is never written to the database, files or logs; only the de-identified result is saved.

**Done when:** tests cover the endpoint with a sample, a refused production save without the flag, a successful production save after de-identification (upload call mocked), and the audit entry holding no values.

**Request for Copilot:** "Implement Part 5 of the KVK Extract Editor spec using the Part 4 engine."

## Part 6: Vue source switch and review panel

**Needs:** Part 3 and Part 5. **Goal:** add the production flow to the existing screen. UI reference: the v1+v2 mock-up with the source set to production.

Build:

1. **Source switch** in block 1: **Source: test data** or **Source: production**. Production is the default for any upload that has not been de-identified. For production, a banner says the data must be de-identified before it can be edited or saved.
2. **Shield icon** next to every parameter that has a `pii` value, taken from the reference JSON.
3. **De-identify button,** shown for production source. It calls `deidentify` and opens a review panel with counts per category, the unlisted places that were changed, any warnings, and a before/after list.
4. **Accept** replaces the working XML with the de-identified one, drops the original from browser memory, and shows a **De-identified** badge. **Cancel** keeps the original and keeps Save disabled.
5. **Save rules.** For production source, the Save button stays disabled until Accept; the screen then sends `sourceType: PRODUCTION` and `deidentified: true`. All Part 3 editing and validation work as before on the de-identified XML.
6. **Re-upload** of a new file resets the flow and the badge.

**Done when:** a production sample cannot be saved before de-identification, shows the review panel with counts, and saves normally after Accept.

**Request for Copilot:** "Implement Part 6 of the KVK Extract Editor spec following the mock-up."

## Acceptance checks and open items

De-identification checks (Part 4 and 5):

- The same name in three places gets the same fake in all three.
- No original value or variant, including BSN and TIN, remains anywhere in the output.
- The output is well-formed and passes the Part 2 validation rules.
- Postcode and city in a fake address match each other.
- Shifted dates are valid, keep their format, and no one becomes under 18.
- KVK number and RSIN are unchanged.
- Running de-identify twice on one file gives different results.

Open items:

- The real xpath list, with `mandatory`, `type`, `pii`, `group` and `genderXpath` filled in.
- The name of the feature flag.
- The request and auth shape of the existing upload endpoint.
- A first real extract to test the sweep against.
- Confirm with your data protection contact that pseudonymization meets the conditions of the production-data approval.
