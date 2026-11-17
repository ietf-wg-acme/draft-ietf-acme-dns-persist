<!-- markdownlint-disable MD013 MD060 -->

# Speaker Notes: draft-ietf-acme-dns-persist — IETF 127

**Total slot: 10 minutes** (Wednesday, November 18, 2026, 09:00 PST). Speak for about 7 minutes on the default route (or 5:30 on the compressed route) and leave 3+ minutes for WG discussion and the WGLC readiness check.

## Timing Summary

| Slide | Topic | Time | Cumulative |
|-------|-------|------|------------|
| 1 | Title | 0:15 | 0:15 |
| 2 | Since IETF 126 (`-02` + `-03` hardening & clarifications) (default: short) | 0:55 | 1:10 |
| 3 | Subdomain validation via `ancestorDomain` (protect) | 1:30 | 2:40 |
| 4 | IP validation via `_ip-validation-persist` (protect) | 1:15 | 3:55 |
| 5 | `persistExpired` error type | 0:45 | 4:40 |
| 6 | Combining with certificate-key attestation | 0:45 | 5:25 |
| 7 | Mailing-list implementation feedback (URI scheme & inner thumbprint) | 0:50 | 6:15 |
| 8 | Implementation status, earlier-draft interop, CABF & WGLC path (protect) | 0:45 | 7:00 |
| 9 | Questions & Discussion | 3:00 | 10:00 |

## Time Discipline

- **Protected slides:** Slides **3** (`ancestorDomain`), **4** (`_ip-validation-persist`), and **8** (WGLC path) are protected—Let's Encrypt and Google Trust Services specifically requested subdomain and IP validation, and Slide 8 sets up the WGLC ask.
- **Default `[IF SHORT]` slide:** Use the `[IF SHORT]` script on **Slide 2** by default in the 10-minute slot (~920 spoken words across Slides 1–8).
- **Hard rules:** If Slide 4 (`_ip-validation-persist`) is not on screen by **3:00 cumulative**, switch Slides 5, 6, and 7 to their `[IF SHORT]` versions (~5:30 total). If **6:15 cumulative** arrives before Slide 8, advance directly to Slide 8 so at least 3 minutes remain for the microphone queue.

---

## Slide 1: Title (0:15)

"Good morning—Shiloh Heurich, presenting `draft-ietf-acme-dns-persist-03` with Henry Birge-Lee and Michael Slaughter. Since `-03` posted on Sunday after the cutoff, today I'll walk through the four feature additions, the review hardening, two October list questions, and the path to Working Group Last Call."

---

## Slide 2: Since IETF 126 (Vienna) (0:55 default short / 1:25 full)

**[DEFAULT — IF SHORT (0:55)]:**
"In September, `-02` introduced the mandatory hashed `accounturi`, the star opt-out, lifetime witnessed-key retention overridden by account deactivation, the `persistUntil` clamp, and directory `issuerDomainNames`.

In `-03`, hash canonicalization is ASCII-only on validated A-labels, `issuer-domain-name` matching is exact, and CAs follow CNAMEs at the Validation Domain Name while keeping `domain_name` bound to the pre-alias `R` (or `*`). Clients and delegated publishers MUST verify `accountHashPrefix` uses only RFC 8659 `value` octets and `%` plus RFC 3986 `reserved`/`unreserved` characters—blocking semicolon injection and zone-file escapes—with no fallback if absent or invalid. Recognized parameter tags are ASCII case-insensitive with duplicate recognized tags rejected; `persistUntil` MUST be unsigned ASCII digits up to `2^63 - 1`, valid through `T`, never combining records. CAs MUST NOT reassign account URLs to different accounts, and directory `issuerDomainNames` MUST remain in issued challenges while advertised, with a recommended migration period on removal."

**[FULL SCRIPT (1:25, only if given extra time)]:**
"On the left: `-02` published in September with all the core decisions from Vienna and the August list threads. The hashed `accounturi` is the sole form, binding the domain length, domain name, 43-octet RFC 7638 JWK thumbprint, and account URL. The domain-omitted `*` opt-out is client-MAY, CA-MUST. For account key rollover, CAs retain witnessed account-key thumbprints for the lifetime of the account while any non-`valid` account status (deactivated or revoked) immediately overrides retention. And `persistUntil` clamps the resulting authorization's `expires` field.

