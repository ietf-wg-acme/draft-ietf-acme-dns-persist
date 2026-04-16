# Copilot Review Instructions

This repository contains `draft-ietf-acme-dns-persist`, an IETF Standards Track Internet-Draft specifying an ACME challenge type for persistent DNS-based domain control validation. The document is authored in kramdown-rfc markdown and processed by the `kramdown-rfc` toolchain into xml2rfc XML and then RFC-format text.

Review every pull request against the rules in this file. Flag violations with the relevant section reference from this file (not from the draft — the draft's section numbers change).

---

## 1. Document Format: kramdown-rfc

Hard constraints on the source format:

- **Bibliographic references**: Normative references use `{{!RFCNNNN}}` (with `!`). Informative references use `{{?RFCNNNN}}` (with `?`). Bare `{{RFCNNNN}}` is a build error. Every RFC cited in normative text MUST use `{{!...}}`.
- **Section anchors**: `{#anchor-name}` after headings, lowercase-hyphenated. Cross-references use `{{anchor-name}}`. Verify all cross-references resolve.
- **Section markers**: `--- abstract`, `--- middle`, `--- back` delimit document structure. Content before `--- abstract` is frontmatter only.
- **BCP 14 boilerplate**: The directive `{::boilerplate bcp14-tagged}` must remain in the Conventions and Definitions section.
- **Figures**: Fenced code blocks use `~~~` (not backticks). Figure titles use `{: #fig-id title="..."}` on the line after the closing fence.
- **Definition lists**: Use kramdown `:` syntax.
- **Frontmatter**: YAML block between `---` delimiters. Do not modify frontmatter structure without understanding xml2rfc implications.

---

## 2. Normative Language (BCP 14)

This document uses BCP 14 keywords per RFC 2119 and RFC 8174.

- **MUST / MUST NOT**: Absolute. Use only when non-compliance breaks interoperability or security. Every MUST imposes an implementation burden — verify it is justified.
- **SHOULD / SHOULD NOT**: Expected behavior with permitted deviation. RFC 2119 requires the specification to state what happens when the requirement is not met. Flag any bare SHOULD without a deviation consequence.
- **MAY**: Truly optional. Verify interoperability is preserved whether or not the feature is implemented. If a MAY creates a dependency on other implementations, it may need to be SHOULD or MUST.
- **Capitalization**: BCP 14 keywords are normative only when fully capitalized. Lowercase "must", "should", "may" are ordinary English. Flag accidental capitalization in non-normative contexts.
- **Consistency**: Two statements about the same behavior must use the same keyword strength. Flag contradictions.

---

## 3. Protocol Review Philosophy

### Persistence Changes Everything

This protocol's defining property is that the validation signal persists in DNS. Every review question flows from this: persistent records have longer exposure windows, survive infrastructure changes, and remain valid even when the domain owner's intent has changed. Any new text must be evaluated against the question: "What is the impact of this being long-lived?"

### Cross-Flow Consistency

The draft defines multiple paths to validation. When a PR modifies one path, check all others:

- Some paths operate without a challenge object. Requirements that depend on challenge object fields must account for their absence.
- Some paths skip the challenge-response interaction. Requirements that assume client interaction must account for flows where the CA acts unilaterally.
- The challenge response payload is an extension point (RFC 8555 §7.5.1). New fields must degrade gracefully with servers that ignore them.

### Forward Compatibility

The draft requires unknown parameters to be ignored. This is a hard design constraint: any new parameter that requires enforcement cannot rely on existing implementations recognizing it. Flag PRs that introduce parameters assuming universal strict enforcement.

---

## 4. Account Identity Review Philosophy

The `accounturi` mechanism is the most security-sensitive part of the protocol. The draft supports multiple account binding modes with different security, privacy, and operational properties. Read the current mode definitions in the draft before reviewing any PR that touches account handling.

### Principles

- **Uniform coverage or explicit qualification.** Text that references account behavior must either apply to all modes or state which modes it covers. Language like "the ACME account" when the context includes non-ACME account URIs is a bug. Check every use of "account" in changed text.
- **Comparison semantics vary by mode.** Some modes use byte-for-byte string comparison (RFC 3986 §6.2.1). Others require CA-internal resolution. Verify that comparison language is qualified.
- **Deactivation semantics vary by mode.** Deactivating a single account may or may not invalidate a group-level authorization. The revocation section must correctly describe all cases.
- **Client verification varies by mode.** Some modes are independently verifiable by the client; others require trusting the CA. Client-facing guidance must distinguish these cases.
- **Privacy properties are mode-dependent and sometimes contradictory.** Some modes reduce correlation; others increase it. Privacy text must correctly characterize each mode.
- **RFC 8657 is the foundation.** The account model builds on RFC 8657 §3 ("CA account" = "a specific entity or group of related entities"), §5.4 (URI uniqueness), and §5.9 (URI revelation). Changes must remain consistent with these definitions.

---

## 5. Security Review Philosophy

### Combinatorial Interaction Surface

Persistent validation creates a combinatorial security surface. Features that individually seem safe can compound dangerously. When a PR adds or modifies any feature, verify that the Security Considerations address its interactions with every other feature. Four interaction patterns recur:

- **Scope amplification.** Features that broaden who can use an authorization and features that broaden what it authorizes compound multiplicatively. The worst case is always the combination of all scope-expanding features simultaneously.
- **Detection window elimination.** Some validation paths skip interactions that would otherwise provide a window for detecting unauthorized use. Features that grant authorization without client interaction must be analyzed for abuse by entities the domain owner did not anticipate.
- **Forward authorization.** Persistent records can authorize entities that do not exist at provisioning time. Any account model where membership can grow after record provisioning creates forward authorization.
- **Revocation asymmetry.** Different account binding modes have different revocation granularity. Verify the revocation section handles every mode, and that "deactivate the account" is not presented as a universal emergency measure when it is only effective for some modes.

### Threat Model

Four threat vectors apply to any persistent validation mechanism:

- **DNS compromise.** Persistence amplifies impact — a compromised record remains exploitable for its lifetime, not a momentary challenge window.
- **Account key compromise.** Pre-existing persistent authorizations are immediately usable without DNS access.
- **Account management system compromise.** For group-level account models, this allows adding accounts that inherit existing authorizations — distinct from key compromise.
- **Stale records.** Records that outlive their intended authorization period.

Verify that new text does not weaken existing mitigations and that new features are analyzed against each vector.

### DNSSEC Stance

The draft uses SHOULD (not MUST) for DNSSEC-validating resolvers — a deliberate choice reflecting deployment reality. If DNSSEC validation is performed and fails, the draft requires fail-closed behavior. This is intentionally stricter than RFC 8555's general DNSSEC guidance because persistent records have longer exposure. Do not change the SHOULD/MUST level or the fail-closed requirement without WG discussion.

---

## 6. Authoritative References

Check consistency against these external specifications. These are stable — unlike draft internals, they do not change.

### Normative
- **RFC 8555** — ACME protocol. Account lifecycle (§7.3), account deactivation (§7.3.6), challenge response extension point (§7.5.1), dns-01 precedent (§8.4), error types (§9.6).
- **RFC 8657** — CAA `accounturi` and `validationmethods`. "CA account" definition (§3), ACME use (§3.1), non-ACME use (§3.2), URI uniqueness (§5.4), URI revelation privacy (§5.9).
- **RFC 8659** — CAA record format. `issue-value` syntax (§4.2) and parameter tag case-insensitivity (§4.1) — the dns-persist record format builds on this.
- **RFC 3986** — URI syntax. Simple String Comparison (§6.2.1) is the baseline comparison method.
- **RFC 5890** — IDNA. A-label (Punycode) format for domain names.
- **RFC 8552** — Underscored DNS node name registration.

### Informative
- **RFC 4033** — DNSSEC.
- **RFC 9444** — ACME for Subdomains. Related to but distinct from subdomain validation in this draft. Do not conflate the mechanisms.
- **CA/Browser Forum Baseline Requirements** — External policy. The draft aligns with but does not normatively depend on the BRs. BR terminology (e.g., "Authorization Domain Name") was intentionally avoided to prevent definition conflicts.

---

## 7. Review Checklist

### Normative Correctness
- [ ] BCP 14 keywords capitalized only when normative.
- [ ] Every SHOULD has a deviation consequence.
- [ ] No unjustified MUST.
- [ ] No contradictory keyword strengths.
- [ ] Normative text is precise enough for independent implementation.

### Internal Consistency
- [ ] Changes in one section do not contradict another. The record format, verification procedure, revocation, security considerations, and error handling sections form a coupled system — check all of them.
- [ ] All account binding modes are handled wherever account behavior is referenced.
- [ ] All validation flows are handled wherever flow-dependent behavior is referenced.
- [ ] Error handling covers all failure modes introduced or changed by the PR.
- [ ] Examples match normative text.

### Cross-Reference Integrity
- [ ] All `{{...}}` references resolve.
- [ ] New sections have `{#anchor-name}` anchors.
- [ ] Cited RFC section numbers are correct.

### Security
- [ ] New features are analyzed for interaction with all existing features.
- [ ] No security-relevant behavior left unspecified.
- [ ] The forward-compatibility rule (ignore unknown parameters) does not create a hole.
- [ ] Revocation mechanisms are adequate for all account binding modes.
- [ ] Threat model vectors are addressed.

### Format
- [ ] `~~~` fences (not backticks), `{: #id title="..."}` figures, kramdown definition lists.
- [ ] No bare `{{RFCNNNN}}` — must be `{{!...}}` or `{{?...}}`.
- [ ] DNS examples use valid zone-file syntax. URI examples are valid per RFC 3986.

---

## 8. Terminology Traps

These cause persistent confusion. Flag them in any PR:

- **"Validation Domain Name" vs. "Authorization Domain Name"**: This draft defines and uses the former. The latter is a CA/Browser Forum Baseline Requirements term with a different definition. They must not be conflated. This was an intentional terminology choice.
- **"ACME account" vs. "account" vs. "CA account"**: The draft supports account URIs that are not ACME account URLs. Generic "account" language must be used where all modes apply. "CA account" has a specific RFC 8657 §3 definition that is broader than "ACME account."
- **CAA `accounturi` vs. dns-persist `accounturi`**: Independent mechanisms. A URI in one does not automatically apply to the other.
- **DNSSEC "mandatory"**: It is SHOULD, not MUST. The deviation consequence is documented. Do not treat it as a requirement.
