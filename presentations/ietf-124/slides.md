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
    padding: 15px 70px 100px 70px;
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
  .timeline {
    background: linear-gradient(to right, #1a5490 0%, #4a90e2 100%);
    color: white;
    padding: 1em;
    border-radius: 5px;
    margin: 1em 0;
  }
  .timeline strong {
    color: white;
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

# Status Update

**WG adopted October 16, 2025**

`draft-sheurich-acme-dns-persist` → `draft-ietf-acme-dns-persist`

**Authors:** Shiloh Heurich (Fastly), Henry Birge-Lee (Crosslayer Labs), Michael Slaughter (Amazon Trust Services)

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

<!-- _style: "font-size: 0.85em; h2 { font-size: 1.3em; margin-bottom: 0.3em; } h3 { font-size: 1em; margin-bottom: 0.3em; } ul { margin-bottom: 0.4em; } li { margin: 0.15em 0; }" -->

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

**dns-account-01** (draft-ietf-acme-dns-account-challenge)
- Scoped labels per account
- Addresses multi-region needs
- Still requires per-validation provisioning

</div>

<div>

### Current Workarounds

**"Magic CNAMEs"** (acme-dns.io)
- Single point of failure
- DNS cache poisoning risk
- BGP hijacking vulnerability

</div>

</div>

---

# Why Standardize?

CAs could deploy via pre-validation without protocol changes, but this bypasses ACME's challenge selection.

**Standardization enables proper protocol integration.**

**Key insight:** ACME account URIs provide durable, cryptographic binding

---

# Implementation Support

<div class="timeline">

**Timeline:**
- **Oct 9:** CA/BF SC088v3 **PASSED** (26 CAs YES, 3 consumers YES)
- **Oct 9 - Nov 8:** IP Rights Review Period
- **Oct 16:** IETF WG adoption
- **2026:** Let's Encrypt Boulder implementation

</div>

### Status

**Committed:** Fastly/Certainly, Let's Encrypt

**Assessing:** Amazon Trust Services

**Proof of Concept:** Pebble server fork, eggsampler client fork

---

<!-- _style: "font-size: 0.85em; h2 { font-size: 1.3em; margin-bottom: 0.3em; } h3 { font-size: 1em; margin-bottom: 0.3em; } ul { margin-bottom: 0.4em; } li { margin: 0.15em 0; }" -->

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
- Related: draft-ietf-dnsop-domain-verification-techniques

</div>

<div>

### Validation Reuse Period
How long CA relies on one DNS check

**Effective period = shortest of:**
- DNS record TTL (MUST respect)
- `persistUntil` parameter
- CA policy (398 days → 10 days by 2029)

</div>

</div>

---

# Evolution to WG Draft

Changes from `draft-sheurich-acme-dns-persist-00` through WG adoption:

✓ **Pre-validation optimization**
  - CA checks existing records during order creation
  - Authorization becomes "valid" immediately if found
  - Skips challenge flow with persistent record

✓ **Security Considerations expanded** - Record risks, account binding, subdomain validation

✓ **Long TXT record guidance** - Multi-string format for >255 characters

✓ **Error handling** - `malformed` for syntax, `unauthorized` for auth failures

✓ **Document renamed** - `draft-ietf-acme-dns-persist` (WG adoption)

---

<!-- _style: "font-size: 0.85em; h2 { font-size: 1.3em; margin-bottom: 0.3em; } ul { margin-bottom: 0.4em; } li { margin: 0.15em 0; } ol { margin-bottom: 0.4em; }" -->

# Seeking WG Input (1/2)

**Acknowledging concerns:** Use case definition, trust relationships, existing validation interactions

**Questions:**

1. **DNSSEC requirement?**
   - Draft: SHOULD validate
   - Alternative: MUST validate
   - Which level?

2. **Protocol validation caps?**
   Beyond `persistUntil`

3. **Security considerations?**
   What else to address?

---

# Seeking WG Input (2/2)

4. **Timeline?**
   Given industry momentum

5. **AccountURI flexibility?** (PR #30)
   Multiple URIs per account?
   - **Pro:** Privacy, access control
   - **Con:** CA cannot predict client's choice
   - Trade-off: Privacy vs. simplicity

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
