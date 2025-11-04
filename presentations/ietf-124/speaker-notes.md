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

## Slide 3: Problem Space (1 minute)

Existing challenges:
- http-01/tls-alpn-01: Port 80 required, geo-blocking prevents validation, no wildcards
- dns-01: Propagation delays, API key risks
- dns-account-01 (draft-ietf-acme-dns-account-challenge): Uses account-scoped labels (_<hash>._acme-challenge) to allow multiple independent validations; addresses multi-region/multi-cloud needs but still requires provisioning for each validation (not persistent)
- "Magic CNAMEs": Single point of failure, cache poisoning, BGP hijacking

## Slide 4: Why Standardize? (30 seconds)

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

Security trade-offs:
- Freshness vs. operational simplification
- Account key = long-lived credential
- Key compromise enables immediate issuance
- Privacy: accounturi in public DNS

DNSSEC validation:
- Draft: SHOULD validate
- Alternative: MUST validate
- Trade-off: Security vs. private PKI flexibility

Validation reuse = shortest of:
- DNS TTL (MUST respect)
- persistUntil parameter
- CA policy (398 days → 10 days by 2029)

## Slide 8: Evolution to WG Draft (1 minute)

Changes from draft-sheurich-acme-dns-persist-00:
- Just-in-time optimization: CA checks existing records, skips challenge if valid
- Enhanced security considerations
- Long TXT record guidance (>255 chars)
- Error handling: `malformed` vs. `unauthorized`

## Slide 9: Seeking WG Input (1.5 minutes)

Acknowledge concerns: Use cases, trust relationships, existing validation

Questions:
1. DNSSEC requirement level? SHOULD vs. MUST
2. Protocol validation caps beyond persistUntil?
3. Additional security considerations?
4. Timeline given industry momentum?
5. AccountURI flexibility (PR #30):
   - Multiple URIs per account?
   - Pro: Privacy, access control
   - Con: CA cannot predict client's choice
   - Trade-off: Privacy vs. simplicity

## Slide 10: Path Forward (30 seconds)

- Incorporate Montreal feedback
- Expand security considerations
- Resolve PR #30 (accounturi flexibility)
- Address use case and trust concerns
- Target WGLC after 1-2 revisions
- Coordinate with CA/Browser Forum

## Slide 11: Discussion (remaining time)

Questions and feedback welcome
