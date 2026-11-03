<!-- markdownlint-disable MD013 MD060 -->

# Speaker Notes: draft-ietf-acme-dns-persist — IETF 127

**Total slot: 10–15 minutes.** Speak for at most 8 minutes on the default route (or 6 minutes on the compressed route) and leave the remainder for WG discussion and the WGLC readiness check.

## Timing Summary

| Slide | Topic | Time | Cumulative |
|-------|-------|------|------------|
| 1 | Title | 0:15 | 0:15 |
| 2 | Since IETF 126 (`-02` + merged `#74`, `#75`, `#77`–`#83`) (default: short) | 0:45 | 1:00 |
| 3 | Subdomain validation via `ancestorDomain` (`#33` / PR `#84`) (protect) | 1:45 | 2:45 |
| 4 | IP validation via `_ip-validation-persist` (`#32` / PR `#85`) (protect) | 1:30 | 4:15 |
| 5 | `persistExpired` error type (`#39` / PR `#86`) | 0:50 | 5:05 |
| 6 | Combining with certificate-key attestation (`#76` / PR `#87`) | 0:50 | 5:55 |
| 7 | Mailing-list implementation feedback (URI scheme & inner thumbprint) | 1:00 | 6:55 |
| 8 | Implementation status, CABF alignment & WGLC path | 0:45 | 7:40 |
| 9 | Questions & Discussion | 2:20+ | 10:00+ |

## Time Discipline

- **Protected slides:** Slides **3** (`ancestorDomain`, `#33`), **4** (`_ip-validation-persist`, `#32`), and **8** (WGLC path) are protected—Let's Encrypt and Google Trust Services specifically requested `#33` and `#32`, and Slide 8 sets up the WGLC ask.
- **Default `[IF SHORT]` slide:** Use the `[IF SHORT]` script on **Slide 2** by default because `-02` was already reviewed on the list and `#77`–`#83` are already merged into `main`.
- **Hard rule:** If Slide 4 (`_ip-validation-persist`) is not on screen by **3:30 cumulative**, switch Slides 5, 6, and 7 to their `[IF SHORT]` versions so you land on Slide 8 by 6:00 and preserve at least 3–4 minutes for the microphone queue.

---

## Slide 1: Title (0:15)

"Good morning—Shiloh Heurich, presenting `draft-ietf-acme-dns-persist` with Henry Birge-Lee and Michael Slaughter. Fourth WG presentation. Today's goal is to walk through the four remaining feature PRs (`#84` through `#87`), close out two implementation questions from the mailing list, and check readiness for Working Group Last Call once `-03` is published."

For newcomers (reinforce the subtitle briefly):
"In one sentence: `dns-persist-01` lets a domain or IP holder publish a long-lived DNS TXT record carrying a hashed `accounturi` and an optional `persistUntil` timestamp so a specific ACME account at a specific CA can validate and renew certificates without modifying DNS on every issuance."

**Transition:** "Quick recap of what landed since Vienna."

---

## Slide 2: Since IETF 126 (Vienna) (0:45 default short / 1:15 full)

**[DEFAULT — IF SHORT (0:45)]:**
"In September we published `-02`, which landed the mandatory hashed `accounturi` construction, the domain-omitted star opt-out, lifetime witnessed-key retention across account key rollover overridden immediately by account deactivation, the `persistUntil` clamp on authorization `expires`, and directory `issuerDomainNames`.

Since `-02`, we merged nine clarification PRs into `main` and incorporated Jacob Hoffman-Andrews's normalization feedback—operating hash canonicalization strictly on validated US-ASCII DNS names (with internationalized labels already in IDNA2008 A-label form per RFC 8555 §§5 and 7.1.4) via ASCII lowercasing and trailing-dot removal, making `issuer-domain-name` matching explicitly ASCII case-insensitive, making parameter names ASCII case-insensitive with duplicate rejection in PR `#80`, constraining `accountHashPrefix` to fit RFC 8659's TXT `value` grammar in PR `#81`, and clarifying validation flows and JIT rate-limiting in PRs `#82` and `#83`."

**[FULL SCRIPT (1:15, only if given a 15-minute slot)]:**
"On the left: `-02` published in September with all the core decisions from Vienna and the August list threads. The hashed `accounturi` is the sole form, binding the domain length, domain name, 43-octet RFC 7638 JWK thumbprint, and account URL. The domain-omitted `*` opt-out is client-MAY, CA-MUST. For account key rollover, CAs retain witnessed account-key thumbprints for the lifetime of the account so pre-provisioned and backup-CA records survive routine key rotations, while account deactivation immediately overrides retention. And `persistUntil` clamps the resulting ACME authorization's `expires` field.