On the right: `-03` incorporates list feedback and pre-WGLC review hardening, plus new RFC 6973 Privacy Considerations and RFC 7942 Implementation Status sections. Following Jacob's list feedback, `-03` keeps hash canonicalization ASCII-only on validated A-labels and makes `issuer-domain-name` comparison exact. CAs follow CNAME aliases at `_validation-persist.<R>`, while `domain_name` stays the pre-alias `R`, never a CNAME target. On `accountHashPrefix`, both the client and any delegated publisher MUST verify the prefix is present and uses only RFC 8659 `value` octets and `%` plus RFC 3986 `reserved`/`unreserved` characters—blocking `;x=` parameter injection and `\`/`"` zone-file escapes—and MUST NOT fall back to another value if missing or invalid. Recognized parameter names are ASCII case-insensitive with duplicate recognized parameters rejected; `persistUntil` MUST be unsigned ASCII digits `<= 9223372036854775807`, valid at or before `T` and expired after `T`, and multiple qualifying records are never combined. Finally, CAs MUST NOT reassign an account URL to a different account, and advertised directory `issuerDomainNames` MUST stay in issued challenges while advertised, with a SHOULD migration period on removal."

**Transition:** "`-03` also adds four features. First up is subdomain validation via `ancestorDomain`."

---

## Slide 3: Subdomain Validation via `ancestorDomain` (1:30, protect)

"For subdomain validation, `-03` avoids DNS tree walks with a request-only `ancestorDomain` field in the challenge-response `POST` payload—reusing RFC 9444 Section 2's Ancestor Domain term while leaving the Challenge Object schema and authorization scope unchanged.

The draft separates the certificate identifier `F`, the authorization's `identifier.value` `A` (with `*.` already stripped per RFC 8555), and the record domain `R`. Default `{}` sets `R = A`—including base wildcards like `*.example.com`. Deeper wildcards (`*.dept.example.com`) or subdomains send `{\"ancestorDomain\": \"example.com\"}`.

On the right:

1. CAs MUST recognize `ancestorDomain`—no silent fallback to `A`.
2. DNS length limits (63-octet labels, 253-octet VDN) apply to every `R`, including `R = A`: if `_validation-persist.<A>` is overlong, the CA MUST reject `{}` with HTTP `malformed` without any DNS or JIT query, and the client can select an ancestor.
3. `R` must be a strict whole-label proper suffix of `A`. CAs SHOULD NOT accept public suffixes or ancestors above nested public suffixes—including suffix-level wildcard or RFC 9444 authorizations—with exceptions where one organization controls every name below `R`, such as a brand TLD or private PKI.
4. The record at `R` requires `policy=wildcard`, and the digest hashes `R` (or `*`).
5. The first accepted POST locks `R`, and `ancestorDomain` coexists with RFC 9444 while satisfying only `A`."

**[IF SHORT]:** "`-03` adds a request-only `ancestorDomain` field to `POST /chall`. CAs must recognize it, enforce the 253-octet VDN limit on both `R` and `R = A`, verify a whole-label proper ancestor while applying `SHOULD NOT` to public suffixes (subject to single-organization exceptions), require `policy=wildcard` at `R`, and hash `R` (or `*`)." (45 sec)

**Transition:** "Second feature addition: IP address validation."

---

## Slide 4: IP Address Validation via `_ip-validation-persist` (1:15, protect)

"For IP address validation, `-03` defines an additive RFC 8738 `ip` profile in reverse DNS. The normalized Reverse Zone Domain Name `R` is four reversed octets under `.in-addr.arpa` for IPv4 or 32 reversed lowercase hex nibbles under `.ip6.arpa` (72 octets) for IPv6.

We prepend `_ip-validation-persist.<R>`, register the node name and `ip` validation method with IANA, keep `ip` and `dns` queries and caches separate, and never follow `PTR` records.

Two key details:

- Matching Section 12.2's CNAME rule, both publisher and CA hash the normalized reverse name `R` (or `*`) prior to any CNAME aliases—never a CNAME target. For RFC 2317 delegation, the alias goes at `_ip-validation-persist.<R>` itself.
- Scope is strictly one IP address: `policy=wildcard` is ignored, `ancestorDomain` is rejected with `malformed`, and Security Considerations cover reverse-zone delegation, DNS wildcards, and IP reassignment cleanup."

