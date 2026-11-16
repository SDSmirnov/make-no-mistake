# Evidence, oracles, and domain verification

Read this when challenging an oracle's independence, testing external contracts or stateful guards, or verifying non-software work. Core provenance, redaction, and claim-tagging rules live in [`SKILL.md`](../SKILL.md).

## Provenance and redaction details

- **Executed checks:** record the exact command, working directory, branch/commit or version, environment, and exit code or pass count. Confirm target checks actually ran:
  - a non-zero passed count for each named target test (`0 collected` is not a pass);
  - no skipped or filtered target test;
  - no masked exit code (`cmd | tail` without `pipefail` hides failures);
  - no stale build, artifact, or cache.
- **Source, data, or document checks:** record the authoritative source, version or effective date, jurisdiction or scope where applicable, and exact locator (section, passage, query, or `file:line`).
- **Redaction mechanics:** remove secrets, tokens, credentials, personal data, and customer identifiers from ledgers, scratch files, reports, and verifier prompts. Keep secrets out of command lines, shell history, and transcripts; use environment variables, files, or stdin. Cite the command, artifact path, `file:line`, hash, or redacted excerpt rather than reproducing sensitive values.

## Oracle provenance and independence

Before treating an observation as evidence that a result is correct, identify the source of its expected outcome:

- **Name the oracle:** user requirement (for intent and product scope, not external facts, laws, or dependency behavior), authoritative specification, primary source, independently calculated result, or observed external behavior.
- **Establish operational independence** from the implementation and its tests:
  - Another test on the same input data, fixture, or formula is not necessarily independent.
  - Another source, wrapper, or summary built on the same mistaken interpretation is not necessarily independent either.
  - An independent check must derive the expected outcome via a **distinct path**, not merely a different tool name.
  - Oracle independence is claim-specific: assess whether the sources share a failure mode for the particular claim being verified.
  - If independence cannot be justified, state that limitation explicitly and do not claim independent verification.
- **For material claims, challenge the oracle itself:** could the contract, expected value, fixture, or source interpretation be wrong?
- **When independent grounding is unavailable,** distinguish *verified against the stated contract* from *verified as factually or externally correct*. External correctness stays `unverified`, or `inferred` only when separate verified premises support it.
- **A test passing against an implementation-derived expectation verifies consistency, not correctness.**

## Mocks, fixtures, and testing the test

- **Treat mocks as assumptions, not external truth.** A unit test against a hand-written mock proves your code matches your mock, not the real dependency (`Mock mirage`). Verify external contracts against a real call, captured fixture, schema, or pinned spec.
- **Never copy production rows** into the repository or test database; use synthetic or irreversibly de-identified fixtures.
- **Confirm every check can fail:**
  - **Bug fixes:** run the new check *before* applying the fix (red-before-green).
  - **New guards:** briefly invert the condition locally, confirm the check fails, then revert that probe edit and confirm `git diff` matches your pre-probe state, preserving any prior edits.
  - **Stateful checks:** run mutating probes only against an isolated copy or disposable container, never against production data or live traffic.
  - **Non-code checks:** verify the source covers the exact date, unit, and jurisdiction, and look for superseding clauses or exceptions.

## Domain-specific verification

Choose the source of truth and independent verification mechanism that fit the domain. Do not force software-style tests where primary-source verification or independent recalculation is the real check:

| Domain | Primary verification mechanism |
| --- | --- |
| **Software** | Tests, runtime behavior, dependency contracts |
| **Finance** | Independent recalculation, source data, units |
| **Legal** | Jurisdiction, authoritative text, effective date |
| **Research** | Primary sources, methodology, reproducibility |
| **Documents** | Source fidelity, citations, rendered output |
| **Operations** | Permissions, blast radius, rollback, recovery |

Recompute numbers by a second method; check jurisdiction and effective date on authoritative text; trace every citation to its actual passage; render documents and read them as the recipient; state a decision's premises and check each one. Ask the user only for intent you cannot infer.

## Three verification lenses

- **Goal fidelity** — does this satisfy the stated goal and the implicit "what they will do with it"?
- **Path validity** — did the method actually work, or merely appear to? Are dependencies and versions what you think?
- **Decision integrity** — were trade-offs made deliberately? For each material decision, name the alternatives, the evidence, and the observation that would reverse it. A decision with no reversal condition is a guess wearing a tie.

## Non-code example (T1 finance/contract check)

**Task:** "Verify our Q3 SaaS credit memo (`$4,320.00` for 36h downtime on a `$90k/yr` contract) before I review it with finance."

- **Oracles & independent check:** open signed MSA `contracts/acme-2026.pdf` §7.2 (effective `2026-01-01`, NY law) instead of summary notes; recompute credit two ways (`monthly_fee * credit_pct` vs annual daily-prorated check).
- **Finding:** §7.2 caps monthly SLA credits at `25%` of monthly fee (`$90,000 / 12 * 0.25 = $1,875.00`), whereas `$4,320.00` erroneously applied a straight hourly rate without the §7.2 cap.

```text
Tier: T1 — local calculation/document audit; no external send or ledger write performed
Outcome: Rejected `$4,320.00` figure; §7.2 caps the credit at `$1,875.00`
Changed: none (verification only)
Verified: `contracts/acme-2026.pdf` §7.2 (signed `2026-01-01`) sets 10% per 6h outage capped at 25% of monthly fee; `python3 -c "print(min(6*0.10, 0.25) * (90000/12))"` -> `1875.0`
Unverified: whether a later signed addendum superseded §7.2 — check legal deal room before issuing memo
Residual risk: unindexed side letter overriding SLA cap; stopped after primary-clause read + independent recalculation
Verifier: none (solo; executed checks verified, unexecuted behavioral claims inferred)
```
