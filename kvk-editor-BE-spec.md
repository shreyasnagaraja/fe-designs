# KVK Extract Editor: backend spec (v1 + v2)

For a GitHub Copilot session in the **backend repo only** (Quarkus). The frontend is specified in a separate file and built in another session. Both sides follow the API contract below.

## 0. What this builds

An internal non-prod tool that takes a KVK company extract XML and lets a developer:

1. read the parameter values the MPMA app takes from it, per xpath,
2. edit them, or copy the extract into a new one, and save the result through the **existing** endpoint that uploads a KVK extract XML to the Azure SQL table,
3. (v2) de-identify personal data in extracts that come from production, before anything can be edited or saved.

Rules for the whole session:

- Quarkus REST, Java, the project's existing structure and conventions. No new libraries; use the JDK's `javax.xml.xpath`, DOM and `javax.crypto`. MapStruct is not used for this feature's own code.
- Everything is behind the project's existing string-based feature flag, placeholder name `kvkExtractSandbox` (replace with the real one). Every endpoint returns 404 when it is off. Non-prod environments only.
- Stateless: no HTTP session, no temp files, no DB access of its own. Saving goes through the existing upload endpoint only.
- Parse XML with DTDs and external entities disabled (XXE-safe). Set secure-processing on all factories.
- **Never log XML content or any extracted value.** Extracts can hold real company data.
- Keep REST classes thin; put logic in plain classes so it is unit-testable.

Work in the order of sections 1 to 6. Each section ends with what must pass before moving on.

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

## 1. Build the parameter registry (one-off, then reviewed by a human)

I will **not** hand you a list of xpaths. Instead I give you:

- the path(s) of the **MapStruct mapper files** that map the KVK extract into MPMA's model, and
- a **Word document** describing the KVK fields (convert it first with `pandoc kvk-fields.docx -t markdown -o kvk-fields.md` and read the markdown, not the .docx).

From these, produce `src/main/resources/kvk-parameters.json` in the schema below, plus `kvk-parameters-review.md` listing what a human must check. Procedure:

1. **Find the parameters.** Each distinct KVK field the mappers read (company name, address parts, foundation date, SBI activity, number of employees, bestuurders and their details, and so on) becomes one parameter with a stable camelCase `key`. Use the Word document for `label`, `hint`, `mandatory` and `type`; if it does not say, use `mandatory: false` and `type: "text"` and list the parameter in the review file.
2. **Find the priority order.** Where a mapper takes one target value from several sources (fallbacks such as `expression`, `qualifiedByName`, `defaultExpression`, or custom `@Named` methods that return the first non-empty value), list every source in the order the mapper tries them. That order is the `xpaths` priority order. Parameters with one source have one xpath.
3. **Turn mapper source paths into xpaths.** MapStruct source paths are Java property paths on the extract's model classes. Resolve each one to an XML path using the class annotations or schema that bind the XML to those classes (JAXB `@XmlRootElement`, `@XmlElement(name=...)`, `@XmlAttribute`, `@XmlType`, `package-info` namespaces; or the XSD if there is one). Use absolute xpaths from the document root. For list properties use an index (`bestuurder[1]`) for the first item only; repeated items beyond the first are out of scope. If the XML uses namespaces, fill the `namespaces` map and use its prefixes in the xpaths.
4. **Unresolvable paths.** If a mapper path cannot be resolved with confidence (custom converters, nested expressions), do not guess: put it in the review file with the mapper file, line, and what is missing. Leave that xpath out of the JSON.
5. **Classify personal data.** Propose `pii` and `group` from the field meaning and mark each proposal in the review file as "proposed, confirm":
   - `personName`: names of natural persons (bestuurders and similar), with a `group` per person shared with that person's `dob`, address and BSN/TIN parameters;
   - `dob`: dates of birth;
   - `address`: addresses of persons (one `group` per address, covering street, number, postcode, city if they are separate parameters);
   - `companyName`, `companyAddress`: company names and addresses;
   - `idNumber`: BSN and TIN values. **KVK number and RSIN get no `pii` value** and are never replaced.
   - On a `personName` parameter, set `genderXpath` if a gender element exists for that person.
