# KVK Extract Editor – implementation spec

Oct 21, 2026 · @Sn

## Purpose, scope and flow

A small non-prod screen in the MPMA customer data app lets you load a KVK extract XML, edit or copy it, and save the result as a new row through the existing upload endpoint. It is gated by the project's existing string-based feature flag and deployed to every non-prod environment through the normal Azure DevOps pipeline.

UI reference: the Design canvas mock-up built in this chat (one page, three blocks, sticky save bar).

1. Upload or paste a KVK extract XML. The backend parses it and returns every parameter from the reference JSON, with one row per xpath.
2. Choose **Edit extract** (keeps the KVK number) or **Copy into new XML** (asks for a new KVK number and optional new name).
3. Edit values per xpath. A parameter with several xpaths shows all of them in priority order, including empty ones, and each is editable. The × button removes that node from the XML.
4. Optionally open the raw XML drawer; it stays in sync with the fields.
5. Save: the backend validates, builds the final XML and calls the existing upload endpoint, which stores it in the Azure SQL table.

## Reference JSON for xpath parameters

The parameter list lives in one JSON file in the backend repo (`src/main/resources/kvk-parameters.json`). It rarely changes, so a DB table or admin screen is not needed. The backend serves it through one GET endpoint, so the frontend never keeps its own copy.

Each parameter has:

- `key`: stable id, e.g. `tradeName`
- `label` and `hint`: shown in the UI
- `mandatory`: true or false, checked before saving
- `type`: `text`, `date` or `number`, used for validation
- `xpaths`: array in priority order; the first non-empty one is the value MPMA reads

Sample (illustrative paths, replace with your real list):

```json
[
  {
    "key": "tradeName",
    "label": "Trade name",
    "mandatory": true,
    "type": "text",
    "xpaths": [
      "/kvkExtract/handelsnaam/volledigeNaam",
      "/kvkExtract/eersteHandelsnaam",
      "/kvkExtract/statutaireNaam"
    ]
  }
]
```

If the XML uses namespaces, add a `namespaces` map (prefix to URI) at the top of the file and use the prefixes in the xpaths.

## Backend (Quarkus)

One REST resource, stateless: the browser sends the XML back with every call, so no session or temp storage is needed. Use the JDK's `javax.xml.xpath` and DOM; MapStruct is not needed here.

| Endpoint | Input | Output |
| --- | --- | --- |
| `GET /kvk-editor/parameters` | none | The reference JSON |
| `POST /kvk-editor/parse` | XML text | Per parameter, per xpath: whether the node exists and its value |
| `POST /kvk-editor/apply` | XML, mode, optional new KVK number and name, list of edits | Updated XML plus validation errors; powers the raw XML drawer |
| `POST /kvk-editor/save` | Same as apply | Runs apply, refuses on validation errors, then calls the existing upload endpoint and returns its result |

An edit is `{key, xpath, action, value}` with action `SET` or `REMOVE`.

- **SET on an existing node:** replace its text. **SET on a missing node:** create the elements along the path, then set the text.
- **REMOVE:** delete that node only; the other xpaths of the parameter are untouched.
- **Creating nodes** works for plain child-element paths. For paths with predicates such as `[@primair="true"]`, create the element and the attribute from the predicate; if a path is too complex to create, return a clear error for that parameter.
- **Copy mode:** write the new KVK number to its xpath (and the new name if given) before applying other edits.
- **Safety:** disable DTDs and external entities when parsing; never log XML content, because extracts can hold real company data; return 404 on every endpoint when the feature flag is off.
- **Upload call:** reuse the existing client or call the endpoint over REST with the pipeline's usual auth; the new code never touches the SQL table itself.

## Frontend (Vue)

One page in Vue + TypeScript + Vite, shown only when the feature flag is on, in the existing app shell. No new libraries; plain components and `fetch`/the project's HTTP client.

