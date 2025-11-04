# Speaker Notes: draft-ietf-acme-dns-persist

## Slide 1: Title & Status (30 seconds)

- WG adopted draft-ietf-acme-dns-persist-00 on October 16
- Authors: Heurich (Fastly), Birge-Lee (Crosslayer), Slaughter (Amazon)
- Thank WG for adoption

## Slide 2: Dual-Track Progress (1 minute)

WG adopted Oct 16: draft-sheurich-acme-dns-persist → draft-ietf-acme-dns-persist-00

Industry validates in parallel:
- CA/Browser Forum passed Ballot SC088v3 on October 9
- 26 CAs voted YES (Fastly, Amazon, DigiCert, GlobalSign, GoDaddy, others)
- 3 consumers voted YES (Google, Mozilla, Cisco)
- IP Rights Review completes November 8

Implementations:
- Fastly/Certainly and Let's Encrypt committed (Boulder integration 2026)
- Amazon Trust Services assessing
- Pebble server fork and eggsampler client fork demonstrate proof of concept

Industry validates the approach while IETF standardizes.

## Slide 3: How dns-persist-01 Works (1 minute)

The mechanics:
- Record lives at _validation-persist subdomain (industry standard label)
- Contains CA identifier (issuer domain) + account binding (accounturi)
- accounturi cryptographically binds to ACME account - the key innovation
- policy=wildcard explicitly opts into broader scope (security trade-off)
- persistUntil lets domain owners control expiration
- Each CA searches for its issuer domain in records (multi-CA support)
- Unlike dns-01's transient challenges, these records persist

## Slide 4: Why We Need This (1 minute)

Real-world pain points:
- Port 80/443 requirements exclude IoT devices and internal services
- Countries and regions increasingly block validation servers
- DNS API credentials on every server multiply compromise risk
- dns-account-01 helps multi-region but requires per-certificate provisioning
- "Magic CNAMEs" (acme-dns.io) create systemic risk
- 10-day validation arrives in 2029 - we need efficiency

Key insight: Persistent records solve automation at scale

## Slide 5: Why Standardize? (30 seconds)

CAs could check persistent DNS records outside ACME, but this bypasses the protocol's challenge negotiation where clients choose validation methods.

Standardization enables proper protocol integration.

Key: ACME account URLs provide durable, cryptographic binding

## Slide 6: Design Principles (1 minute)

Design choices that solve the problems:

Account binding:
- ACME accounturi provides durable, cryptographic identity
- Leverages existing ACME infrastructure
- Requires no new trust anchors

Multi-CA:
- Per-issuer TXT records contain issuer domain
- CAs operate independently
- Clients control which CAs validate

Flexibility within constraints:
- persistUntil expiration (optional)
- policy=wildcard enables subdomain coverage
- Unknown parameters ignored for forward compatibility

Security constraints:
- Must respect DNS TTL
- CA/BF certificate lifetimes bound validation (398d → 10d by 2029)
- Just-in-time validation optimizes performance

These principles bridge motivation with open questions.

## Slide 7: Active WG Discussions (2 minutes)

The philosophical tensions:

**Freshness vs. operations:**
- Traditional ACME requires fresh challenges every time
- This draft persists trust with the account key
- Account key compromise has lasting impact
- accounturi reveals business relationships (privacy concern)

**DNSSEC debate:**
- SHOULD allows private PKI flexibility
- MUST matches CA/BF security expectations
- Balance: prescribe requirements or let deployers decide?

**Validation reuse math:**
- Shortest-wins principle in action
- 398 days shrinking to 10 days changes everything
- DNS TTL respect remains non-negotiable

## Slide 8: Evolution to WG Draft (1 minute)

What WG input changed:
- **Just-in-time validation:** Major efficiency win - like checking for a season pass before selling a ticket
- **Security section growth:** We take concerns seriously
- **Long TXT handling:** DNS has 255-byte string limits - a practical issue many miss
- **Error codes:** malformed (syntax) vs unauthorized (wrong account) help debugging
- Implementer feedback from Pebble testing drove these changes

## Slide 9: Seeking WG Input (1.5 minutes)

Each question and its implications:

1. **DNSSEC:** Beyond security theater - affects private PKI deployments
2. **Security gaps:** What attacks have we missed?
3. **Timeline pressure:** CA/BF moves ahead - IETF must guide, not follow
4. **PR #30 (accounturi flexibility):**
   - Multiple URIs increase privacy but add complexity
   - CAs cannot predict which URI clients will use
   - Does privacy justify operational complexity?

Your input shapes industry practice for years

## Slide 10: Path Forward (30 seconds)

Timeline:
- Montreal feedback → -01 draft (December)
- PR #30 blocks progress
- WGLC realistic Q1 2026 if we address concerns
- CA/BF implements in 2026 regardless
- We must finalize before deployment

Key message: We move deliberately, but industry won't wait

## Slide 11: Discussion (remaining time)

Questions and feedback welcome