On the right: since `-02` we merged PRs `#74`, `#75`, and `#77` through `#83` into `main`. PR `#79` specifies IDNA2008 `ToASCII` conversion without transitional processing before hashing so `faß.example` and `fass.example` never collide. PR `#80` makes parameter names (`accounturi`, `policy`, `persistUntil`) ASCII case-insensitive while keeping parameter values case-sensitive and rejecting duplicate recognized parameters. PR `#81` constrains `accountHashPrefix` to printable US-ASCII excluding semicolons and whitespace so constructed URIs always fit RFC 8659 TXT syntax. And PRs `#75`, `#82`, and `#83` clarify per-attempt computation bounds, delegated account-proof ordering, and JIT rate limiting."

**Transition:** "With those merged into `main`, four open feature PRs remain—starting with subdomain validation in `#33`."

---

## Slide 3: Subdomain Validation via `ancestorDomain` (`#33` / PR `#84`) (1:45, protect)

"At IETF 125 we discussed how a CA knows which parent domain holds a `policy=wildcard` record when validating a subdomain like `server.dept.example.com`, without doing blind DNS tree walks. PR `#84` resolves Issue `#33`.

First, we aligned the field name with RFC 9444's camelCase convention: `ancestorDomain`, sent only in the client's challenge-response `POST` payload. The server's Challenge Object resource is unchanged.

To avoid any ambiguity with wildcards, PR `#84` separates three names: the requested certificate identifier `F`, the authorization object's `identifier.value` `A`—where RFC 8555 Section 7.1.3 already strips the `*.` prefix—and the record domain `R`.
- When the client sends the default empty JSON object `{}`, `R` equals `A`. Notice that for a base-level wildcard `*.example.com`, `A` is already `example.com`, so the client just sends `{}`.
- For a deeper wildcard `*.dept.example.com` where `A` is `dept.example.com`, or a subdomain `server.dept.example.com`, the client sends `{\"ancestorDomain\": \"example.com\"}`.

On the right are the six normative guardrails:
1. Every CA implementing `dns-persist-01` MUST recognize `ancestorDomain` and never silently ignore it to query `A`.
2. Both `R` and the constructed `_validation-persist.<R>` must fit the 63-octet label and 253-octet total DNS length limits, and `R` must be a strict whole-label proper suffix of normalized `A`.
3. Public suffixes are rejected with `unauthorized` unless expressly permitted by CA policy—such as a dot-brand TLD or private PKI.
4. A qualifying record at `_validation-persist.<R>` MUST include `policy=wildcard`, and the `accounturi` digest is computed over `R` (or `*`), not the child name `A`.
5. The first accepted challenge-response POST locks `R` for that validation attempt; conflicting retries are rejected with HTTP `malformed`.
6. Finally, PR `#84` specifies clean coexistence with RFC 9444 (`subdomainAuthAllowed`): `ancestorDomain` in a challenge response satisfies only that single authorization for `A` without mutating `identifier.value` or setting `subdomainAuthAllowed`, whereas if a CA validates an RFC 9444 ancestor authorization (`subdomainAuthAllowed: true`) via `dns-persist-01`, `policy=wildcard` is required."

**[IF SHORT]:** "PR `#84` adds a request-only `ancestorDomain` field to the `POST /chall` payload. CAs must recognize it, verify `R` is a non-public-suffix proper ancestor within the 253-octet VDN limit, require `policy=wildcard` at `_validation-persist.<R>`, and hash `R` (or `*`). It coexists cleanly with RFC 9444 without altering the authorization object." (45 sec)

**Transition:** "Second feature PR: IP address validation in `#32`."

---

## Slide 4: IP Address Validation via `_ip-validation-persist` (`#32` / PR `#85`) (1:30, protect)

"PR `#85` resolves Issue `#32` by defining an additive profile for ACME `ip` identifiers under RFC 8738 using the reverse DNS tree.

For IPv4, the normalized Reverse Zone Domain Name `R` is the four reversed decimal octets under `.in-addr.arpa`. For IPv6, `R` is all 32 reversed lowercase hexadecimal nibbles under `.ip6.arpa`—always 72 octets.

We prepend a distinct underscore label, `_ip-validation-persist.<R>`, and register both `_ip-validation-persist` in the IANA Underscored DNS Node Names registry and `dns-persist-01` for the `ip` identifier type. CAs query `_ip-validation-persist` only for `ip` authorizations, query `_validation-persist` only for `dns` authorizations, and never follow `PTR` records.

