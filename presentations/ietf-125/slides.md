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

<!-- _class: lead -->

# ACME Persistent DNS Challenge

## draft-ietf-acme-dns-persist-01

**Shiloh Heurich** (Fastly)
**Henry Birge-Lee** (Crosslayer Labs)
**Michael Slaughter** (Amazon Trust Services)

IETF 125 ACME WG — Shenzhen

---

# Since IETF 124 (Montreal)

## Merged

- **PR #35 — accounturi decoupled from challenge object** — DNS record is self-contained; enables pre-provisioning and JIT validation; many-to-one URI model for privacy
- **PR #36 — Case-handling rules** explicit for all parameters
- **PR #41 — Renamed** "Authorization Domain Name" → "Validation Domain Name" (BR term conflict)

## Implementation & Status

- **Pebble** testing CA — dns-persist-01 [merged](https://github.com/letsencrypt/pebble/pull/536); **lego** client — [merged](https://github.com/go-acme/lego/pull/2871)
- **cert-manager** — [community feature request](https://github.com/cert-manager/cert-manager/issues/8373) open; [author prototype](https://github.com/sheurich/cert-manager/pull/2)
- -01 publication pending editorial fixes; four open issues on today's agenda

---

<!-- _class: discuss -->

# Subdomain Validation Gap (Issue #33)

<div class="columns">
<div>

### The problem

Client has a record at `_validation-persist.example.com` with `policy=wildcard`

Wants a cert for `server.dept.example.com`

`policy=wildcard` authorizes subdomains, but the CA **does not know which level the client intended**

</div>
<div>

### Validation Domain Name lookup

```text
_validation-persist
  .server.dept.example.com
  └─ query: nothing found           ✗

_validation-persist.dept.example.com
  └─ query: nothing found           ✗

_validation-persist.example.com
  └─ record: policy=wildcard        ✓
```

</div>
</div>

<div class="ask">

**Proposal:** New `adn` response field directs the CA to the ancestor record.

</div>

---

<!-- _class: discuss -->

# Proposed: `adn` Response Field

RFC 8555 §7.5.1 extension point — future specs may define additional response fields

```json
POST /chall/PAniVnsZcis/dns-persist-01  →  payload: { "adn": "example.com" }
```

- CA confirms `example.com` is a valid Authorization Domain Name for `server.dept.example.com`
- CA looks up `_validation-persist.example.com` → finds `policy=wildcard` → validates

<div class="ask">

**Open questions:**
1. **Depth limits:** Restrict label pruning? *(BRs allow pruning to base domain)*
2. **Scope:** Same draft, targeting -02?

</div>

---

<!-- _class: discuss -->

# TTL as Validation Ceiling (Issue #42)

| Ceiling | Controlled by | Proposal |
|---------|---------------|----------|
| CA Validation Reuse Period | BRs / root programs | ✓ **Keep** |
| `persistUntil` timestamp | Domain owner (explicit) | ✓ **Keep** |
| DNS TXT Record TTL | DNS cache config | ✗ **Remove** |

## Why remove?

- **Layer confusion** — TTL governs DNS caching, not CA policy; resolvers return *remaining* cache time, not authoritative TTL
- **Already bypassed** — Let's Encrypt already caps TTL to 1 min
- **Operational breakage** — default TTLs of 30–60s silently block issuance
- **Author correction** — We wrote this MUST; mailing list analysis changed our position

<div class="ask">

**Proposal:** Remove TTL ceiling (currently a §9.5 MUST) in -02.

</div>

---

<!-- _class: discuss -->

# IP Validation via Reverse DNS (Issue #32)

## Context

- **Proposed new scope** — not yet in draft text
- CA/BF ballot SC-91 **passed** (Nov 2025) — new BR DCV method based on dns-persist for IP addresses via `in-addr.arpa` / `ip6.arpa`
- Uses **`_ip-validation-persist`** label (distinct from `_validation-persist`)
- Same mechanism, different validation target and IANA registration
- **Google Trust Services** supports same-draft approach

<div class="ask">

**Question:** Same draft or companion document?

| Option | Pro | Con |
|--------|-----|-----|
| **Same draft** | Less overhead, same mechanism | WGLC blocked if IP text not ready |
| **Companion** | Independent timeline, cleaner IANA | Extra coordination overhead |

</div>

---

# Quick Items

## Client-Side `persistUntil` Check (Issue #38)

- Should clients check `persistUntil` before responding to a challenge?
- **Concern:** Split-horizon DNS — enterprise clients may not see the record
- *Trade-off: early detection of stale records vs. silent failure behind split-horizon DNS*

## Error Types (Issue #39)

- Deferred until `adn` failure modes are understood

---

# Path Forward

## Priorities for -02

1. Resolve TTL question — remove or keep ceiling (#42)
2. Spec `adn` field for subdomain validation (#33)
3. Decide IP validation scope — same draft or companion (#32)
4. Editorial cleanup from review findings

## Timeline

- **-01:** Editorial fixes → **-02:** Subdomain + TTL
- **WGLC** at -02; slips to -03 if IP validation is same-draft

## Implementation

- **Pebble** testing CA — merged; **lego** client — merged
- **Let's Encrypt** — [implementing](https://letsencrypt.org/2026/02/18/dns-persist-01); **Fastly** — committed; **ATS** — assessing

---

<!-- _class: lead -->

# Questions & Discussion

**Thank you!**

**Contact:**
- Mailing list: acme@ietf.org
- GitHub: https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist
- Draft: https://datatracker.ietf.org/doc/draft-ietf-acme-dns-persist/
