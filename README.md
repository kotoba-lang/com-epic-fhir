# Epic_fhir Clean Room Actor

Clean-room API-compatible implementation of the epic_fhir deep system protocol, backed by Datomic and Py Kotodama WASM.

## Provenance

Relocated 2026-07-04 from `etzhayyim/root/20-actors/epic_fhir-compat` to
`kotoba-lang/com-epic-fhir` per the org-taxonomy library-placement rule (any
library/substrate code belongs in `kotoba-lang`, ADR-2606302300), following
the same relocation pattern as `kami-nv-compat` (ADR-2607020130). See
ADR-2607041500 for the full ~1,027-repo migration plan and naming convention.

## Maturity note (2026-07-08)

As relocated, this repo was a `deepen_actors.py`-generated generic CRUD
scaffold: FHIR resource *names* (Patient/Observation/Encounter/
MedicationRequest/AllergyIntolerance/Condition) with no HL7/FHIR-specific
validation -- `:required`/`:coerce` only check presence and JSON-ish type
coercion, not domain format. `kotoba-lang/com-hl7-fhir` (an identical
sibling scaffold) had already closed this gap for itself
(ADR-2607083000 US-Claim / ADR-2607083100 EU-Consent passes) but explicitly
left `com-epic-fhir` and `com-eclinicalworks` un-validated since they're
separate git repos and the fix doesn't propagate. This pass ports that same
increment here, by value (no cross-repo dependency; each actor stays a
standalone deploy unit):

- `Claim` entity modeling the CMS-1500 / UB-04 / X12 837 professional-claim
  minimum field set (`billingProviderNpi`/`subscriberId`/`diagnosisCode`/
  `procedureCode`), with `:validate` entity-spec checks for:
  - `billingProviderNpi` -- NPI check digit (ISO/IEC 7812-1 Luhn over the
    `"80840"`-prefixed 9-digit identifier, 45 CFR 162.410).
  - `diagnosisCode` -- ICD-10-CM structural shape.
  - `procedureCode` -- CPT Category I/II/III or HCPCS Level II structural
    shape.
- `Consent` entity recording, per patient, which of the ten exceptions in
  **Art. 9(2)(a)-(j) of Regulation (EU) 2016/679 (GDPR)** is relied on to
  lift the Art. 9(1) prohibition on processing "data concerning health"
  (`specialCategoryData` boolean flag + `lawfulBasisArt9` code, validated
  against the fixed ten-point-letter set).

See `src/epic_fhir/validation.cljc` (pure validators, ported by value from
`hl7-fhir.validation`, with their own docstring caveats about what "format
valid" does and doesn't guarantee) and `test/epic_fhir/validation_test.cljc`
/ the `claim-domain-validation` and `consent-domain-validation` deftests in
`test/epic_fhir/main_test.cljc` for pass/fail coverage. `bb test` runs both
files. `com-eclinicalworks` still needs the same follow-up (or may already
have received it in a separate pass -- check its own README/git log). The
`manifest.json` capability declaration was intentionally left unchanged for
the same reason as `com-hl7-fhir`'s: it's paired with a specific built
`wasmCid` and hand-editing its `entities`/`routes`/`mcp.tools` lists without
rebuilding that WASM artifact would make the manifest *less* accurate, not
more.

## Maturity note (2026-07-08) -- EU: EHDS Article 3 (Regulation (EU) 2025/327)

Follow-up porting `kotoba-lang/com-hl7-fhir`'s EHDS Article 3 increment
(ADR-2607083200) to this sibling actor, by value, closing the gap the
Claim/Consent pass above left open (that pass ported Claim/Consent but
predated the hl7-fhir `PatientAccessRequest` entity).

Adds a `PatientAccessRequest` entity modeling **Article 3 ("Right of natural
persons to access their personal electronic health data"), paragraphs
(1)-(3) only** -- the only EHDS text with a verified primary-source reading
to hand. The verbatim Article 3(1)-(3) text was retrieved by
`kotoba-lang/com-hl7-fhir` via a real-browser EUR-Lex session on 2026-07-08
(EUR-Lex blocks automated `curl`/headless fetches with an AWS WAF JS
challenge) and is archived, with full retrieval-method provenance, at
[`kotoba-lang/emr-claims-primary-sources`](https://github.com/kotoba-lang/emr-claims-primary-sources)'s
`eu-ehds/ehds-article3-excerpt.md` (CELEX:32025R0327):

- `priorityCategory` (boolean) -- whether the underlying record belongs to
  the "priority categories" Article 3(1)/(2) refer to. **This is a bare
  flag, not an enumerated category list**: the priority-categories list
  itself is defined in **Article 14**, which has not yet been retrieved from
  a primary source -- inventing that list here would be exactly the kind of
  unverified-legal-content guess this codebase's working agreement forbids.
- `accessMethod` (enum `"view"` / `"download"`, case-insensitive, required)
  -- `"view"` is Art. 3(1) (immediate, free, easily-readable/consolidated
  access once data is registered in an EHR system); `"download"` is
  Art. 3(2) (a free electronic copy in the European electronic health
  record exchange format). Validated by
  `epic_fhir.validation/valid-ehds-access-method?`; anything else is
  rejected with 400.
- `restrictionApplied` (boolean) + `restrictionReason` (optional string) --
  Art. 3(3): a Member State may restrict/delay both rights "in accordance
  with Article 23" of GDPR (Regulation (EU) 2016/679), e.g. for
  patient-safety/ethical reasons. Enforced by a new **cross-field**
  validator, `epic_fhir.validation/valid-ehds-restriction?`, wired through a
  new entity-spec key `:validate-record` (complementing the existing
  single-field `:validate`) and a new `validate-record` fold in
  `src/epic_fhir/main.cljc`'s `handle-create`/`handle-update`:
  `restrictionApplied=true` with a blank/absent `restrictionReason` is
  rejected with 400 on both create and update (update checks the *merged*
  record, so patching only `restrictionApplied` against an existing
  reason-less record is still caught).

**Explicitly out of scope / not implemented, and why**: same as
`com-hl7-fhir`'s own EHDS pass -- Article 14 (the priority-categories list
itself), Article 4 (the "electronic health data access services"
definition) and Article 15 (the European electronic health record exchange
format's technical schema) were confirmed to exist but their text has
**not** been retrieved yet, so none of the three is modeled.
`com-athenahealth` was not touched (vendor-specific field names, out of
scope per this cohort's working agreement).

See `src/epic_fhir/validation.cljc` (`valid-ehds-access-method?` /
`valid-ehds-restriction?`, with the scope caveats inline) and
`test/epic_fhir/validation_test.cljc`'s `ehds-access-method-format` /
`ehds-restriction-cross-field` deftests / `test/epic_fhir/main_test.cljc`'s
`patient-access-request-domain-validation` deftest for pass/fail coverage
(both access methods and case-insensitivity accepted, an out-of-set method
rejected, a restriction without a reason rejected on both create and merged
update, a restriction with a reason accepted). `bb test`: 14 deftests / 254
assertions as of this pass (up from 11/201).