6. **Validate the result** with a test that loads the JSON, checks every xpath compiles, and checks keys are unique.

Registry schema (this is what `GET /kvk-editor/parameters` returns, as `Parameter[]` from the contract, plus top-level `namespaces`):

```json
{
  "namespaces": { "k": "urn:example:kvk" },
  "parameters": [
    {
      "key": "tradeName",
      "label": "Trade name",
      "mandatory": true,
      "type": "text",
      "pii": "companyName",
      "xpaths": ["/k:kvkExtract/k:handelsnaam/k:volledigeNaam", "/k:kvkExtract/k:statutaireNaam"]
    },
    {
      "key": "director1Name",
      "label": "Director 1 name",
      "mandatory": false,
      "type": "text",
      "pii": "personName",
      "group": "director1",
      "genderXpath": "/k:kvkExtract/k:bestuurder[1]/k:geslacht",
      "xpaths": ["/k:kvkExtract/k:bestuurder[1]/k:volledigeNaam"]
    }
  ]
}
```

(The sample paths above are illustrative; real ones come from the mappers.)

`GET /kvk-editor/parameters` returns the `parameters` array; `namespaces` and `genderXpath` stay server-side.

**Done when:** the JSON loads, the validation test passes, and `kvk-parameters-review.md` exists. **Stop here and let me review the file before building on it**, because every later step depends on it.

## 2. XML service

A plain class with no REST code:

- parse text to a DOM document (namespace-aware, XXE-safe) using the registry's namespaces;
- read the value at an xpath (empty string if the node is missing; report `exists` separately);
- `SET`: replace the text of the node; if it is missing, create the elements along the path and then set the text. Support plain child-element paths and predicates of the form `[@attr="value"]` (create the element and attribute). If a path is too complex to create, return a clear error naming the parameter and xpath;
- `REMOVE`: delete that node only; the parameter's other xpaths stay as they are;
- serialise back to a string, keeping the original declaration and encoding.

**Done when:** unit tests with a sample extract cover read, SET existing, SET missing, REMOVE, a priority parameter whose first xpath is empty, namespaces, and malformed XML.

## 3. `parse` and `apply` with validation

- `POST /kvk-editor/parse`: for each parameter, per xpath, return `{xpath, exists, value}`.
- `POST /kvk-editor/apply`: parse `xml`, handle COPY (write `newKvkNumber`, and `newName` if given, to the xpaths of the KVK number and name parameters), replay `edits` in order, validate, return the new XML and all errors.

Validation, run on every `apply`:

| Rule | Check |
| --- | --- |
| Well-formed | The XML parses; otherwise `400` |
| Mandatory | A parameter with `mandatory: true` needs a non-empty value on at least one of its xpaths |
| Type | `date` is `yyyy-MM-dd`; `number` is an integer; `text` has no check |
| Copy mode | `newKvkNumber` required, 8 digits, different from the source's KVK number |

Errors are `{key, xpath?, message}`; do not stop at the first one.

**Done when:** tests cover each rule, COPY mode, and an edits list that sets, removes and re-sets the same path.

## 4. `save`