Two important details in PR `#85`, including updates from Max Hearnden's review on GitHub:
- First, for the digest `domain_name` input, both the publisher and CA use the normalized lowercase `.in-addr.arpa` or `.ip6.arpa` Reverse Zone Domain Name `R` (or `*`) prior to following any CNAME aliases—not the DNS canonical name of a CNAME target. And because a CNAME at `<R>` redirects only `<R>` itself and not child labels beneath it, RFC 2317 classless CNAME delegation of `_ip-validation-persist.<R>` places the CNAME at `_ip-validation-persist.<R>` while still hashing the `.in-addr.arpa` / `.ip6.arpa` name `R`.
- Second, validation scope is strictly the single IP address: `policy=wildcard` is ignored on `_ip-validation-persist` records, `ancestorDomain` is rejected with `malformed`, and Security Considerations call out reverse-zone delegation boundaries, DNS wildcard synthesis in delegated zones, and cleaning up records or bounding `persistUntil` across IP reassignments."

**[IF SHORT]:** "PR `#85` adds IPv4 and IPv6 validation at `_ip-validation-persist.<R>` in `in-addr.arpa` and `ip6.arpa`. Digest input uses the normalized reverse name `R` (or `*`) even across RFC 2317 CNAME delegation, scope is strictly one IP address (`policy=wildcard` ignored, `ancestorDomain` rejected), and both IANA registries are updated." (40 sec)

**Transition:** "Next two feature PRs are smaller and self-contained: `#39` and `#76`."

---

## Slide 5: `persistExpired` Error Type (`#39` / PR `#86`) (0:50)

"PR `#86` addresses Issue `#39` on dedicated error types. Rather than minting separate error codes for every sub-case—which risks creating an oracle across other accounts' or CAs' records at the same domain—PR `#86` registers a single error type, `urn:ietf:params:acme:error:persistExpired`, with a strict two-part rule:

1. A mandatory `MUST NOT` privacy gate: a CA MUST NOT return `persistExpired` if any record at the queried name satisfies validation, or if no record at that name is otherwise fully qualifying for the CA and the live requesting account and failed solely due to an expired `persistUntil` timestamp—regardless of RRset evaluation order.
2. A `SHOULD` diagnostic recommendation: when that eligibility condition holds on a challenge attempt, the CA SHOULD return `persistExpired`, and MAY return `unauthorized` if it enforces a uniform disclosure policy. Just-in-Time validation continues to fall back to `pending`."

**[IF SHORT]:** "PR `#86` registers `persistExpired` with a mandatory `MUST NOT` privacy gate so it only fires when a record belonging to the requesting account and CA failed solely due to `persistUntil`, never leaking another account's or CA's expiry state." (20 sec)

**Transition:** "And PR `#87` covers combining `dns-persist-01` with hardware key attestation."

---

## Slide 6: Combining with Certificate-Key Attestation (`#76` / PR `#87`) (0:50)

"Issue `#76` asked how an enterprise or IoT deployment can require both `dns-persist-01` domain validation and hardware key attestation such as `device-attest-01`.

As Slide 6 notes, ACME challenge arrays within a single authorization object are `OR` alternatives, CAA `validationmethods` lists are `OR` alternatives, and unknown `_validation-persist` parameters must be ignored by CAs that don't implement them.

So PR `#87` specifies the conjunction at the ACME Order policy layer under RFC 8555 Sections 7.1.3 and 7.4, citing `draft-ietf-acme-device-attest` informatively so we create no normative reference hold:
1. A CA enforcing the conjunction for an account or identifier MUST include the attestation authorization on every Order in that scope—covering all wildcard and subordinate names—and MUST NOT allow the client to waive it.
2. At Order finalization, the CA MUST verify that all Order authorizations are `valid` AND that the CSR public key matches the Order's attested public key—even when `dns` validated via JIT or reuse.
3. On renewal with a new CSR key, JIT reuses the `_validation-persist` record while the new Order requires fresh attestation for the new key."

**[IF SHORT]:** "PR `#87` specifies `dns-persist-01` plus certificate-key attestation as a non-waivable Order-level authorization conjunction with CSR public-key binding at finalization, citing `device-attest` informatively." (20 sec)

**Transition:** "Before wrapping up, two questions from Seo Suchan's October 13 Pebble implementation report on the list."

---

## Slide 7: Mailing-List Implementation Feedback (1:00)

"Seo updated his Pebble fork to `-02` and raised two questions on the list that Henry asked us to bring to the group:

