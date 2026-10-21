# Story: Pulp npm package attestations (DSSE / in-toto)

**Audience:** Pulp / pulp_npm maintainers  
**Requesting consumers:** Calunga / Lightwell npm release pipelines (Konflux)  
**Analogy:** [pulp_python attestation hosting (PEP 740)](https://pulpproject.org/pulp_python/docs/user/guides/attestation/) — same *role* (package-bound signed metadata), different envelope (DSSE, not PEP 740)  
**Status:** Draft for implementation planning

---

## Summary

Add first-class **package attestation** support to `pulp_npm`: store **cosign DSSE** envelopes that wrap **in-toto** statements, associated with an npm package content unit (tarball).

MVP is **not** limited to SLSA provenance. Pulp should accept an **allowlist** of in-toto `predicateType` values so the same storage path can hold:

- SLSA provenance — `https://slsa.dev/provenance/v1`
- Verification Summary Attestation (VSA) — see [in-toto VSA predicate](https://github.com/in-toto/attestation/blob/main/spec/predicates/vsa.md)
- Additional predicate types later via config/allowlist expansion (no new content type per predicate)

Calunga’s first producer will upload SLSA provenance (`.att`); other predicates (e.g. VSA) must upload through the same API without a redesign.

Today npm packages upload via `content/npm/packages/` (and optional labels). There is **no** npm attestation content type.

---

## Problem

| Ecosystem | Package upload | Attestations today |
|-----------|----------------|--------------------|
| Python (`pulp_python`) | packages + twine | `content/python/provenance/`, Simple + Integrity API, PEP 740 (provenance-shaped) |
| npm (`pulp_npm`) | `content/npm/packages/upload/` | **None** (compliance is labels / optional file-repo sidecars / Quay OCI) |

Without package-bound attestation content:

- Release pipelines cannot attach signed in-toto attestations (SLSA, VSA, …) to published `.tgz` units.
- Consumers cannot fetch package-bound attestations from Pulp with repo add/remove/sync semantics.
- Narrowing the API to “SLSA only” would force a second content type later for VSA and peers.

---

## Goals (MVP)

1. **Content type** for npm package attestations (prefer name like `NpmPackageAttestation`; avoid baking “provenance-only” into the type name). Content units are unique by digest of the stored envelope (immutable content).
2. **Associate** each attestation with an existing npm package content unit (by package PRN/href and/or tarball **sha256**). The package must already be present in the **target repository version** when associating (no orphan attestations in a version that lacks the package).
3. **Upload APIs:**
   - Create attestation(s) for an **already-published** package (backfill / remediation).
   - Optionally attach attestation(s) **at package upload** time.
4. **Repository semantics:** attestations are repository content — add/remove/sync with the package (removing the package from a repo version removes its attestation associations in that version).
5. **Uniqueness:** at most **one attestation per `(package, predicateType)` per repository version**, enforced atomically on create/replace. **Different** `predicateType`s (e.g. SLSA + VSA) **coexist**. Same envelope digest re-upload is **idempotent** (reuse the existing content unit / association); a **different** envelope for the same `predicateType` creates a new unit and rewires the association.
6. **Retrieve** attestations for a package (REST required for MVP), filterable by `predicateType`.
7. **Format:** raw **cosign DSSE** JSON (producer often uses `.att`), with:
   - in-toto statement payload
   - `predicateType` on an **allowlist** (see Validation)
   - subject bound to the **npm tarball** (digest of the package blob)

Producer reference (SLSA path today — thin predicate; Pulp must not hard-code `buildType` / builder IDs or other predicate **body** fields):

- https://github.com/calungaproject/plumbing/blob/main/utils/scripts/generate-and-sign-npm-attestations

---

## Non-goals (MVP)

- **PEP 740** wrappers or Python Integrity API compatibility for npm.
- Full **npmjs.com / `npm audit signatures`** parity (Sigstore Bundle + Fulcio OIDC + Rekor + sha512 PURL subjects).
- Cryptographic signature verification inside Pulp (store + associate + serve; clients verify with cosign/sigstore/slsa-verifier).
- Open-ended “any DSSE” dumping ground — use an **allowlist**, not “parseable only”.
- Requiring rich Chains-enriched SLSA predicates (placeholders OK; richer predicates must still upload).
- Compliance metadata (`tl.compliance_level`, closure index OCI) — handled elsewhere.
- Implementing VSA *producers* in Calunga (Pulp only needs to **accept/store** VSA when uploaded).

---

## Proposed behavior (sketch for design)

Python used `provenance` for historical/PEP 740 reasons. For npm prefer an **attestations** content API (name TBD with maintainers):

```text
POST .../content/npm/attestations/
  - file: <dsse>.att  (or JSON body)
  - package: <npm package pulp_href | prn>
  - repository: <npm repo href>   # optional if using modify flow
  # and/or package sha256

POST .../content/npm/packages/upload/  (enhancement)
  - existing package fields
  - attestation(s): one or more DSSE files   # optional in MVP if backfill-only ships first
```

**Attach-at-upload atomicity:** If attestations are accepted on the package upload path, treat package + all supplied attestations as one transaction: on **any** failure (validation **or** persistence after validation), **fail the whole upload**, roll back, and do not leave the package in the repository without the supplied attestations. Validate-then-create alone is insufficient unless writes are also transactional. Backfill-only creates only fail the attestation create.

Retrieve (minimum):

```text
GET .../content/npm/attestations/?package=<href>&repository_version=<rv>
GET .../content/npm/attestations/?package=<href>&predicate_type=<uri>&repository_version=<rv>
# or GET by attestation pulp_href
```

**Uniqueness / immutability:** Content units are immutable (never mutate an existing unit in place). For a given `(package, predicateType)` in a repository version:

- **Identical envelope** (same content digest already stored and associated): **idempotent** success — do not create a duplicate unit.
- **Different envelope** (new digest, same `predicateType`): create a **new** unit and rewire the association (remove/replace the prior association in that repo version).
- **Different `predicateType`:** additional association; both remain.

Create/replace of the `(package, predicateType)` association must be **atomic** so concurrent uploads cannot leave two associations for the same pair in one repository version.

**Validation (MVP — fail closed):**

- Parseable DSSE envelope (`payload`, `payloadType`, `signatures`).
- **Required:** `signatures` is a non-empty array (structural guard only — presence of at least one signature object, not cryptographic validity).
- **Required:** `payloadType` compatible with in-toto (e.g. `application/vnd.in-toto+json` or cosign’s attested in-toto type).
- **Required:** decode the in-toto statement; `predicateType` must be on the **MVP allowlist**:
  - `https://slsa.dev/provenance/v1`
  - VSA predicate type URI as defined by [in-toto VSA](https://github.com/in-toto/attestation/blob/main/spec/predicates/vsa.md) (pin the exact string in implementation docs when implementing)
  - Allowlist must be **extensible** (setting / constant) so new predicate types do not need a new content type
- **Required:** reject if no subject digest matches the selected package tarball’s digest (sha256 for MVP; document sha512 as a later enhancement).
- **Required:** package is present in the target repository version (when associating into a repo).
- Still **out of scope:** cryptographic signature verification / trust-policy checks inside Pulp. Docs and client guidance must state that **hosting ≠ verification**; consumers must verify with cosign/sigstore/slsa-verifier (or equivalent).

Do **not** require specific SLSA `buildType` / builder IDs or deep validation of VSA predicate body fields beyond schema/allowlist membership — producer policy.

Authorization follows normal Pulp RBAC on repositories/content (write to associate; read visibility on retrieve). No special cross-tenant model beyond that.

---

## Acceptance criteria

1. Given an npm package in a repository, an authenticated client can upload a DSSE `.att` with an allowlisted `predicateType` and matching tarball subject digest and see a new attestation content unit linked to that package.
2. The same package can hold **both** an SLSA provenance attestation and a VSA attestation in one repository version.
3. Listing/filtering by `package` (and optionally `predicate_type`) returns the expected object(s); download returns stored DSSE bytes/JSON.
4. Re-upload of a **different** envelope for the **same** `predicateType` creates a **new** immutable unit and updates the repo association; re-upload of the **identical** envelope is **idempotent**.
5. Upload is **rejected** when subject digest does not match the package tarball, when `signatures` is empty, when the package is not in the target repository version, or when `predicateType` is not on the allowlist.
6. When attach-at-upload is implemented: any failure (validation or write) **fails the entire package upload** with rollback (no package left without the supplied attestations).
7. Attestations participate in repository add/remove/sync like other content; removing the package removes attestation associations in that version.
8. Documented API + `curl`/`pulp-cli` examples; functional tests/fixtures for `.att` (SLSA and at least one non-SLSA allowlisted type, e.g. VSA), including idempotent re-upload and empty-`signatures` rejection.
9. Docs note: format is **DSSE / in-toto**, not PEP 740; hosting does not imply signature verification; not claiming npm CLI audit-signatures compatibility in MVP.

---

## Consumer integration (out of scope for Pulp, for context)

After Pulp lands MVP:

1. Wire [`generate-and-sign-npm-attestations`](https://github.com/calungaproject/plumbing/blob/main/utils/scripts/generate-and-sign-npm-attestations) into [`calunga-push-npm-to-pulp`](https://github.com/konflux-ci/release-service-catalog/blob/development/pipelines/managed/calunga-push-npm-to-pulp/calunga-push-npm-to-pulp.yaml) (future managed task patterned on [`rh-sign-python-wheels`](https://github.com/konflux-ci/release-service-catalog/blob/development/tasks/managed/rh-sign-python-wheels/rh-sign-python-wheels.yaml); no `rh-sign-npm-packages` task exists yet).
2. Extend [`npm-pulp-upload`](https://github.com/calungaproject/plumbing/blob/main/utils/scripts/npm-pulp-upload) / [`npm-release-upload`](https://github.com/calungaproject/plumbing/blob/main/utils/scripts/npm-release-upload) (via [`upload-npm-pulp`](https://github.com/konflux-ci/release-service-catalog/blob/development/tasks/managed/upload-npm-pulp/upload-npm-pulp.yaml)) to POST attestations.
3. Optional later: enrich SLSA predicates from Tekton Chains ([`npm-fetch-chains-provenance`](https://github.com/calungaproject/plumbing/blob/main/utils/scripts/npm-fetch-chains-provenance)); upload VSA or other predicates from separate producers — **same Pulp API**.

---

## Open questions (design spike OK)

| # | Question | Notes / preference |
|---|----------|--------------------|
| 1 | Attach-at-upload in MVP, or backfill-only first? | Prefer both if cheap; attach-at-upload must be **atomic**. |
| 2 | Serve via npm registry metadata vs REST-only? | REST-only enough for Konflux; registry field helps generic clients. |
| 3 | Store raw DSSE vs Sigstore Bundle? | **Raw DSSE for MVP**. |
| 4 | Subject digest: sha256 vs sha512? | sha256 for MVP (cosign); sha512 enhancement. |
| 5 | Endpoint naming | Prefer `content/npm/attestations/` (broader than `provenance/`). Symmetry with Python is secondary to accurate naming. |
| 6 | Allowlist mechanism | Constant vs settings; process to add predicates (e.g. SCITT, npm publish) without migrations. |
| 7 | Exact VSA `predicateType` URI | Pin to current in-toto VSA spec string at implementation time. |
| 8 | pulp CLI / content guards | Malformed DSSE, non-allowlisted `predicateType`, subject mismatch fail closed. |

---

## References

- Python guide: https://pulpproject.org/pulp_python/docs/user/guides/attestation/
- DSSE: https://github.com/secure-systems-lab/dsse
- in-toto envelope: https://github.com/in-toto/attestation/blob/main/spec/v1/envelope.md
- SLSA provenance v1: https://slsa.dev/spec/v1.0/provenance
- VSA predicate: https://github.com/in-toto/attestation/blob/main/spec/predicates/vsa.md
- Producer stub (SLSA): https://github.com/calungaproject/plumbing/blob/main/utils/scripts/generate-and-sign-npm-attestations
- npm public provenance (future parity, not MVP): https://github.com/npm/provenance
- Related follow-ups: `docs/prod_followup.md`

---

## Suggested Jira title / description blurb

**Title:** pulp_npm: host package attestations as cosign DSSE (in-toto allowlist)

**Description:** Implement npm package attestation content (prefer `attestations` over provenance-only naming) storing cosign DSSE / in-toto statements linked to npm packages by tarball digest. MVP allowlist includes SLSA provenance v1 and VSA; uniqueness is one attestation per (package, predicateType) per repo version. Upload backfill + preferably with package (atomic). Non-goal: npm CLI Sigstore Bundle parity; non-goal: crypto verify inside Pulp.