**[IF SHORT]:** "`-03` adds IPv4 and IPv6 validation at `_ip-validation-persist.<R>` in `in-addr.arpa` and `ip6.arpa`. Digest input uses the normalized reverse name `R` (or `*`) even across RFC 2317 CNAME delegation, scope is strictly one IP address (`policy=wildcard` ignored, `ancestorDomain` rejected), and both IANA registries are updated." (40 sec)

**Transition:** "The next two additions are smaller and self-contained: expiration diagnostics and certificate-key attestation."

---

## Slide 5: `persistExpired` Error Type (0:45)

"`-03` registers `urn:ietf:params:acme:error:persistExpired` with a two-part rule:

1. Mandatory `MUST NOT` privacy gate: never return `persistExpired` if any record validates, or unless a record otherwise qualifying for the CA and live requesting account failed solely due to `persistUntil`, independent of RRset order.
2. `SHOULD` diagnostic: when eligible on a challenge attempt, return `persistExpired` (or `unauthorized` for uniform disclosure). Records are valid through `T` and expired after `T`, `unauthorized` versus `malformed` ordering across failed records is left to CA policy, and JIT still falls back to `pending`."

**[IF SHORT]:** "`-03` registers `persistExpired` with a mandatory `MUST NOT` privacy gate so it only fires when a record belonging to the requesting account and CA failed solely due to `persistUntil`, never leaking another account's or CA's expiry state." (20 sec)

**Transition:** "Next is combining `dns-persist-01` with certificate-key attestation."

---

## Slide 6: Combining with Certificate-Key Attestation (0:45)

"Section 4.6 adds informative deployment guidance on combining `dns-persist-01` with certificate-key attestation. Because authorization `challenges` and CAA `validationmethods` are disjunctive and unknown TXT parameters are `MUST ignore`, deployments achieve conjunction through ACME Order policy (citing `device-attest` informatively):

1. CA order policy includes both the domain authorization—covering wildcards and subdomains—and a non-waivable attestation authorization.
2. At finalization, all Order authorizations are `valid` and the CSR key matches the attested key, even across JIT or authorization reuse.
3. Key-rotating renewals reuse `_validation-persist` via JIT with fresh attestation for the new CSR key, and domain owners scope `_validation-persist` to CAs and accounts enforcing that policy."

**[IF SHORT]:** "Section 4.6 describes combining `dns-persist-01` with certificate-key attestation via ACME Order policy—non-waivable attestation authorization plus CSR public-key verification at finalization—citing `device-attest` informatively." (20 sec)

**Transition:** "Before wrapping up, two questions from Seo Suchan's October 13 Pebble implementation report on the list."

---

## Slide 7: Mailing-List Implementation Feedback (0:50)

"Two questions from Seo Suchan's October 13 Pebble `-02` report:

First, keep `https://` `accountHashPrefix` rather than a custom `accounthash:` scheme: RFC 8657 Section 5.4 recommends an authority component, pre-provisioning already fetches the directory for `issuerDomainNames`, we avoid a BCP 35 scheme registration while matching the CA/B Forum proposal's URI form, and Section 4.5 covers rebrands with required migration periods for `accountHashPrefix` and account URLs and a recommended migration period for directory `issuerDomainNames`.

Second, keep inner `key` fixed to the 43-octet RFC 7638 SHA-256 thumbprint: clients and CAs already compute it for RFC 8555 Section 8.1, and CAs store just one 43-byte thumbprint per historical key regardless of outer `<hash-alg>`."

**[IF SHORT]:** "On the October 13 list thread: we recommend keeping `https://` `accountHashPrefix` (matches RFC 8657 §5.4 authority guidance, saves a BCP 35 URI-scheme registration, and a custom scheme wouldn't remove the directory fetch for `issuerDomainNames`) and keeping the inner JWK thumbprint fixed at RFC 7638 SHA-256 (matching RFC 8555 §8.1 and keeping CA historical key storage to one thumbprint per key)." (25 sec)

**Transition:** "Here is where that leaves the draft and our WGLC timeline."

---

## Slide 8: Implementation Status & Path to WGLC (0:45, protect)

"`-03` resolves all open feature and hardening items—the repository now has zero open issues and zero open pull requests.