First: should we keep the `https://` `accountHashPrefix` construction or mint a bespoke `accounthash:` URI scheme? The authors recommend keeping `https://` `accountHashPrefix`:
- RFC 8657 Section 5.4 explicitly recommends satisfying `accounturi` uniqueness with URIs that include an authority component, which `https://` provides and a bare `accounthash:sha-256/...` lacks.
- A custom scheme saves zero ACME directory lookups, because pre-provisioning already uses the directory's `issuerDomainNames` array (and in delegated DNS flows, the ACME client passes the computed URI or prefix to the DNS publisher).
- Keeping `https://` avoids adding a BCP 35 IANA URI-scheme registration dependency and stays aligned with the draft CA/B Forum ballot text. And the one real trade-off—CA domain rebranding—is already handled by the Section 4.5 dual-acceptance migration rule.

Second: why is the inner `key` input fixed to the 43-octet RFC 7638 SHA-256 JWK thumbprint while the outer `<hash-alg>` is agile?
- Because RFC 8555 Section 8.1 hardcodes the 43-octet SHA-256 JWK thumbprint for ACME key authorizations, every client and CA already computes it.
- Even more importantly for CAs: keeping `key` fixed to that 43-octet string means a CA's historical key-retention table stores **one** 43-byte thumbprint per key regardless of which outer `<hash-alg>` the client uses, and keeps the field boundary before `account_URL` fixed-length."

**[IF SHORT]:** "On the October 13 list thread: we recommend keeping `https://` `accountHashPrefix` (matches RFC 8657 §5.4 authority guidance, saves a BCP 35 URI-scheme registration, and a custom scheme wouldn't remove the directory fetch for `issuerDomainNames`) and keeping the inner JWK thumbprint fixed at RFC 7638 SHA-256 (matching RFC 8555 §8.1 and keeping CA historical key storage to one thumbprint per key)." (25 sec)

**Transition:** "Here is where that leaves the draft and our WGLC timeline."

---

## Slide 8: Implementation Status & Path to WGLC (0:45, protect)

"With PRs `#74`, `#75`, and `#77` through `#83` merged into `main`, and PRs `#84` through `#87` open and reviewed for Issues `#33`, `#32`, `#39`, and `#76`, every open issue in the repository is addressed.

On the implementation side: Pebble and lego have support merged, Seo's Pebble fork exercised `-02`, cert-manager design and prototype work is active, and CAs including Let's Encrypt, Fastly, Amazon Trust Services, and Google Trust Services have been actively engaged. On the CA/B Forum side, Michael's draft Servercert ballot defines the dedicated ACME `dns-persist-01` validation method aligned with `-03`.

Our plan is to merge `#84` through `#87` following today's feedback, submit `draft-ietf-acme-dns-persist-03`, and ask the chairs to initiate Working Group Last Call."

**Transition:** "With that, I'll open the floor for questions on `#84` through `#87`, the URI scheme, or WGLC readiness."

---

## Slide 9: Questions & Discussion (Remaining Time)

Open the floor. Standing asks for the room and chairs:
1. "Any objections to merging PRs `#84` (`ancestorDomain`), `#85` (`_ip-validation-persist`), `#86` (`persistExpired`), and `#87` (attestation conjunction) into `-03`?"
2. "Any objections to keeping `https://` `accountHashPrefix` and the fixed RFC 7638 SHA-256 inner thumbprint?"
3. "Chairs: once `-03` is posted with `#84`–`#87` merged, we would like to request Working Group Last Call."

---

## Likely Questions

### 1. Why is `ancestorDomain` only in the challenge response (`POST /chall`) and not echoed in the server's Challenge Object? (#33 / PR #84)

RFC 8555 Section 7.5.1 explicitly provides the client's challenge-response `POST` payload for per-attempt client parameters (the same pattern used by `attObj` in `draft-ietf-acme-device-attest`), while keeping the server's Challenge Object schema (RFC 8555 Section 8) static. In asynchronous ACME servers like Boulder, challenge sub-resources are projections of a compact authorization row rather than mutable per-POST JSON documents. To ensure safety, Item 1 of Section 6.2 makes CA recognition of `ancestorDomain` a strict `MUST` (forbidding silent fallback to `_validation-persist.<A>`), and Section 4 specifies that the first accepted `POST` fixes `R` for that validation attempt.

### 2. For a base-level wildcard `*.example.com`, does the client send `"ancestorDomain": "example.com"` or `{}`? (#33 / PR #84)

The client sends `{}`. Under RFC 8555 Section 7.1.3, the ACME server strips the `*.` prefix when creating the authorization object for `*.example.com`, so the authorization's `identifier.value` (`A`) is already `example.com` (with `wildcard: true`). Because a domain is not a proper ancestor of itself (`R` must have strictly fewer labels than `A`), sending `"ancestorDomain": "example.com"` when `A = example.com` is rejected with `malformed`. You only send `"ancestorDomain": "example.com"` when `A` is a deeper name—such as `*.dept.example.com` (`A = dept.example.com`) or `server.dept.example.com`.