- **State:** one reactive object holding the original XML, the mode, the optional new KVK number and name, and the parameter rows. Each row has a list of path entries `{xpath, exists, originalValue, value, removed}`.
- **Layout:** follow the mock-up: source box with Edit/Copy switch, parameter table with one line per xpath (priority badge, xpath, value input, × button), collapsible raw XML drawer, sticky bar with Download XML and Save.
- **Empty nodes:** a path whose node does not exist shows an empty input. Typing a value sends a SET, which creates the node on the backend.
- **Remove:** the × button marks the entry removed (greyed, can be undone by typing a value again) and sends REMOVE.
- **Highlighting:** an entry whose value differs from the original is tinted; the sticky bar counts changed entries.
- **Raw XML drawer:** read-only by default; refreshed by calling `apply` after changes (debounced). Direct editing of the raw XML is out of scope for v1.
- **Mandatory fields:** parameters marked mandatory show a required marker on the parameter, and errors from the backend appear next to the parameter.
- **Save:** disabled until at least one change or a new KVK number in copy mode; on success show the id returned by the upload endpoint.

## Validation and save

Validation runs on the backend inside `apply`, so the screen and `save` behave the same. `save` is refused while any error exists.

| Rule | Check |
| --- | --- |
| Well-formed | The XML parses; otherwise one error for the whole file |
| Mandatory | A parameter with `mandatory: true` needs a non-empty value on at least one of its xpaths |
| Type | `date` must be `yyyy-MM-dd`; `number` must be an integer; `text` has no check |
| Copy mode | The new KVK number is required, 8 digits, and different from the source |

The `mandatory` and `type` fields come from the reference JSON, so adding a rule means editing that file only. Errors return as `{key, xpath?, message}` so the screen can show them next to the right parameter.

## Out of scope and build order

Out of scope for v1: direct editing of the raw XML, undo per field, adding repeated list items such as extra bestuurders, authentication beyond what the app already has, and production use.

Suggested build order, each step small enough for one Copilot prompt:

1. Add `kvk-parameters.json`, the DTO classes and the GET parameters endpoint, behind the feature flag.
2. Add the XML service: parse with hardened settings, read values per xpath, apply SET and REMOVE, create missing nodes. Unit-test it with a sample extract.
3. Add validation and the `parse`, `apply` and `save` endpoints, with `save` calling the existing upload endpoint.
4. Build the Vue page from the mock-up with static data, then connect it to the endpoints.
5. Add the pipeline flag setting per non-prod environment and smoke-test with a de-identified extract in Acceptance.

Open items: the real xpath list with mandatory flags and types, the name of the feature flag, and the exact request and auth shape of the existing upload endpoint.

# v2 add-on: de-identifying production extracts

v2 adds one step to v1: when an extract comes from production, the tool replaces personal data before anything can be edited or saved. Everything in v1 (edit, copy, xpath rows, validation, save) is unchanged and works on the de-identified XML afterwards.

## Decisions

- **Scope:** bestuurder names, dates of birth, person addresses, and company names and addresses.
- **Consistency:** within one file only. A fresh random key is created for every de-identify run and discarded afterwards, so running it twice on the same file gives different fakes and nothing links two files.
- **Unlisted places:** after the listed xpaths are replaced, every other occurrence of the same original values anywhere in the XML is replaced too, with the same fake value.
- **Dates of birth:** shifted by a fixed offset per person.

## Identifiers and gender

- **BSN and TIN are replaced** wherever they appear: add them to the reference JSON as `pii: "idNumber"`. KVK number and RSIN are kept as they are.
- **BSN replacement:** a fake 9-digit number that passes the 11-test, so it stays valid. **TIN replacement:** keeps the original length and character pattern (digits stay digits, letters stay letters), because foreign TIN formats vary.
- **Unlisted places:** the sweep replaces every other occurrence of an original BSN or TIN value with the same fake. The sweep does not replace numbers only because they look like a BSN: RSIN has the same 9-digit shape and passes the same test, so such numbers are listed as warnings for you to check.
- **Gender:** if the XML has a gender element, add it to the reference JSON as `genderXpath`, and first names are picked from the matching list. Without it, names come from a neutral list. The gender value itself is not changed.

## Reference JSON additions

Add `pii` to each parameter that holds personal data; leave it out for the rest.