On implementations (`-03` conformance not yet verified): Pebble and lego have merged support, Pebble has an open `-02` update, Boulder merged `-01` and is tracking `-02`, and cert-manager's design proposal is in review. Note in Section 8 that earlier-draft deployments keep the `dns-persist-01` protocol-v1 label (like `http-01` and `tls-alpn-01` across pre-RFC drafts) and fail closed without interoperating with `-03`: `-03` clients require `accountHashPrefix` and skip `-01` directories that omit it, such as GTS's `-01` deployment—happy to hear if any operator needs a label bump. In CA/B Forum, Michael's Method 23 proposal currently references `-02`; updating it for `-03` is a next step.

Since `-03` just posted Sunday after the cutoff, we'd like folks to read through it on the list and then ask the chairs to start Working Group Last Call shortly after IETF 127."

**Transition:** "With that, I'll open the floor for questions on the `-03` changes or on WGLC."

---

## Slide 9: Questions & Discussion (Remaining Time)

Open the floor and take questions on the `-03` changes—proactively inviting WG reactions on exact `issuer-domain-name` matching vs. RFC 8659 case-insensitivity, keeping the `dns-persist-01` label across `-01` to `-03`, lifetime witnessed-key retention across account key rollover, and reusing `ancestorDomain` on `POST /chall`. Then make one request to the chairs:

"Chairs: every tracked issue now has text in `-03`. Since `-03` posted after the pre-meeting cutoff, we'd love working group review on the list over the next couple of weeks and would like to ask the chairs to start Working Group Last Call once folks have had a chance to read it."

---

## Likely Questions

### 1. Why is `ancestorDomain` only in the challenge response (`POST /chall`) and not echoed in the server's Challenge Object?

RFC 8555 Section 7.5.1 explicitly provides the client's challenge-response `POST` payload for per-attempt client parameters (the same pattern used by `attObj` in `draft-ietf-acme-device-attest`), while keeping the server's Challenge Object schema (RFC 8555 Section 8) static. In asynchronous ACME servers like Boulder, challenge sub-resources are projections of a compact authorization row rather than mutable per-POST JSON documents. To ensure safety, Item 1 of Section 6.2 makes CA recognition of `ancestorDomain` a strict `MUST` (forbidding silent fallback to `_validation-persist.<A>`), and Section 4 specifies that the first accepted `POST` fixes `R` for that validation attempt.

### 2. For a base-level wildcard `*.example.com`, does the client send `"ancestorDomain": "example.com"` or `{}`?

The client sends `{}`. Under RFC 8555 Section 7.1.3, the ACME server strips the `*.` prefix when creating the authorization object for `*.example.com`, so the authorization's `identifier.value` (`A`) is already `example.com` (with `wildcard: true`). Because a domain is not a proper ancestor of itself (`R` must have strictly fewer labels than `A`), sending `"ancestorDomain": "example.com"` when `A = example.com` is rejected with `malformed`. You only send `"ancestorDomain": "example.com"` when `A` is a deeper name—such as `*.dept.example.com` (`A = dept.example.com`) or `server.dept.example.com`.

### 3. If a client sends an invalid or public-suffix `ancestorDomain`, does that transition the challenge to `invalid` or return an HTTP error while leaving the challenge `pending`?

Section 6.2 specifies this explicitly: syntax, non-ancestor, and public-suffix/policy rejections on the `POST` request (Items 2–4 and 7) are HTTP request errors (RFC 8555 Section 7.5.1) that leave the existing challenge state unchanged (so a challenge that has not yet accepted a response remains `pending`), whereas a DNS verification failure at `_validation-persist.<R>` (Item 5, such as missing `policy=wildcard` or non-matching `accounturi`) is a validation failure that transitions the challenge to `invalid` (RFC 8555 Section 7.1.6).

### 4. How does RFC 2317 classless `in-addr.arpa` CNAME delegation work with `_ip-validation-persist`?

In DNS, a `CNAME` record at `<R>` (for example, `1.2.0.192.in-addr.arpa`) redirects queries for `<R>` itself (such as `PTR`), but does **not** redirect queries for child labels beneath `<R>` like `_ip-validation-persist.1.2.0.192.in-addr.arpa` (which would require a `DNAME` at the parent or an explicit `CNAME` at `_ip-validation-persist.<R>`). Section 7 states this explicitly: delegating `_ip-validation-persist.<R>` via `CNAME` requires an alias at `_ip-validation-persist.<R>` itself, and both the DNS publisher and the CA still compute `domain_name` in the hash over the normalized `.in-addr.arpa` or `.ip6.arpa` Reverse Zone Domain Name `R` (or `*`), never the `CNAME` target.