### 3. If a client sends an invalid or public-suffix `ancestorDomain`, does that transition the challenge to `invalid` or return an HTTP error while leaving the challenge `pending`? (#33 / PR #84)

PR `#84` specifies this explicitly: syntax, non-ancestor, and public-suffix/policy rejections on the `POST` request (Items 2–4) are HTTP request errors (RFC 8555 Section 7.5.1) that leave the challenge in the `pending` state, whereas a DNS verification failure at `_validation-persist.<R>` (Item 5, such as missing `policy=wildcard` or non-matching `accounturi`) is a validation failure that transitions the challenge to `invalid` (RFC 8555 Section 7.1.6).

### 4. How does RFC 2317 classless `in-addr.arpa` CNAME delegation work with `_ip-validation-persist`? (#32 / PR #85)

In DNS, a `CNAME` record at `<R>` (for example, `1.2.0.192.in-addr.arpa`) redirects queries for `<R>` itself (such as `PTR`), but does **not** redirect queries for child labels beneath `<R>` like `_ip-validation-persist.1.2.0.192.in-addr.arpa` (which would require a `DNAME` at the parent or an explicit `CNAME` at `_ip-validation-persist.<R>`). PR `#85` states this explicitly: delegating `_ip-validation-persist.<R>` via `CNAME` requires an alias at `_ip-validation-persist.<R>` itself, and both the DNS publisher and the CA still compute `domain_name` in the hash over the normalized `.in-addr.arpa` or `.ip6.arpa` Reverse Zone Domain Name `R` (or `*`), never the `CNAME` target.

### 5. Why did you change `"canonical Reverse Zone Domain Name"` to `"normalized Reverse Zone Domain Name"` in PR #85?

Max Hearnden pointed out in his review of PR `#85` that `"canonical name"` is a term of art in DNS (RFC 1034 Section 3.6.2 / RFC 1035 Section 3.3.1) meaning the target of a `CNAME` alias—which is the exact opposite of `R` when `_ip-validation-persist.<R>` is aliased via `CNAME`. Using `"normalized Reverse Zone Domain Name"` (and noting that RFC 1035 and RFC 3596 write `IN-ADDR.ARPA` and `IP6.ARPA` in uppercase while we normalize to lowercase) eliminates that ambiguity.

### 6. Can a reverse-zone operator use a DNS wildcard record (`*.2.0.192.in-addr.arpa`) with `domain_name=*` to answer `_ip-validation-persist` queries for multiple IPs? (#32 / PR #85)

Yes—and PR `#85` documents this explicitly in Security Considerations (`{{#ip-validation-risks}}`). Ignoring `policy=wildcard` ensures that a single ACME authorization covers only one IP address, and separate node names (`_ip-validation-persist` vs. `_validation-persist`) and typed caches keep `ip` and `dns` validations separate. However, because the reverse-zone operator controls all nodes in its zone, it can synthesize DNS wildcard responses across addresses in that zone if it uses the `domain_name=*` opt-out. That is inherent to reverse-DNS delegation, which is why CAs only enable `dns-persist-01` for `ip` identifiers where permitted by their CP/CPS and trust-program rules.

### 7. Why not add a `require-attestation` parameter to the `_validation-persist` DNS TXT record instead of using CA Order policy? (#76 / PR #87)

Because Section 4.1 Item 1 requires CAs to ignore unrecognized parameters in `_validation-persist` records (so future optional extensions don't break legacy CAs). A DNS parameter that an unaware CA silently ignores cannot fail closed across CAs. Instead, a domain owner that requires certificate-key attestation scopes its `_validation-persist` record (via `issuer-domain-name` and `accounturi`, reinforced by CAA) to a CA and account whose Order policy non-waivably requires the attestation authorization on every Order and verifies the CSR public key at finalization.

### 8. What about the recomputation DoS discussion on Issue #69 (retained historical keys)?

Merged PR `#75` and `#83` note in Section 10.3 (`{{#ca-implementation-guidelines}}`) that while current-key-first evaluation makes normal validations `O(1)` and failed-validation rate limits bound failed attempts, an admitted attempt that succeeds against an older retained thumbprint can still scan the account's retained-key history. Because the account holder controls how many times it calls `keyChange`, CAs bound worst-case per-attempt work by rate-limiting `keyChange` or admitted attempts per account (and computing candidate hashes once per retained key across an RRset), without reintroducing a 50-key eviction cap that would break dormant backup-CA records.
