# Story: Pulp npm package provenance (DSSE)

**Audience:** Pulp / pulp_npm maintainers  
**Requesting consumers:** Calunga / Lightwell npm release pipelines (Konflux)  
**Analogy:** [pulp_python attestation hosting (PEP 740)](https://pulpproject.org/pulp_python/docs/user/guides/attestation/)  
**Status:** Draft for implementation planning

---

## Summary

Add first-class **package provenance** support to `pulp_npm`, parallel to Python’s `PackageProvenance` / Integrity API path, but storing **cosign DSSE** envelopes (SLSA provenance over the tarball) instead of PEP 740 JSON.

Today npm packages upload via `content/npm/packages/` (and optional labels). There is **no** npm provenance content type. Calunga/Lightwell already generate (or will generate) `<tarball>.att` DSSE files at release time; Pulp cannot host them next to the package the way it hosts Python `.attestation` files.

---

## Problem

| Ecosystem | Package upload | Provenance today |
|-----------|----------------|------------------|
| Python (`pulp_python`) | packages + twine | `content/python/provenance/`, Simple + Integrity API, PEP 740 |
| npm (`pulp_npm`) | `content/npm/packages/upload/` | **None** (compliance is labels / optional file-repo sidecars / Quay OCI — not package provenance) |

Without npm provenance content:

- Release pipelines cannot attach signed SLSA attestations to published `.tgz` units.
- Consumers cannot fetch package-bound provenance from the registry the way PyPI/Pulp Integrity does for wheels.
- Workarounds (Quay-only, file repo) do not give repository sync/add/remove semantics tied to the npm content unit.

---

## Goals (MVP)

1. **Content type** for npm package provenance (name TBD; e.g. `NpmPackageProvenance`), unique by digest of the stored envelope (same immutability model as Python provenance).
2. **Associate** provenance with an existing npm package content unit (by package PRN/href and/or tarball **sha256**).
3. **Upload APIs:**
   - Create provenance for an **already-published** package (backfill / remediation; mirrors Python `content/python/provenance/` create).
   - Optionally attach provenance **at package upload** time (mirrors Python `--attestation` / twine `--attestations`).
4. **Repository semantics:** provenance is repository content — add/remove/sync with the package; **one provenance per package per repository version** (replace by uploading a new immutable object), matching Python.
5. **Retrieve** provenance for a given package (REST is required for MVP; registry/metadata exposure can be a follow-up — see Open questions).
6. **Format accepted in MVP:** raw **cosign DSSE** JSON envelope (file extension `.att` in producer tooling), with:
   - in-toto statement payload
   - `predicateType` / cosign type `https://slsa.dev/provenance/v1`
   - subject bound to the **npm tarball** (digest of the package blob)

Producer reference (current Calunga helper — thin predicate today; Pulp must require DSSE + in-toto + `predicateType` SLSA v1, but must not hard-code `buildType` / builder IDs or other predicate body fields):

- https://github.com/calungaproject/plumbing/blob/main/utils/scripts/generate-and-sign-npm-attestations

---

## Non-goals (MVP)

- **PEP 740** wrappers or Python Integrity API compatibility for npm.
- Full **npmjs.com / `npm audit signatures`** parity (Sigstore Bundle + Fulcio OIDC + Rekor + sha512 PURL subjects). DSSE is the shared envelope; packaging may differ in v1.
- Validating signature cryptographically inside Pulp (store + associate + serve; verification is client-side with cosign/sigstore/slsa-verifier).
- Requiring rich Chains-enriched predicates (empty/`buildDefinition` placeholders are OK; richer predicates must still upload without API changes).
- Compliance metadata (`tl.compliance_level`, closure index OCI) — already handled elsewhere.

---

## Proposed behavior (sketch for design)

Mirror [pulp_python attestation hosting](https://pulpproject.org/pulp_python/docs/user/guides/attestation/) where possible:

```text
POST .../content/npm/provenance/
  - file: <dsse>.att  (or JSON body)
  - package: <npm package pulp_href | prn>
  - repository: <npm repo href>   # optional if using modify flow
  # and/or package sha256

POST .../content/npm/packages/upload/  (enhancement)
  - existing package fields
  - attestation(s): one or more DSSE files   # optional in MVP if backfill-only ships first
```

**Attach-at-upload atomicity:** If provenance is accepted on the package upload path, treat package + provenance as one transaction: if DSSE validation fails (parse, `payloadType`, `predicateType`, or subject digest), **fail the whole upload** and do not leave a package content unit in the repository without provenance. Prefer validate-then-create, or roll back package association on provenance failure. Backfill-only uploads (package already present) only fail the provenance create.

Retrieve (minimum):

```text
GET .../content/npm/provenance/?package=<href>&repository_version=<rv>
# or GET by provenance pulp_href
```

**Uniqueness / immutability:** Content units are immutable. Re-upload must **create a new** provenance content unit and update the repository version so the package’s association points at the new unit (remove/replace the old association in that repo version)—never mutate or overwrite the previous provenance object in place. At most one provenance is associated with a given npm package content unit **in a given repository version**.

**Validation (MVP — fail closed on wrong type or subject):**

- Parseable DSSE envelope (`payload`, `payloadType`, `signatures`).
- **Required:** `payloadType` compatible with in-toto (e.g. `application/vnd.in-toto+json` or cosign’s attested in-toto type).
- **Required:** decode the in-toto statement and reject unless `predicateType` is exactly `https://slsa.dev/provenance/v1` (do not store unrelated statements — e.g. npm publish attestations or other predicate types — as npm provenance).
- **Required:** reject if no subject digest matches the selected package tarball’s digest (sha256 for MVP; document sha512 as a later enhancement). Do not associate a mismatched envelope.
- Still **out of scope:** cryptographic signature verification inside Pulp (clients verify with cosign/sigstore).

Do **not** require specific `buildType` / builder IDs or other predicate body fields — those are producer policy.

---

## Acceptance criteria

1. Given an npm package already in a repository, an authenticated client can upload a DSSE `.att` whose subject digest matches that package’s tarball and see a new provenance content unit linked to that package.
2. Listing/filtering provenance by `package` (and repository version) returns that object; downloading returns the stored DSSE bytes/JSON.
3. Uploading a second provenance for the same package in the same repo creates a **new** immutable content unit and updates the repo association (does not mutate the old unit).
4. Upload is **rejected** (no association) when the decoded statement’s subject digest does not match the selected package tarball digest.
5. Upload is **rejected** when the decoded statement’s `predicateType` is not `https://slsa.dev/provenance/v1`.
6. When attach-at-upload is implemented: invalid provenance **fails the entire package upload** (no package left in the repo without provenance).
7. Provenance participates in repository add/remove/sync like other content (document remote flag if sync-from-upstream is in scope, same as Python `provenance` on remote).
8. Documented API + example `curl`/`pulp-cli` flow for Calunga/Lightwell integration; functional tests/fixtures for `.att` uploads (e.g. pulp-smash or pulp_npm suite).
9. Unit/functional tests covering create, associate, replace-via-new-unit, retrieve, **subject-digest mismatch rejection**, **non-SLSA predicateType rejection**, and **atomic attach-at-upload failure** (if that path ships).
10. Explicit docs note: format is **DSSE / SLSA v1**, not PEP 740; not claiming npm CLI audit-signatures compatibility in MVP.

---

## Consumer integration (out of scope for Pulp, for context)

After Pulp lands MVP:

1. Wire [`generate-and-sign-npm-attestations`](https://github.com/calungaproject/plumbing/blob/main/utils/scripts/generate-and-sign-npm-attestations) into [`calunga-push-npm-to-pulp`](https://github.com/konflux-ci/release-service-catalog/blob/development/pipelines/managed/calunga-push-npm-to-pulp/calunga-push-npm-to-pulp.yaml) (future managed task patterned on [`rh-sign-python-wheels`](https://github.com/konflux-ci/release-service-catalog/blob/development/tasks/managed/rh-sign-python-wheels/rh-sign-python-wheels.yaml); no `rh-sign-npm-packages` task exists yet).
2. Extend [`npm-pulp-upload`](https://github.com/calungaproject/plumbing/blob/main/utils/scripts/npm-pulp-upload) / [`npm-release-upload`](https://github.com/calungaproject/plumbing/blob/main/utils/scripts/npm-release-upload) (invoked from [`upload-npm-pulp`](https://github.com/konflux-ci/release-service-catalog/blob/development/tasks/managed/upload-npm-pulp/upload-npm-pulp.yaml)) to POST provenance (with package or at upload time).
3. Optional later: enrich predicate from Tekton Chains ([`npm-fetch-chains-provenance`](https://github.com/calungaproject/plumbing/blob/main/utils/scripts/npm-fetch-chains-provenance)) without changing Pulp API.

---

## Open questions (design spike OK)

| # | Question | Notes / preference |
|---|----------|--------------------|
| 1 | Attach-at-upload in MVP, or backfill-only first? | Backfill-only unblocks remediation; attach-at-upload matches Python UX — prefer both if cheap. If attach-at-upload ships: **atomic** (invalid DSSE fails the whole package upload). |
| 2 | Serve via npm registry metadata vs REST-only? | REST-only is enough for Konflux; registry field helps generic clients. |
| 3 | Store raw DSSE vs Sigstore Bundle? | **Raw DSSE for MVP** (matches current producer). Bundle can be a later content version or alternate media type. |
| 4 | Subject digest algorithm: sha256 vs sha512? | Cosign `attest-blob` uses sha256 today; npmjs uses sha512 on PURLs. Prefer accept sha256 for MVP; document sha512 as enhancement. |
| 5 | Endpoint naming | Prefer symmetry: `content/npm/provenance/` ↔ `content/python/provenance/`. |
| 6 | pulp CLI / content guards | Empty/malformed DSSE, non-SLSA `predicateType`, and subject-digest mismatch fail closed on create. |

---

## References

- Python guide: https://pulpproject.org/pulp_python/docs/user/guides/attestation/
- DSSE: https://github.com/secure-systems-lab/dsse
- in-toto envelope (DSSE recommended): https://github.com/in-toto/attestation/blob/main/spec/v1/envelope.md
- SLSA provenance v1: https://slsa.dev/spec/v1.0/provenance
- Producer stub: https://github.com/calungaproject/plumbing/blob/main/utils/scripts/generate-and-sign-npm-attestations
- npm public provenance (future parity, not MVP): https://github.com/npm/provenance — Sigstore Bundle wrapping DSSE
- Related follow-ups: `docs/prod_followup.md` (package signing / attestations at release)

---

## Suggested Jira title / description blurb

**Title:** pulp_npm: host package provenance as cosign DSSE (SLSA v1)

**Description:** Implement npm package provenance content analogous to pulp_python PEP 740 provenance, storing cosign DSSE envelopes linked to npm package content by tarball digest. MVP: upload (backfill + preferably with package), one provenance per package per repo, retrieve via content API. Non-goal: npm CLI Sigstore Bundle / audit-signatures parity.
