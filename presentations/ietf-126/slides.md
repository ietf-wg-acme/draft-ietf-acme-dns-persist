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
  ul {
    line-height: 1.25;
    margin-top: 0.1em;
    margin-bottom: 0.4em;
  }
  li {
    margin: 0.1em 0;
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
    font-size: 0.85em;
    margin-top: 0.3em;
    margin-bottom: 0.3em;
  }
  th {
    background-color: #f0f4fa;
  }
---

<!-- markdownlint-disable MD013 MD033 MD034 MD060 MD025 MD001 -->

<!-- _class: lead -->

# ACME Persistent DNS Challenge

## draft-ietf-acme-dns-persist

*A long-lived DNS TXT record, carrying an `accounturi` and a `persistUntil` expiry, authorizes one CA account to issue for a domain with no action needed at each issuance.*

**Shiloh Heurich** (Fastly)
**Henry Birge-Lee** (Crosslayer Labs)
**Michael Slaughter** (Amazon Trust Services)

IETF 126 ACME WG — Vienna

---

# Since IETF 125: Choosing a Binding Mechanism

## Security guidance corrected

- The CA-side DNSSEC guidance now protects the **ACME directory URL**, the name clients resolve and connect to; `issuer-domain-name` is an opaque identifier inside the record

## Three approaches to substitution hardening

| Approach | Outcome |
| --- | --- |
| [Authenticated POST to supplied `accounturi`](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/62) | **Closed:** requires a live endpoint and online client |
| [`pubkey:` hash of the account key](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/65) | **Closed:** new wire format; DNS update on rollover |
| [Hash name + key + account URL](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/67) | **Selected** (open draft): offline binding with rollover continuity |

<div class="ask">

**Goal:** restore non-transferability; keep offline provisioning and rollover.

</div>

---

<!-- _class: discuss -->

# The Substitution Gap

### Mode 3a — raw accounturi (dropped)

<div class="seq">
<div class="lane">Victim (domain owner)</div>
<div class="lane">Adversary ACME proxy</div>
<div class="lane">Real ACME server</div>
<div class="msg s24"><span class="num">1</span> Adversary runs the proxy and holds account <code>A</code> at the real server</div>
<div class="msg s13"><span class="num">2</span> Victim creates an account at what it believes is the CA →</div>
<div class="msg s13 attack"><span class="num">3</span> ← Proxy returns attacker's account URL <code>A</code> as the victim's own</div>
<div class="msg c1 side"><span class="num">4</span> Victim publishes TXT: <code>accounturi=A</code></div>
<div class="msg s24"><span class="num">5</span> Adversary orders a cert for victim's domain under account <code>A</code> →</div>
<div class="msg c3 attack"><span class="num">6</span> Simple String Comparison passes against account <code>A</code>; cert issued to attacker</div>
</div>

<div class="ask">

**Fail closed:** the hashed form defeats this. The real server recomputes over the requesting account's proven key and URL, so a poisoned record validates for no account.

</div>

---

<!-- _class: discuss -->

# Offline Binding with Account Continuity

The client and CA independently construct the same hashed `accounturi`:

```text
<accountHashPrefix><hash-alg>/<base64url hash value>
H(length_of_domain || domain_name || key || account_URL)

Example:
_validation-persist.example.com. IN TXT "example.ca accounturi=https://example.ca/acme/hash/sha-256/47DEQpj8HBSa..."
```

- `domain_name` prevents cross-name reuse; `key` supplies the client-computed proof
- `account_URL` binds the value to one account; the 1-octet length plus the fixed-length `key` make every field boundary unambiguous
- `sha-256` identifies SHA-256 (`H`); future algorithms use different tokens
- The CA **recomputes** and string-compares; prior-key matching supports rollover continuity
- `accountHashPrefix` (new directory `meta` field) lets the client build the URI offline

<div class="ask">

**Why this design:** a poisoned value fails validation instead of transferring authorization, without requiring a live account endpoint or a public-key URI scheme.

</div>

---

<!-- _class: discuss -->

# Mandatory Hashed-URI Binding

