---
marp: true
theme: default
size: 16:9
paginate: true
style: |
  section {
    background-color: #ffffff;
    color: #333333;
    font-family: 'Helvetica Neue', Arial, sans-serif;
    font-size: 28px;
    padding: 15px 70px 30px 70px;
  }
  section:not(.lead) {
    justify-content: flex-start;
  }
  h1 {
    color: #1a5490;
    font-size: 1.55em;
    border-bottom: 3px solid #1a5490;
    padding-bottom: 0.15em;
    margin-top: 0;
    margin-bottom: 0.2em;
  }
  h2 {
    color: #2c5aa0;
    font-size: 1.2em;
    margin-top: 0;
    margin-bottom: 0.3em;
  }
  h3 {
    color: #3d6ab0;
    font-size: 1.05em;
    margin-top: 0;
    margin-bottom: 0.2em;
  }
  code {
    background-color: #f4f4f4;
    padding: 0.15em 0.35em;
    border-radius: 3px;
    font-family: 'Courier New', monospace;
  }
  pre {
    background-color: #f8f8f8;
    border: 1px solid #ddd;
    border-radius: 5px;
    padding: 0.6em;
    font-size: 0.7em;
    line-height: 1.2;
  }
  ul, ol {
    line-height: 1.22;
    margin-top: 0.1em;
    margin-bottom: 0.35em;
  }
  li {
    margin: 0.08em 0;
  }
  strong {
    color: #1a5490;
  }
  .columns {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1em;
  }
  .seq {
    display: grid;
    grid-template-columns: repeat(3, 1fr);
    gap: 0.3em 0.8em;
    font-size: 0.68em;
    margin-top: 0.3em;
  }
  .seq .lane {
    text-align: center;
    font-weight: bold;
    color: #1a5490;
    border-bottom: 3px solid #1a5490;
    padding-bottom: 0.1em;
  }
  .seq .msg {
    border: 1.5px solid #2c5aa0;
    border-radius: 5px;
    background: #f0f4fa;
    padding: 0.15em 0.5em;
  }
  .seq .c1 { grid-column: 1; }
  .seq .c3 { grid-column: 3; }
  .seq .s13 { grid-column: 1 / 3; }
  .seq .s24 { grid-column: 2 / 4; }
  .seq .attack { border-color: #c0392b; background: #fdecea; }
  .seq .attack code { background: #f9d7d3; }
  .seq .attack strong { color: #c0392b; }
  .seq .side { border-style: dashed; background: #fffbf0; border-color: #d4940a; }
  .num {
    display: inline-block;
    background: #1a5490;
    color: #ffffff;
    border-radius: 50%;
    width: 1.3em;
    height: 1.3em;
    line-height: 1.3em;
    text-align: center;
    font-size: 0.85em;
    margin-right: 0.25em;
  }
  /* Discussion slides — gold accent signals "we need your input" */
  section.discuss {
    border-left: 8px solid #d4940a;
    padding-left: 62px;
  }
  section.discuss h1 {
    color: #b37800;
    border-bottom-color: #d4940a;
  }
  /* Question/ask box */
  .ask {
    background: #fffbf0;
    border: 2px solid #d4940a;
    border-radius: 6px;
    padding: 0.3em 0.7em;
    margin-top: 0.3em;
  }
  .ask strong {
    color: #b37800;
  }
  table {
    font-size: 0.82em;
    margin-top: 0.25em;
    margin-bottom: 0.25em;
  }
  th {
    background-color: #f0f4fa;
  }
---

<!-- markdownlint-disable MD013 MD033 MD034 MD060 MD025 MD001 -->

<!-- _class: lead -->

# ACME Persistent DNS Challenge

## draft-ietf-acme-dns-persist

*A long-lived DNS TXT record, carrying a hashed `accounturi` and an optional `persistUntil` expiry, authorizes one CA account to issue for a domain or IP address with no DNS change at each issuance.*

**Shiloh Heurich** (Fastly)
**Henry Birge-Lee** (Crosslayer Labs)
**Michael Slaughter** (Amazon Trust Services)

IETF 127 ACME WG — San Francisco

---

# Since IETF 126 (Vienna)

<div class="columns">
<div>

## Published in `-02` (Sept 2026)

- **Mandatory hashed `accounturi`**: `H(len || domain_name || key || account_URL)` under `accountHashPrefix` ([#64](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/64))
- **Domain-omitted `*` opt-out**: client MAY / CA MUST
- **Rollover durability**: CAs retain witnessed account-key thumbprints for account lifetime; account deactivation overrides retention ([#69](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/69))
- **`persistUntil` clamp**: clamps authorization `expires`
- **Directory `issuerDomainNames`**: pre-order discovery ([#60](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/60))

</div>
<div>

## Landed in `-03`

- [ASCII-only hash canonicalization](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/89) (after [#77](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/77), [#79](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/79); list feedback): validated A-label input, ASCII lowercase + strip trailing dot; exact `issuer-domain-name` match
- [Case-insensitive parameter names](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/80): `accounturi`, `policy`, `persistUntil`; duplicate recognized names rejected
- [`accountHashPrefix` syntax](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/81): fits RFC 8659 TXT `value` grammar
- [Validation flows & security limits](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/82) + [normative & JIT guidance](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/83) + [per-attempt cost](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/75)

</div>
</div>

---

<!-- _class: discuss -->

# Subdomain Validation: `ancestorDomain` ([#33](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/33))

<div class="columns">
<div>

### Request-only selector ([PR #84](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/84))

```json
POST /chall/abc  →  { "ancestorDomain": "example.com" }
```

- **Three names:** cert identifier `F`, authz `identifier.value` `A` (`*.` stripped), record domain `R`
- Default `{}` sets `R = A` (including base wildcard `*.example.com`)
- Subdomain or deeper wildcard (`*.dept.example.com`) sends `"ancestorDomain": "example.com"`
- Challenge Object schema unchanged

</div>
<div>

### Guardrails & RFC 9444

- **CA MUST recognize:** no silent fallback to `A`
- **Proper ancestor:** strict suffix of `A`; VDN ≤ 253 octets
- **Public suffixes: SHOULD NOT** accept (`unauthorized` if declined)
- **Requires `policy=wildcard`** at `R`; digest hashes `R` (or `*`)
- **First accepted POST locks `R`** for the attempt
- **RFC 9444 (`subdomainAuthAllowed`):** coexists; satisfies only `A`

</div>
</div>

<div class="ask">

**Why:** Client names the record location; no DNS tree walk.

</div>

---

<!-- _class: discuss -->

# IP Validation: `_ip-validation-persist` ([#32](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/32))

Additive profile for ACME `ip` identifiers (RFC 8738) in the reverse DNS tree ([PR #85](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/85)):

```text
_ip-validation-persist.1.2.0.192.in-addr.arpa. IN TXT "authority.example; accounturi=https://ca.example/account-hash/sha-256/CnwSwHOHJbE-..."
```

- **Normalized Reverse Zone Domain Name (`R`):** lowercase `.in-addr.arpa` (4 reversed octets) or `.ip6.arpa` (32 reversed hex nibbles, 72 octets)
- **Distinct node name:** `_ip-validation-persist.<R>` (IANA Node Names + ACME Validation Methods `ip` registrations; no `PTR` indirection)
- **Digest `domain_name` input:** normalized `R` (or `*`) prior to following any CNAME aliases — deterministic across RFC 2317 CNAME delegation
- **Exact-address scope:** single IP only (`policy=wildcard` ignored; `ancestorDomain` rejected with `malformed`; caches typed by `ip` vs `dns`)

<div class="ask">

**Why:** Same record format; separate node name keeps `ip` and `dns` apart.

</div>

---

<!-- _class: discuss -->

# `persistExpired` Error Type ([#39](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/39) / [PR #86](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/86))

Registers `urn:ietf:params:acme:error:persistExpired` (RFC 8555 §9.7.4):

| Rule | Requirement |
| --- | --- |
| **1. Privacy Gate (`MUST NOT`)** | CA **MUST NOT** return `persistExpired` if any record at the queried name validates, or if no record at that name is otherwise qualifying for the CA and **live requesting account** and failed solely due to an expired `persistUntil` (order-independent across RRsets). |
| **2. Diagnostic (`SHOULD` / `MAY`)** | When eligible on a challenge attempt, CA **SHOULD** return `persistExpired`, and **MAY** return `unauthorized` for uniform disclosure. |

- **Just-in-Time validation:** continues to fall back to a `pending` challenge
- **Clients:** check `persistUntil` locally before ordering; handle both `persistExpired` and `unauthorized`

<div class="ask">

**Why this split:** Actionable expiration diagnostics with zero cross-account or cross-CA expiration oracle.

</div>

---

<!-- _class: discuss -->

# Certificate-Key Attestation Conjunction ([#76](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/76))

Why DNS parameters and CAA alone cannot express `dns-persist-01` **AND** `device-attest-01`:
- Authorization `challenges` (RFC 8555 §7.1.4) and CAA `validationmethods` (RFC 8657 §4) are **disjunctive (`OR`)**; unknown TXT parameters are `MUST ignore`

## Order-level conjunctive policy ([PR #87](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/87))

1. **Non-waivable Order `authorizations` (RFC 8555 §7.1.3):** CA **MUST** include the attestation authorization on every Order in scope (covering wildcard/subordinate names) and **MUST NOT** allow client waiver
2. **CSR key binding at finalization (RFC 8555 §7.4):** CA **MUST** verify the CSR public key matches the Order's attested key — even across JIT or reused `dns` authorizations
3. **Key-rotation renewal:** JIT reuses `_validation-persist`; new Order requires fresh attestation for the rotated CSR key (`device-attest` cited informatively)

<div class="ask">

**Why:** Orders already require every authorization; no new syntax.

</div>

---

<!-- _class: discuss -->

# Mailing-List Implementation Feedback (Oct 13)

From [Seo Suchan's Pebble `-02` implementation report](https://mailarchive.ietf.org/arch/msg/acme/Qm1zUtpcUR5dogeTawwlSaFRTbU/):

<div class="columns">
<div>

## 1. Keep `https://` prefix

- **RFC 8657 §5.4:** `accounturi` URIs include an authority component
- **Zero directory savings:** pre-provisioning already uses directory `issuerDomainNames`
- **No BCP 35 hold:** avoids IANA scheme registration; aligned with [servercert PR #9](https://github.com/slghtr-says/servercert/pull/9)
- **Rebrands:** covered by §4.5 dual-acceptance migration

</div>
<div>

## 2. Keep inner `key` on SHA-256

```text
H(len || domain_name || key || account_URL)
                        ^^^ 43-octet RFC 7638
```

- **Matches RFC 8555 §8.1:** reuses the exact `keyAuthorization` thumbprint clients and CAs already compute
- **One stored value per key:** CA retention table stores **one** 43-byte thumbprint per key regardless of outer `<hash-alg>`

</div>
</div>

<div class="ask">

**Direction:** Keep `https://` `accountHashPrefix` and fixed SHA-256 key input.

</div>

---

# Implementation Status & Path to WGLC

<div class="columns">
<div>

## Issue & PR Status

- **Clarifications since `-02`:**
  - [#74](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/74), [#75](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/75), [#77](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/77)–[#83](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/83), [#89](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/89)
- **New in `-03`:**
  - [PR #84](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/84): `#33` `ancestorDomain` + RFC 9444
  - [PR #85](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/85): `#32` `_ip-validation-persist`
  - [PR #86](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/86): `#39` `persistExpired`
  - [PR #87](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/87): `#76` attestation conjunction
- **All tracked issues addressed**

</div>
<div>

## Implementations & CABF

- **Pebble:** merged ([#536](https://github.com/letsencrypt/pebble/pull/536)); `-02` fork tested
- **lego:** merged ([#2871](https://github.com/go-acme/lego/pull/2871))
- **cert-manager:** design & prototype ([#8373](https://github.com/cert-manager/cert-manager/issues/8373))
- **CAs:** Let's Encrypt, Fastly / Certainly, Amazon Trust Services, Google Trust Services
- **CA/B Forum:** draft ballot ([servercert PR #9](https://github.com/slghtr-says/servercert/pull/9)) aligned with `-03`

</div>
</div>

<div class="ask">

**Next step:** Request **WGLC** on `-03`.

</div>

---

<!-- _class: lead -->

# Questions & Discussion

- Feedback on how `-03` resolves `#32`, `#33`, `#39`, and `#76`
- Is `-03` ready for **Working Group Last Call**?

Mailing list: acme@ietf.org<br>
GitHub: [repository](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist)<br>
Draft: https://datatracker.ietf.org/doc/draft-ietf-acme-dns-persist/