| `pii` value | Used for | Replacement |
| --- | --- | --- |
| `personName` | Bestuurder and other person names | Fake Dutch name from a list, keeping initials and tussenvoegsel format |
| `dob` | Dates of birth | Original date shifted by the person's offset, same format |
| `address` | Person addresses (street, number, postcode, city as one group) | One coherent fake address from a list |
| `companyName` | Company and trade names | Fake name keeping the legal form, e.g. B.V. |
| `companyAddress` | Company addresses | Same as `address` |

Parameters that belong together (a person's name and date of birth; an address's parts) share a `group` id so they are replaced as one unit.

## Backend: de-identify endpoint

Add `POST /kvk-editor/deidentify` with the XML as input. It returns the de-identified XML plus a report. The backend stays stateless; the browser replaces its working XML with the returned one, so the random key never has to be sent back.

1. **Collect.** For every `pii` parameter, read the values at all its xpaths. Normalise each (trim, lowercase, strip accents, collapse spaces) and group by `group` id. Each distinct normalised person or address becomes one entity.
2. **Generate.** Create a random key. For each entity, take HMAC-SHA256 of key plus category plus normalised value and use the result to pick an entry from the fake lists in `src/main/resources`. If two different originals pick the same fake, probe to the next entry, so fakes stay unique.
3. **Replace listed xpaths.** Write the fake into every xpath of that parameter, including the lower-priority ones.
4. **Replace unlisted occurrences.** Build a replacement table of every original value and its variants: full name, surname alone (only if 4 or more characters), initials plus surname, street name, postcode. Walk all text nodes and attribute values, replace matches case-insensitively on word boundaries, longest match first, and record each place changed.
5. **Final check.** Search the result for every original value and variant; any survivor goes into the report as a warning.
6. **Date offsets.** Each person gets one offset between 61 and 365 days, never zero, from the same HMAC. If the shifted date would make the person under 18 or after the company's foundation date, clamp it.

The report holds counts per category, the list of unlisted places that were changed (path only), warnings, and before/after pairs for the review screen. It is returned to the browser only; never log it, never persist it.

## Frontend changes

- **Source switch** in block 1: **Source: test data** or **Source: production**. Production is the safe default for any upload that has not been de-identified.
- **PII marker:** parameters with a `pii` value show a small shield icon in the table.
- **De-identify button** appears for production source. It calls the endpoint and opens a review panel: counts per category, the unlisted places that were changed, any warnings, and a before/after list.
- **Accept** replaces the working XML with the de-identified one, drops the original from browser memory and shows a **De-identified** badge. **Cancel** keeps the original and keeps Save disabled.
- **Save** stays disabled for production source until the de-identify step is accepted. After that, all v1 editing and validation work as before.

## Safeguards

- The original production XML is never written to the database, files or logs; only the de-identified result is saved.
- Audit log per run: user, time, source file name, counts per category and number of warnings; never values.
- The save endpoint also takes the source type. For production it refuses a request not marked as de-identified. This is a guard against mistakes, not against someone bypassing the UI, because the stateless backend cannot prove the data was changed.
- This is pseudonymization, not anonymization. Confirm with your data protection contact that it meets the conditions of the production-data approval.

## Acceptance checks

- The same name in three places gets the same fake in all three.
- No original value or variant remains anywhere in the output.
- The output is well-formed and passes the v1 validation rules.
- Postcode and city in a fake address match each other.
- Shifted dates are valid, keep their format, and no one becomes under 18.
- Running de-identify twice on one file gives different results.

## Build order for Copilot

1. Add `pii` and `group` to the reference JSON and DTOs, plus the fake name, street and postcode/city lists.
2. Build the entity collector and generator with unit tests for consistency and uniqueness.
3. Build the replacement and sweep step, the final check and the report; test with a sample extract.
4. Add the `deidentify` endpoint and the source type guard on `save`.
5. Add the source switch, shield icons, button and review panel to the Vue page.

Open items: the real list of BSN and TIN xpaths, the gender element if there is one, and the first real extract to test the sweep against.