The post-`-01` working copy grew three `accounturi` modes. The current `-02` direction **collapses them to one**:

| Working mode | `accounturi` | `-02` disposition |
|------|--------------|-------------|
| **3a** | ACME account URL (raw) | **Dropped:** leaves the substitution gap open |
| **3b** | Alternative CA-assigned URI | **Redefined:** the client-computed hashed URI is the sole form |
| **3c** | Subscriber-account URI (many) | **Dropped:** remains available under CA/B Forum BR 3.2.2.4.22 and CA policy (CP/CPS) |

**No raw-URL downgrade.**

<div class="ask">

**Ask:** any WG objection before we publish `-02` along these lines?

</div>

---

<!-- _class: discuss -->

# Final Construction Review

## Settled in review

- Exact recomputation over a fixed field structure makes length extension irrelevant
- `account_URL` provides RFC 8657 §5.4 uniqueness; the record's FQDN adds per-name privacy and prevents cross-name reuse

## Interoperability edits before `-02`

- Use the unpadded base64url RFC 7638 thumbprint ACME libraries already produce
- Make `sha-256` mandatory; future tokens come from the RFC 6920 registry (non-truncated, at least 256 bits)
- Bound prior-key acceptance by time without adding a client key hint

## Privacy limit

HTTP-01 exposes the account-key thumbprint on the CA's cleartext validation fetch; hashing is best-effort anti-correlation, not strong unlinkability.

---

# What Lands in `-02`, and Why

<div class="columns">
<div>

## Land now: author-settled and scoped

- Avoid provisioning an already-expired record ([#38](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/38))
- Separate DNS cache lifetime from authorization lifetime ([#42](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/42))
- Align challenge-object naming with ACME JSON ([#56](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/56))
- Discover issuer names before creating an order ([#60](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/60))

</div>
<div>

## Defer past `-02`

- IP validation ([#32](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/32)): defer to keep this revision focused
- Subdomains ([#33](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/33)): choose RFC 9444 reuse or a new signal
- Error codes ([#39](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/39)): decide whether to register challenge-specific errors
- Mismatch behavior ([#59](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/59)) and substitution binding ([#64](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/64)) close with the hashed URI
- Privacy analysis ([#61](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/61)): keep open

</div>
</div>

---

# Path Forward

## Settled fixes in the `-02` candidate

- Rotation-text rewrite, RFC 8657 clarification, base64url thumbprint with new vectors
- RFC 6920 hash tokens (mandatory `sha-256`), HTTP-01 privacy caveat, and the four scoped fixes
- IANA early review requested for the new directory metadata

## Two extensions, two states

- **Current author proposal — keep `domain_name = "*"`** (client MAY / CA MUST): authors aligned, WG review pending; a DNS observer can link the account's domains; one record reused across names
- **Post-IETF design — key rollover durability:** not author-settled; bounded `keyRolloverWindow` vs. non-breaking records, both must preserve non-transferability

Submission blackout: gather feedback Thursday, confirm on-list, then publish `-02`.

<div class="ask">

**Room ask:** objections to the star-form proposal? Input for the rollover design?

</div>

---

<!-- _class: lead -->

# Questions & Discussion

- Any objection to dropping 3a and 3c and making the hashed URI mandatory?
- Input on the post-IETF rollover design: bounded prior-key window vs. records that survive rollover?
- Authors propose keeping the domain-omitted `*` form (client MAY / CA MUST): objections?

**Parallel CA/Browser Forum work** (BR Method 3.2.2.4.22):
[#674](https://github.com/cabforum/servercert/issues/674) one-to-many `accounturis` · [#675](https://github.com/cabforum/servercert/issues/675) parameter-key case-sensitivity
[#676](https://github.com/cabforum/servercert/issues/676) `policy=wildcard` · [#677](https://github.com/cabforum/servercert/issues/677) separate validation method for dns-persist-01

Mailing list: acme@ietf.org<br>
GitHub: [current `-02` candidate and discussion](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/pull/67) · [repository](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist)<br>
Draft: https://datatracker.ietf.org/doc/draft-ietf-acme-dns-persist/
