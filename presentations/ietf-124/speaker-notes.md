# Speaker Notes: draft-ietf-acme-dns-persist

## Slide 1: Title & Status (30 seconds)

- Adopted as draft-ietf-acme-dns-persist-00 (Oct 16)
- Authors: Heurich (Fastly), Birge-Lee (Crosslayer), Slaughter (Amazon)
- Thank WG for adoption

## Slide 2: Technical Recap (1 minute)

Persistent DNS TXT record for ACME validation

Format: `_validation-persist.example.com IN TXT "ca.example; accounturi=https://ca.example/acct/123"`

Features:
- Reusable across issuances
- Account binding via accounturi
- Multi-CA support
- Optional expiration (persistUntil)
- policy=wildcard covers subdomains

## Slide 3: Problem Space (1 minute)

Existing challenges:
- http-01/tls-alpn-01: Port 80 required, geo-blocking prevents validation, no wildcards
- dns-01: Propagation delays, API key risks
- dns-account-01 (draft-ietf-acme-dns-account-challenge): Uses account-scoped labels (_<hash>._acme-challenge) to allow multiple independent validations; addresses multi-region/multi-cloud needs but still requires provisioning for each validation (not persistent)
- "Magic CNAMEs": Single point of failure, cache poisoning, BGP hijacking

## Slide 4: Why Standardize? (30 seconds)

CAs could deploy via pre-validation without protocol changes, but this bypasses ACME's challenge selection.

Standardization enables proper protocol integration.

Key: ACME account URLs provide durable, cryptographic binding

## Slide 5: Industry Momentum (1 minute)

CA/Browser Forum: Ballot SC088v3 PASSED Oct 9
- 26 CAs voted YES (including Fastly, Amazon, DigiCert, GlobalSign, GoDaddy)
- 3 consumers voted YES (Google, Mozilla, Cisco)
- IP Rights Review through Nov 8

Implementations:
- Committed: Let's Encrypt (2026)
- Expected: Fastly/Certainly, Amazon Trust Services
- PoC: Pebble server, eggsampler client

## Slide 6: WG Concerns (2 minutes)

Security trade-offs:
- Freshness vs. operational simplification
- Account key = long-lived credential
- Key compromise enables immediate issuance
- Privacy: accounturi in public DNS

DNSSEC validation:
- Draft: SHOULD validate
- Alternative: MUST validate
- Trade-off: Security vs. private PKI flexibility
- Related: draft-ietf-dnsop-domain-verification-techniques

Validation reuse = shortest of:
- DNS TTL (MUST respect)
- persistUntil parameter
- CA policy (398 days → 10 days by 2029)

## Slide 7: Changes in -00 (1 minute)

- Pre-validation optimization: CA checks existing records, skips challenge if valid
- Enhanced security considerations
- Long TXT record guidance (>255 chars)
- Error handling: `malformed` vs. `unauthorized`

## Slide 8: WG Input Part 1 (1 minute)

Acknowledge concerns: Use cases, trust relationships, existing validation

Questions:
1. DNSSEC requirement level? SHOULD vs. MUST
2. Protocol validation caps beyond persistUntil?
3. Additional security considerations?

## Slide 9: WG Input Part 2 (1 minute)

4. Timeline given industry momentum?
5. AccountURI flexibility (PR #30):
   - Multiple URIs per account?
   - Pro: Privacy, access control
   - Con: CA cannot predict client's choice
   - Most deployments pre-provision records
   - Trade-off: Privacy vs. simplicity

## Slide 10: Next Steps (30 seconds)

- Incorporate WG feedback
- Expand security considerations
- Target for WGLC?
- Coordinate with CA/BF timeline

## Slide 11: Discussion (remaining time)

Questions and feedback welcome
