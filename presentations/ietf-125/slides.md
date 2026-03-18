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

- **PR #35 — accounturi decoupled from challenge object** — DNS record is self-contained; enables pre-provisioning, JIT validation, many-to-one URI model
- **PR #36 — Case-handling rules** explicit for all parameters
- **PR #41 — Renamed** "Authorization Domain Name" → "Validation Domain Name" (BR term conflict)

## Implementation & Status

- **Pebble** testing CA — dns-persist-01 [merged](https://github.com/letsencrypt/pebble/pull/536); **lego** client — [merged](https://github.com/go-acme/lego/pull/2871)
- **cert-manager** — [feature request](https://github.com/cert-manager/cert-manager/issues/8373) open, [WIP implementation](https://github.com/sheurich/cert-manager/pull/2)
- -01 publication pending editorial fixes; five open issues, all on today's agenda

---

<!-- _class: discuss -->

# Subdomain Validation Gap (Issue #33)

<div class="columns">
<div>

### The problem

Client has `_validation-persist` at `example.com` with `policy=wildcard`

Wants a cert for `server.dept.example.com`

**No mechanism for the client to specify the parent domain**

Without guidance, the CA either:
- Queries each ancestor *(expensive, ambiguous, MPIC amplification)*
- Checks only the FQDN → **fails**

</div>
<div>

### DNS lookup today

```text
example.com
  └─ _validation-persist   ← record
       policy=wildcard

dept.example.com           ← no record
  │
server.dept.example.com    ← cert request
                              CA looks here
                              → nothing found
```

</div>
</div>

---

<!-- _class: discuss -->

# Proposed: `adn` Response Field

RFC 8555 §7.5.1 extension point — "future specifications might define [response fields]"

```json
POST /chall/PAniVnsZcis/dns-persist-01
payload: { "adn": "example.com" }
```

- CA confirms `example.com` is a proper suffix of `server.dept.example.com`
- CA looks up `_validation-persist.example.com` → finds `policy=wildcard` → validates

<div class="ask">

**Open questions:**
1. **Depth limits:** Restrict label pruning? *(BRs allow pruning to base domain)*
2. **In this draft or separate?** *(We prefer this draft — same mechanism, ships in -02)*

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
- **Already bypassed** — Let's Encrypt caps resolved TTL to 1 min *(Ilari Liusvaara, Oct 2025)*
- **Operational breakage** — default TTLs of 30–60s would block issuance unintentionally

<div class="ask">

**Question:** Does any implementation use TTL to force revalidation? *(Only 2 have weighed in — seeking input)*

</div>

---

<!-- _class: discuss -->

# IP Validation via Reverse DNS (Issue #32)

## Context

- CA/BF ballot SC-091 proposes dns-persist for IP addresses via `in-addr.arpa` / `ip6.arpa`
- Uses **`_ip-validation-persist`** label (distinct from `_validation-persist`)
- Same mechanism and parameters — different validation target and IANA registration
- **Google Trust Services** supports same-draft approach *(Frolov, Dec 2025)*

<div class="ask">

**Question:** Same draft or companion document?

| Option | Pro | Con |
|--------|-----|-----|
| **Same draft** | Less overhead, same mechanism | Depends on CA/BF ballot timeline |
| **Companion** | Independent timeline, cleaner IANA | More documents to track |

</div>

---

# Quick Items

## Error Types (Issue #39)

- Three failure modes all return generic `unauthorized` — clients cannot distinguish
- No ACME extension has registered error types yet — **precedent question**
- **Suggestion:** Design the error space after #33 settles (new failure modes expected)
- *Is this blocking other work?*

## Client-Side `persistUntil` Check (Issue #38)

- Should clients check `persistUntil` before responding to a challenge?
- **Concern:** Split-horizon DNS — enterprise clients may not see the record
- *Useful SHOULD guidance, or more confusing than helpful?*

---

# Path Forward

## Priorities for -02

1. Resolve TTL question — remove or keep ceiling (#42)
2. Spec `adn` field for subdomain validation (#33)
3. Decide IP validation scope — same draft or companion (#32)
4. Editorial cleanup from review findings

## Timeline

- **-01:** Publish after editorial fixes *(imminent)* → **-02:** After subdomain + TTL resolution
- **WGLC target:** After -02 or -03

## Implementation

- **Pebble** testing CA — merged; **lego** client — merged
- **Let's Encrypt** — [implementing](https://letsencrypt.org/2026/02/18/dns-persist-01): staging late Q1, production Q2 2026
- **Fastly/Certainly** — committed for 2026; **Amazon Trust Services** — assessing

---

<!-- _class: lead -->

# Questions & Discussion

**Thank you!**

**Contact:**
- Mailing list: acme@ietf.org
- GitHub: https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist
- Draft: https://datatracker.ietf.org/doc/draft-ietf-acme-dns-persist/
