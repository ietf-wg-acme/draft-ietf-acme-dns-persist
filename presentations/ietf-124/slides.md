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
    padding: 15px 70px 50px 70px;
  }
  h1 {
    color: #1a5490;
    font-size: 2.2em;
    border-bottom: 3px solid #1a5490;
    padding-bottom: 0.3em;
    margin-top: 0;
    margin-bottom: 0.4em;
  }
  h2 {
    color: #2c5aa0;
    font-size: 1.5em;
    margin-top: 0;
    margin-bottom: 0.5em;
  }
  h3 {
    color: #3d6ab0;
    font-size: 1.15em;
    margin-top: 0;
    margin-bottom: 0.4em;
  }
  code {
    background-color: #f4f4f4;
    padding: 0.2em 0.4em;
    border-radius: 3px;
    font-family: 'Courier New', monospace;
  }
  pre {
    background-color: #f8f8f8;
    border: 1px solid #ddd;
    border-radius: 5px;
    padding: 1em;
    font-size: 0.75em;
  }
  ul {
    line-height: 1.3;
    margin-top: 0.3em;
    margin-bottom: 0.8em;
  }
  li {
    margin: 0.2em 0;
  }
  strong {
    color: #1a5490;
  }
  .columns {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 1.5em;
  }
---

<!-- _class: lead -->

# ACME Persistent DNS Challenge Update

## draft-ietf-acme-dns-persist-00

**Shiloh Heurich** (Fastly)
**Henry Birge-Lee** (Crosslayer Labs)
**Michael Slaughter** (Amazon Trust Services)

IETF ACME WG - Montreal

---

<!-- _style: "font-size: 0.88em; h1 { font-size: 2.0em; margin-bottom: 0.4em; } h2 { margin-bottom: 0.3em; } ul { margin-bottom: 0.4em; line-height: 1.25; } li { margin: 0.12em 0; } strong { display: inline; }" -->

# Status Update

**WG adopted October 16, 2025**

`draft-sheurich-acme-dns-persist` → `draft-ietf-acme-dns-persist`

**Parallel Industry Progress:**
- **Oct 9:** CA/BF SC088v3 **PASSED** (26 CAs YES, 3 consumers YES)
- **Nov 8:** IP Rights Review completes

**Implementation Status:**
- **Committed (2026):** Fastly/Certainly, Let's Encrypt
- **Assessing:** Amazon Trust Services

**Proof of Concept:** pebble server, eggsampler client
- Full interoperability with regular and wildcard issuance

---

# How dns-persist-01 Works

<div class="columns">

<div>

### Record Format

```dns
_validation-persist.example.com. IN TXT
  "ca.example; accounturi=https://ca.example/acct/123"
```

</div>

<div>

### Key Features

✓ **Persistent** - Reuse for multiple certificates
✓ **Account-bound** - `accounturi` parameter
✓ **CA-specific** - Issuer domain name
✓ **Multi-CA** - Separate TXT records per issuer
✓ **Expiration** - Optional `persistUntil`
✓ **Scope** - `policy=wildcard` covers subdomains

</div>

</div>

---

<!-- _style: "font-size: 0.88em; h2 { font-size: 1.3em; margin-bottom: 0.3em; } h3 { font-size: 1.05em; margin-bottom: 0.25em; } ul { margin-bottom: 0.4em; } li { margin: 0.15em 0; } strong { display: inline-block; margin-bottom: 0.1em; }" -->

# Why We Need This

<div class="columns">

<div>

### Current Challenges

**http-01/tls-alpn-01**
- Port 80/443 required
- Geo-blocking blocks validation
- Wildcards unsupported

**dns-01**
- DNS propagation delays
- API keys on servers risk compromise
- Automation complexity

</div>

<div>

**dns-account-01**
- Scoped labels per account
- Addresses multi-region needs
- Still requires per-validation provisioning

### Current Workarounds

**"Magic CNAMEs"** (acme-dns.io)
- Single point of failure
- DNS cache poisoning risk
- BGP hijacking vulnerability

</div>

</div>

---

# Why Standardize?

CAs could check persistent DNS records outside ACME, but this would bypass the protocol's challenge negotiation where clients choose their validation method.

**Standardization enables proper protocol integration.**

**Key insight:** ACME account URIs provide durable, cryptographic binding

---

