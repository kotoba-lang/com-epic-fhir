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
