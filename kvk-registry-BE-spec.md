# KVK Studio: backend-repo spec (generate registry v2)

For one GitHub Copilot session in the **backend repo** (the one with the MapStruct mappers). **No runtime backend code, no endpoint and no database change** is built here: the existing upload endpoint stays exactly as it is. The output is a set of static files that I copy into the frontend repo, where the studio screen and the de-identification read them.

## 0. Context and goal

I already generated a first version of the registry (`kvk-parameters.json`, xpaths with priorities, a `pii` marker per parameter) and the replacement files (fake names, addresses, ...) using an earlier spec. Version 2 changes the registry so the frontend can be completely generic:

1. **Leaf-level parameters.** One parameter per value element in the XML, so test data can be created for many scenarios. An address becomes street, house number, house number addition, postcode, city, country, and so on, wherever the XML splits them.
2. **Display groups.** Parameters sit in named groups (company, company address, director 1, director 1 home address, ...), in screen order.
3. **Typed fields.** Each parameter has a type (`text`, `date`, `number`, `enum`, `boolean`), with `options` for coded values, and optional `pattern` and `maxLength`, so the screen can offer dropdowns and catch invalid test data.
4. **De-identification per parameter.** A `deid` block says whether the parameter is replaced, with which strategy, and which replacement file (and field) to take the value from. The frontend then needs no hard-coded field names.

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

## 1. Inputs

Use what is already in the repo or that I pass in; list in your summary what you used:

- the MapStruct mapper files (paths given in the request),
- the Word document describing the KVK fields, converted first with `pandoc kvk-fields.docx -t markdown -o kvk-fields.md` (read the markdown, not the .docx),
- the XSD or the JAXB-annotated classes that bind the extract XML, if present,
- the **existing registry v1 and the existing replacement files**: start from them. Keep every xpath and priority order that is still correct; do not re-derive from scratch what v1 already got right, and report every v1 parameter you changed, split or dropped.

## 2. Procedure

1. **Inventory the replacement files.** For each existing file, decide `kind`. A JSON array of strings is `strings`; a JSON array of objects is `records` (list its property names as `fields`). Convert other layouts into one of these two (for example a file with separate `male` and `female` arrays becomes a `records` file with a `gender` property), put the results under `replacements/`, and fill `replacementFiles`. Rules for the files: clearly fictional content only (no real person's full name or real address), at least 300 entries per file, and at most about 200 KB per file because they ship in the frontend and load on demand. Address records must be coherent: the postcode matches the city and the street exists in that city, or at least looks plausible.
2. **Split to leaf parameters.** Walk the mappers and the Word document and create one parameter per XML value element (and attribute) that MPMA reads or that the Word document describes as part of the extract. Do not invent elements that are not in the XSD, the classes or the document. If the XML holds only a combined string (for example a full address in one element), keep one parameter for it; do not guess a split. If the XML has both a combined element and split elements, include both and use `compose` for the combined one.
3. **Keys and groups.** Keys are dot-separated and stable (`company.address.street`, `director1.lastName`). Create groups for the company, each address, and each repeating element. Expand repeating elements (directors, activities, establishments) to the first 3 instances unless the Word document gives a different maximum; instances 2 and 3 get `collapsedWhenEmpty: true`. Use index xpaths (`bestuurder[1]`, `bestuurder[2]`). State the instance count you used in the review file so I can change it.
4. **Priority xpaths.** As before: where a mapper takes one value from several sources, list every source in the order the mapper tries them (fallbacks, `defaultExpression`, `@Named` methods that return the first non-empty value). Resolve mapper source paths to absolute xpaths from the XML binding annotations or the XSD, using the `namespaces` map for prefixed paths. Never guess an unresolvable path: list it in the review file with the mapper file, line and what is missing, and leave that xpath out.
5. **Types and checks.**
   - `type`, `mandatory`, `maxLength`: from the Word document; if it is silent, `text`, `false`, none, and list the parameter in the review file.
   - Coded values (legal form, status, gender, country code, SBI list if short, boolean flags): `enum` with `options` from the Word document or the code lists in the repo. If a code list is long (more than 100 entries), keep `text` and add a `pattern`.
   - Add `pattern` for: KVK number (8 digits), postcode (`^[1-9][0-9]{3} ?[A-Za-z]{2}$`), BSN (9 digits), and any other format the Word document defines.
   - For `date`, set `dateFormat` to match the XML (`yyyy-MM-dd` or `yyyyMMdd`).
6. **De-identification per parameter.** Add a `deid` block with these rules, then mark every choice in the review file as "proposed, confirm":

| Parameter holds | `deid` |
| --- | --- |
| Trade name, statutory name, other company names | `pickFromFile` from the company names file; `sweepLastToken: false` |
| Last name of a natural person | `pickFromFile` from the last names file, `identity: true` |
| First name of a natural person | `pickFromFile` from the first names file, `identity: true`; if a gender parameter exists in the same entity, add `filter` using it |
| Initials | `initials` from the first-name parameter(s) |
| Name prefix (tussenvoegsel) | `keep` |
| Full name element that exists besides split ones | `compose` from the split name parameters; `sweepLastToken: true` |
| Full name element with no split parameters | `pickFromFile` is not possible: use `compose` only if split ones exist; otherwise tell me in the review file that a combined name needs a decision |
| Date of birth | `dateShift` |
| Street, house number, house number addition, postcode, city of a person's address or of the company's address | `pickFromFile` from the addresses file with the matching `field`; same `entity` for all parts of one address; `identity: true` on postcode and house number |
| Country, country code | `keep` |
| BSN | `bsn` |
| TIN and other foreign identifiers | `patternPreserving` |
| KVK number, RSIN, SBI code, legal form, dates other than a date of birth, counts, flags, status codes | `keep` (no `deid` block) |
| Phone number, e-mail address | `patternPreserving` (flag in the review file; I may want a dedicated strategy later) |

   Give every person and every address its own `entity` id (`director1`, `director1.address`, `company.address`). A person's name and date-of-birth parameters share the person's `entity`.
7. **Copy mode.** Fill `copyMode` with the keys of the KVK number and the trade name.
8. **Write the files.** `kvk-parameters.json` (formatted, two-space indent), `replacements/*.json`, and `kvk-parameters-review.md`.

## 3. Review file (`kvk-parameters-review.md`)

Sections, each a short list: v1 parameters changed, split or dropped; unresolved mapper paths; parameters with defaulted `type`/`mandatory`; every `deid` proposal "proposed, confirm" (grouped by strategy); phone and e-mail handling; combined elements without split parameters; replacement files converted or created (with size and entry count); the repeating-element instance count used.

## 4. Test (the only code to write)

A unit test (the repo's usual test framework) that loads the registry and the replacement files and runs the **load-time checks** listed in the schema, plus:

- every parameter's first xpath exists in a sample extract fixture if one is in the repo (skip with a clear message if not);
- every `records` file entry has all its listed `fields`, non-empty;
- every `strings` file entry is a non-empty string;
- no two entries in a file are identical;
- the KVK number and RSIN parameters have no `deid`;
- the total size of the files under `replacements/` is under 2 MB.

## 5. Done when

The test passes; the review file exists; I have reviewed it. **Stop after writing the files and the review file, and wait for my confirmation.** Do not build endpoints, services or frontend code.

## 6. Open items to confirm with me

- The repeating-element instance count (default 3).
- Any field the Word document marks as sensitive that is not in the table above.
- Whether phone numbers and e-mail addresses need their own replacement files.