`POST /kvk-editor/save` takes `SaveRequest`: runs the `apply` logic; if there are errors, returns `422 { errors }` and does not call the upload endpoint; otherwise calls the **existing** upload endpoint with the final XML (reuse the existing client or call it over REST with the pipeline's usual auth) and returns `{ uploadResult }` with whatever that endpoint returns. Production rule (v2): if `sourceType` is `PRODUCTION` and `deidentified` is not true, return `422` with a clear message. This guards against mistakes only; the stateless backend cannot prove the XML was changed.

I will tell you the request and auth shape of the existing upload endpoint when you reach this step; until then, call it through an interface so it can be mocked.

**Done when:** tests cover a refused save, a successful TEST save, a refused PRODUCTION save without the flag, and a PRODUCTION save with it (upload call mocked).

## 5. De-identification engine (v2)

A plain class: input XML text, output de-identified XML text plus a `DeidReport`. No REST code.

Decisions:

- **Scope:** names of natural persons, dates of birth, person addresses, company names, company addresses, and BSN and TIN values, **anywhere in the file**. KVK number and RSIN are kept.
- **Consistency:** the same real value gets the same fake everywhere in one file. A fresh random key is created for every run and discarded afterwards, so nothing links two files and a second run gives different fakes.

Steps:

1. **Collect.** For every parameter with a `pii` value, read all its xpaths. Normalise each value (trim, lowercase, strip accents, collapse whitespace). Group by `group`: each distinct person or address becomes one entity; a parameter without a group is its own entity.
2. **Generate.** Take HMAC-SHA256 of the run key, the category and the normalised value, and use it to pick an entry from the fake lists in `src/main/resources` (first names by gender and neutral, surnames with tussenvoegsels, company names, streets, and postcode with matching city). If two different originals pick the same fake, probe to the next entry so fakes stay unique. You write the fake lists: a few hundred plausible Dutch entries each, clearly fictional, no real person's full name.
3. **Replace listed xpaths** in every xpath of the parameter, including lower-priority ones:

| `pii` | Replacement |
| --- | --- |
| `personName` | Fake Dutch name keeping initials and tussenvoegsel format; first names from the matching list if `genderXpath` gives a gender, else the neutral list. The gender value itself is not changed |
| `dob` | The original date shifted by the person's offset, same format |
| `address`, `companyAddress` | One coherent fake address (street, number, postcode, city together; postcode and city match) |
| `companyName` | Fake name keeping the legal form, e.g. B.V. |
| `idNumber` | BSN: fake 9-digit number that passes the 11-test. TIN: same length and character pattern (digits stay digits, letters stay letters) |

4. **Date offsets.** Each person gets one offset between 61 and 365 days, never zero, derived from the same HMAC. Clamp if the shifted date would make the person under 18 or fall after the company's foundation date.
5. **Sweep unlisted places.** Build a table of every original value and its variants: full name, surname alone (only if 4 or more characters), initials plus surname, street name, postcode, each BSN or TIN value. Walk all text nodes and attribute values, replace matches case-insensitively on word boundaries, longest match first, using the same fake as the listed replacement. Record each place changed (path only).
6. **Final check.** Search the result for every original value and variant; survivors go into `warnings`. Also list any 9-digit number that looks like a BSN but was **not** replaced (it may be an RSIN) as a warning; do not replace it.
7. **Report.** `counts` per `pii` category, `pairs` for the review screen, `unlisted`, `warnings`.

**Done when** unit tests (sample extract with repeated names and addresses) prove:

- the same name in three places gets the same fake in all three;
- no original value or variant, BSN and TIN included, remains in the output;
- the output is well-formed and passes the section 3 validation;
- postcode and city of a fake address match;
- shifted dates are valid, keep their format, nobody becomes under 18;
- KVK number and RSIN are unchanged;
- two runs on the same file give different results.

## 6. `deidentify` endpoint and audit

- `POST /kvk-editor/deidentify`: runs the engine, returns `{ xml, report }`.
- Audit log per run: user, time, source file name if sent, counts per category, number of warnings. **Never values.**
- The original production XML is never written to a database, file or log; only the de-identified result can be saved.

**Done when:** an endpoint test with a sample passes, and a test proves the audit entry contains no extract values.

## 7. Open items to confirm with me

- The real feature flag name and how flags are read in this project.
- The request and auth shape of the existing upload endpoint.
- The mapper file paths and the Word document (converted to markdown).
- A first real or de-identified extract to test the sweep against.
- Whether the data protection contact accepts pseudonymization for the production-data approval (not a coding task).