### 5. Why does Section 7 use `"normalized Reverse Zone Domain Name"` rather than `"canonical Reverse Zone Domain Name"`?

In DNS (RFC 1034 Section 3.6.2 / RFC 1035 Section 3.3.1), `"canonical name"` is a term of art meaning the target of a `CNAME` alias—which is the exact opposite of `R` when `_ip-validation-persist.<R>` is aliased via `CNAME`. Using `"normalized Reverse Zone Domain Name"` (and noting that RFC 1035 and RFC 3596 write `IN-ADDR.ARPA` and `IP6.ARPA` in uppercase while Section 7 normalizes to lowercase) eliminates that ambiguity.

### 6. Can a reverse-zone operator use a DNS wildcard record (`*.2.0.192.in-addr.arpa`) with `domain_name=*` to answer `_ip-validation-persist` queries for multiple IPs?

Yes—and Section 10.6 (`{{#ip-validation-risks}}`) documents this explicitly in Security Considerations. Ignoring `policy=wildcard` ensures that a single ACME authorization covers only one IP address, and separate node names (`_ip-validation-persist` vs. `_validation-persist`) and typed caches keep `ip` and `dns` validations separate. However, because the reverse-zone operator controls all nodes in its zone, it can synthesize DNS wildcard responses across addresses in that zone if it uses the `domain_name=*` opt-out. That is inherent to reverse-DNS delegation, which is why CAs only enable `dns-persist-01` for `ip` identifiers where permitted by their CP/CPS and trust-program rules.

### 7. Why not add a `require-attestation` parameter to the `_validation-persist` DNS TXT record instead of using CA Order policy?

Because Section 4.1 Item 1 requires CAs to ignore unrecognized parameters in `_validation-persist` records (so future optional extensions don't break legacy CAs). A DNS parameter that an unaware CA silently ignores cannot fail closed across CAs. Instead, as Section 4.6 describes informatively, a domain owner that requires certificate-key attestation scopes its `_validation-persist` record (via `issuer-domain-name` and `accounturi`, reinforced by CAA) to a CA and account whose Order policy includes a non-waivable attestation authorization on each Order and verifies the CSR public key at finalization.

### 8. What about the recomputation DoS discussion on retained historical keys?

Section 12.3 (`{{#ca-implementation-guidelines}}`) notes that while current-key-first evaluation makes normal validations `O(1)` and failed-validation rate limits bound failed attempts, an admitted attempt that succeeds against an older retained thumbprint can still scan the account's retained-key history. Because the account holder controls how many times it calls `keyChange`, CAs bound worst-case per-attempt work by rate-limiting `keyChange` or admitted attempts per account (and computing candidate hashes once per retained key across an RRset), without reintroducing a 50-key eviction cap that would break dormant backup-CA records.

### 9. Can `_validation-persist.<R>` be a CNAME alias, and what is hashed in `domain_name`?

Yes. Section 12.2 (`{{#normalization-algorithm}}`) specifies that CAs follow CNAME aliases at the Validation Domain Name as for any TXT query, while `domain_name` in `H(length_of_domain || domain_name || key || account_URL)` remains the pre-alias `R` (or `*`), never a CNAME target. Thus a record computed over a specific `R` cannot be lifted to authorize another domain via CNAME, while a domain owner who wants one shared CNAME target across many domains uses the explicit `domain_name = *` opt-out.

### 10. Why do `-03` clients skip `-01` deployments like Google Trust Services today?

Section 4.1 Item 3 requires clients and delegated publishers to verify before building a record that `accountHashPrefix` is present in the directory and contains only RFC 8659 `value` octets and `%` plus RFC 3986 `reserved`/`unreserved` characters, with no fallback to raw account URLs. That check prevents a tampered or crafted prefix (such as one containing `;x=` or master-file escapes) from turning the client's digest into an ignored parameter. Because `-01` directories omit `accountHashPrefix` while using the same `dns-persist-01` challenge name, `-03` clients treat `dns-persist-01` as unavailable there until the CA advertises `accountHashPrefix` (noted non-normatively in Section 8, Implementation Status).
