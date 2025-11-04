# Speaker Notes: draft-ietf-acme-dns-persist

## Slide 1: Title & Status (30 seconds)

- Adopted as draft-ietf-acme-dns-persist-00 (Oct 16)
- Authors: Heurich (Fastly), Birge-Lee (Crosslayer), Slaughter (Amazon)
- Thank WG for adoption

## Slide 2: Dual-Track Progress (1 minute)

WG adopted Oct 16: draft-sheurich-acme-dns-persist → draft-ietf-acme-dns-persist-00

Industry validation parallel track:
- CA/Browser Forum Ballot SC088v3 PASSED Oct 9
- 26 CAs voted YES (Fastly, Amazon, DigiCert, GlobalSign, GoDaddy, others)
- 3 consumers voted YES (Google, Mozilla, Cisco)
- IP Rights Review completes Nov 8

Implementations:
- Committed: Fastly/Certainly, Let's Encrypt (2026 Boulder integration)
- Assessing: Amazon Trust Services
- PoC: Pebble server fork, eggsampler client fork

This shows real-world validation while IETF process continues.

## Slide 3: How dns-persist-01 Works (1 minute)

Explain the mechanics:
- Record lives at _validation-persist subdomain (industry standard label)
- Contains CA identifier (issuer domain) + account binding (accounturi)
- accounturi is the key innovation - cryptographically ties to ACME account
- policy=wildcard is explicit opt-in for broader scope (security trade-off)
- persistUntil gives domain owners control over expiration
- Multi-CA support: Each CA looks for their own issuer domain in records
- This is fundamentally different from dns-01's transient challenges

## Slide 4: Why We Need This (1 minute)

Real-world pain points driving this work:
- Port 80/443 requirements exclude IoT devices, internal services
- Geo-blocking is increasing - validation servers blocked by country/region
- DNS API credentials on every server = compromise risk multiplier
- dns-account-01 helps multi-region but still requires per-cert provisioning
- "Magic CNAMEs" (acme-dns.io) work but create systemic risk
- Industry seeing 10-day validation coming (2029) - need efficiency

Key insight: Persistent records solve automation at scale

## Slide 5: Why Standardize? (30 seconds)

CAs could check persistent DNS records outside ACME, but this would bypass the protocol's challenge negotiation where clients choose their validation method.

Standardization enables proper protocol integration.

Key: ACME account URLs provide durable, cryptographic binding

## Slide 6: Design Principles (1 minute)

Key design choices that address the problems:

Account binding:
- ACME accounturi provides durable, cryptographic identity
- Leverages existing ACME infrastructure
- No new trust anchors needed

Multi-CA:
- Per-issuer TXT records (issuer domain in record)
- No coordination between CAs required
- Client controls which CAs can validate

Flexibility within constraints:
- Optional persistUntil expiration
- policy=wildcard for subdomain coverage
- Unknown parameters ignored for forward compatibility

Security constraints:
- MUST respect DNS TTL
- Bounded by CA/BF certificate lifetimes (398d → 10d by 2029)
- Just-in-time validation optimization

This bridges motivation with open questions ahead.

## Slide 7: Active WG Discussions (2 minutes)

Frame the philosophical tensions:

**Freshness vs. operations:**
- Traditional ACME = fresh challenge every time
- This draft = trust persists with the account key
- Account key compromise now has lasting impact
- Privacy consideration: accounturi reveals business relationships

**DNSSEC debate:**
- SHOULD allows private PKI flexibility
- MUST would match CA/BF security expectations
- Balance: How much to prescribe vs. let deployers decide?

**Validation reuse math:**
- Show how shortest-wins principle works
- 398 days shrinking to 10 days changes the equation
- TTL respect is non-negotiable (DNS fundamentals)

## Slide 8: Evolution to WG Draft (1 minute)

Highlight what changed with WG input:
- **Just-in-time validation:** Major efficiency win - explain it like checking for a season pass before selling a ticket
- **Security section growth:** Shows we're taking concerns seriously
- **Long TXT handling:** Practical issue many miss - DNS has 255-byte string limits
- **Error codes:** Help debugging - malformed (syntax) vs unauthorized (wrong account)
- These changes came from implementer feedback (Pebble testing)

## Slide 9: Seeking WG Input (1.5 minutes)

Frame each question with its implications:

1. **DNSSEC:** Not just security theater - affects private PKI deployments
2. **Security gaps:** What attacks haven't we considered?
3. **Timeline pressure:** CA/BF moving ahead - IETF needs to guide vs follow
4. **PR #30 (accounturi flex):**
   - Multiple URIs = privacy win but complexity cost
   - CA can't predict which URI client will use
   - Is the privacy gain worth the operational complexity?

Emphasize: Your input shapes industry practice for years

## Slide 10: Path Forward (30 seconds)

Timeline context:
- Montreal feedback → -01 draft (December)
- Resolve PR #30 is blocking issue
- WGLC realistic Q1 2026 if we address concerns
- CA/BF implementation starts 2026 regardless
- This is our chance to get it right before deployment

Key message: Moving deliberately but industry won't wait forever

## Slide 11: Discussion (remaining time)

Questions and feedback welcome