<!-- _style: "font-size: 0.72em; h1 { font-size: 1.8em; margin-bottom: 0.2em; margin-top: -10px; } h2 { font-size: 1.15em; margin-bottom: 0.1em; } h3 { font-size: 0.95em; margin-bottom: 0.1em; margin-top: 0.05em; } ul { margin-bottom: 0.15em; margin-top: 0; line-height: 1.05; } li { margin: 0.02em 0; padding: 0; font-size: 0.95em; }" -->

# Design Principles

<div class="columns">

<div>

### Strong Account Binding
- **ACME accounturi**: Durable identity
- Prevents unauthorized use with DNS access alone
- Survives account key rotation
- No new trust anchors

### Multi-CA Architecture
- **Per-issuer TXT records**
- Each CA validates only their records
- No CA coordination required
- Domain owner controls authorization

</div>

<div>

### Flexibility & Extensibility
- **Optional parameters:**
  - `persistUntil` - explicit expiration
  - `policy=wildcard` - subdomain coverage
- **Unknown parameters ignored**
- Enables future protocol extensions

### Security Constraints
- **MUST respect DNS TTL**
- Bounded by CA policy limits
- DNSSEC SHOULD be validated

</div>

</div>

---

<!-- _style: "font-size: 0.68em; h1 { font-size: 1.75em; margin-bottom: 0.15em; margin-top: -10px; } h2 { font-size: 1.1em; margin-bottom: 0.05em; margin-top: 0; } h3 { font-size: 0.92em; margin-bottom: 0.05em; margin-top: 0.1em; } ul { margin-bottom: 0.1em; margin-top: 0; line-height: 1.05; } li { margin: 0.01em 0; padding: 0; font-size: 0.95em; } p { margin: 0.05em 0; } strong { display: inline; }" -->

# Active WG Discussions

<div class="columns">

<div>

### Security Trade-offs
- Freshness vs. operational simplicity
- Account key becomes long-lived credential
- **Key compromise:** Immediate issuance without DNS access
- **Privacy:** `accounturi` in public DNS

### DNSSEC Validation
- **Draft:** SHOULD validate signatures
- **Alternative:** MUST use validating resolver
- **Trade-off:** Security vs. private PKI flexibility

</div>

<div>

### Validation Reuse Period

**Effective period = shortest of:**
- DNS record TTL (MUST respect)
- `persistUntil` parameter
- CA/BF policy (398d → 10d by 2029)

</div>

</div>

---

# Evolution to WG Draft

Changes from `draft-sheurich-acme-dns-persist-00` through WG adoption:

**Just-in-Time Validation**
  - CA checks for existing DNS records when authorization requested
  - If valid record found → instant "valid" status (no challenge needed)
  - If no record found → normal challenge flow continues

**Security Considerations expanded** - Record risks, account binding, subdomain validation

**Long TXT record guidance** - Multi-string format for >255 characters

**Error handling** - `malformed` for syntax, `unauthorized` for auth failures

**Document renamed** - `draft-ietf-acme-dns-persist` (WG adoption)

---

<!-- _style: "font-size: 0.85em; h2 { font-size: 1.3em; margin-bottom: 0.3em; } ul { margin-bottom: 0.4em; } li { margin: 0.15em 0; } ol { margin-bottom: 0.4em; }" -->

# Seeking WG Input

**Acknowledging concerns:** Use case definition, trust relationships, existing validation interactions

<div class="columns">

<div>

**Questions:**

1. **DNSSEC requirement?**
   - Draft: SHOULD validate
   - Alternative: MUST validate
   - Which level?

2. **Security considerations?**
   What else to address?

</div>

<div>

3. **Timeline?**
   Given industry momentum

4. **AccountURI flexibility?** (PR #30)
   Multiple URIs per account?
   - **Pro:** Privacy, access control
   - **Con:** CA cannot predict client's choice
   - Trade-off: Privacy vs. simplicity

</div>

</div>

---

# Path Forward

- Incorporate Montreal feedback
- Expand security considerations
- Resolve PR #30 (accounturi flexibility)
- Address use case and trust concerns
- Target WGLC after 1-2 revisions
- Coordinate with CA/Browser Forum

**Feedback → Revision → WGLC → RFC**

---

<!-- _class: lead -->

# Questions & Discussion

**Thank you!**

**Contact:**
- Mailing list: acme@ietf.org
- GitHub: https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist
- Draft: https://datatracker.ietf.org/doc/draft-ietf-acme-dns-persist/
