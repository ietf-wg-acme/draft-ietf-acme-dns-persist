# Speaker Notes: draft-ietf-acme-dns-persist — IETF 125

**Total time: ~15 minutes**
**Speaking: ~10 minutes | Discussion/Q&A: ~5 minutes**

## Timing Summary

| Slide | Topic | Time | Cumulative |
|-------|-------|------|------------|
| 1 | Title | 0:15 | 0:15 |
| 2 | Since IETF 124 | 1:30 | 1:45 |
| 3 | Subdomain: The Gap | 1:00 | 2:45 |
| 4 | Subdomain: adn Proposal | 2:30 | 5:15 |
| 5 | TTL Ceiling | 2:00 | 7:15 |
| 6 | IP Validation | 1:00 | 8:15 |
| 7 | Quick Items | 1:00 | 9:15 |
| 8 | Path Forward | 0:30 | 9:45 |
| 9 | Questions | 5:15 | 15:00 |

## Time Discipline

At IETF 124, the dns-persist slot ran over and the chair cut the queue. With
4 discussion items in a shorter slot, strict time management is critical.

**Hard rule:** If the subdomain discussion (slides 3–4) runs past 6:00
cumulative, compress everything from TTL onward to `[IF SHORT]` versions.
Subdomain is the highest-value discussion — protect that time; defer remaining
topics to the mailing list.

---

## Slide 1: Title (0:15)

"dns-persist-01 — our second WG presentation. Substantial changes since
Montreal."

For newcomers (verbal only, not on slide): "Quick context for anyone new:
dns-persist lets domain owners place a persistent DNS TXT record that authorizes
a specific CA to issue certificates, without requiring per-issuance interaction.
It was adopted as a WG document last October."

---

## Slide 2: Since IETF 124 (1:30)

Three PRs merged since Montreal.

**accounturi (PR #35):** The most significant change. In -00, accounturi
validation was tied to the challenge object — that model breaks for
pre-provisioned records and JIT validation where no challenge object exists.
Now the DNS record stands on its own. Also added a many-to-one URI model: CAs
MUST accept the account URL but MAY issue alternative URIs to prevent
cross-domain correlation.

**Case-handling (PR #36):** Came from a Pebble implementer question — "why is
`policy` case-insensitive but `accounturi` isn't?" Answer: different types. URI
comparison follows RFC 3986, integers don't have case, only protocol keywords
get case-insensitivity. Now explicit for all three parameters.

**VDN rename (PR #41):** We used "Authorization Domain Name" but the BRs define
that term differently. Renamed to "Validation Domain Name," following the
precedent in RFC 8555 §8.4. Unblocks the subdomain work we'll discuss next.

Implementation: Pebble merged in February — the spec is implementable. lego
client support also merged. cert-manager has a community feature request with
multiple +1s and a work-in-progress implementation behind a feature gate.

-01 is pending editorial fixes.

**[IF SHORT]:** "Three PRs merged. Biggest: accounturi decoupled from the
challenge object, enabling pre-provisioning. Pebble, lego, and cert-manager
implementations underway. -01 coming soon." (30 sec)

**Transition:** "The accounturi decoupling enables pre-provisioning, and that
leads directly to our primary discussion topic..."

---

## Slide 3: Subdomain Validation — The Gap (1:00)

Walk through the diagram:

"Imagine you manage example.com. You've set up a validation-persist record with
policy=wildcard. You want a cert for server.dept.example.com — three labels
down.

The right side shows what happens. The CA prepends _validation-persist to the
requested FQDN and queries DNS. Starting at the bottom — nothing at
server.dept. Move up — nothing at dept. The record is at
_validation-persist.example.com, but the CA does not know which level the
client intended. Without a signal from the client, the CA either stops at the
first failure or fans out across every level — unspecified, expensive, and
ambiguous when records exist at multiple levels."

Context: The Let's Encrypt blog post (Feb 2026) already describes
`policy=wildcard` as covering "subdomains whose suffix matches the validated
FQDN" — implementers expect this to work. The `adn` field is the mechanism
that makes it work.

**[IF SHORT]:** Show the diagram, two sentences. "No mechanism to say 'check
the parent.' Here's what we propose." (30 sec)

Note: Tell the room before advancing to slide 4: "I'll take subdomain
questions during Q&A — let me get through the remaining topics first."

**Transition:** Point to the gold box. "Our answer — a new adn response field.
Next slide walks through the mechanism."

---

## Slide 4: Proposed adn Field (2:30 — primary discussion)

Walk through the mechanism:

"RFC 8555 §7.5.1 designed this extension point. The client POSTs an adn field
to the challenge URL with the domain to check. The CA confirms it's a valid
Authorization Domain Name — obtained by pruning labels from the requested FQDN
per the BRs — looks up the record there, and validates.

Note: This modifies the currently-specified empty-object POST to the challenge
URL — the current draft says clients send {}. Backward compatibility is
guaranteed by RFC 8555 §7.5.1: servers MUST ignore unrecognized response fields,
so old servers that do not support adn will ignore it.

Two open questions."

**Question 1 — Depth:** "Should we limit how many labels can be pruned? The
BRs allow pruning to the base domain name. We think matching BR depth rules
is the right approach."

**Question 2 — Where:** "We propose adding adn to this draft, targeting -02.
Does the room agree with same-draft scope?"

**Related mailing list discussion:** On Oct 29, 2025, Aaron Gable suggested
that tree-climbing could happen at order creation time — the CA populates the
order with a pre-validated authorization from a cached dns-persist record. Our
`adn` field takes a different approach: the client specifies the ancestor in
the challenge response. These are complementary — order-creation caching is a
CA optimization, while the `adn` field gives clients explicit control when no
cached validation exists.

**[IF SHORT]:** Show the mechanism, ask depth question only. "Should we match
BR pruning rules?" (1:30)

**Transition:** "Next discussion topic — a question about validation reuse..."

---

## Slide 5: TTL as Validation Ceiling (2:00)

Context: We wrote the TTL ceiling as a conservative safety measure — cap
validation reuse at the DNS TTL so there is some responsiveness to DNS changes.
We now think this was a mistake. Say so directly.

Present the model:

"Current text defines three ceilings on how long a CA can reuse validation data.
The table shows all three. The third — DNS TTL — is normative MUST language in
§9.5 that we wrote. This proposal removes that normative requirement in -02, and
I want to explain why we changed our position.

Issue 42 argues this is layer confusion. TTL governs DNS caching — a hint to
resolvers. It was never designed to control CA business logic.

The practical problem is worse: when you query a recursive resolver, you don't
see the authoritative TTL — you see remaining cache time. If someone else
queried 29 seconds ago and the TTL is 30, you see 1 second. Your validation
reuse period becomes 1 second. Clearly not the intended behavior.

Operationally: many DNS providers default to 30–60 second TTLs. A subscriber
who sets up dns-persist without adjusting TTL would find validation expires
before the CA can issue."

**Counterargument worth raising:** "The strongest argument for keeping it:
TTL adjustment does not require modifying the record. A subscriber whose key is
compromised can lower the TTL to force revalidation without touching record
content — useful when DNS and PKI are managed by different teams. The counter:
what you see as a resolver client is remaining cache time, not authoritative TTL,
so the signal is unreliable. And persistUntil exists for exactly this purpose,
though it does require a record change."

**Mailing list context (all Oct 10, 2025):**

- *Ilari Liusvaara:* "Resolvers can cap the DNS TTL. AFAIK, Let's Encrypt
  caps the TTL to 1 minute." — if accurate, TTL ceiling is already meaningless
  in practice for the largest CA.
- *Aaron Gable:* Validation reuse should be "addressed at the PKI policy level,
  but not necessarily something that should be baked into the normative text."
- *Ben Kaduk:* Called for security considerations discussion but said he was
  "merely soliciting discussion on what (if any) hard protocol requirements we
  need to include."

Two comments on the GitHub issue; broader discussion on the mailing list in
October. We will remove TTL-as-ceiling in -02. The counterargument above is
prepared if someone objects.

**[IF SHORT]:** Show table, one sentence per "why remove" bullet, ask the
question. (1:00)

**Transition:** "One more discussion topic, then quick items..."

---

## Slide 6: IP Validation via Reverse DNS (1:00)

"CA/Browser Forum ballot SC-91 passed unanimously in November — it adds
dns-persist for IP addresses using reverse DNS. The new BR method 3.2.2.5.8
is effective now; the old reverse PTR method (3.2.2.5.3) it replaces sunsets
March 15, 2027.

Same mechanism, same parameters, different validation target.

The question: same draft or companion document? Trade-offs on the slide."

**Context:** SC-91 was proposed by Gurleen Grewal (GTS) and endorsed by
Michael Slaughter (our co-author at ATS). On the mailing list (Dec 17, 2025),
Sergey Frolov (GTS) wrote: "Should the current dns-persist-01 draft also
define a validation method satisfying SC-91's requirements? This would
eliminate the need for a separate, nearly duplicate IETF stream." GTS offered
to help draft text.

We prefer same-draft — less overhead, same mechanism — but a companion
document is fine if the WG prefers a narrower scope.

Fabien Hochstrasser (GTS) is participating remotely in this session. Make sure
to invite their input during this slide.

**[IF SHORT]:** "IP validation via reverse DNS — same draft or companion?
GTS supports same-draft. Any input from the room?" (30 sec)

**Transition:** "Two quick items..."

---

## Slide 7: Quick Items (1:00)

**Client persistUntil check (45 sec):**

"Should clients check persistUntil before responding to a challenge? Enterprise
clients behind split-horizon DNS cannot see the validation record. The trade-off:
a SHOULD catches stale records early for clients that can see them, but fails
silently for clients that cannot. Worth adding, or does split-horizon make this
guidance unreliable?"

**Error types (15 sec):**

"Error types deferred — new failure modes from the adn field may reshape the
error space. We want to design it once."

**[IF SHORT]:** "Quick item — should clients check persistUntil locally?
Split-horizon makes this tricky. Error types deferred. Objections?" (15 sec)

**Transition:** "Wrapping up..."

---

## Slide 8: Path Forward (0:30)

"Four priorities for -02: close TTL, spec the adn field, decide IP validation
scope, and editorial cleanup.

WGLC at -02; slips to -03 if IP validation is same-draft.

Implementation continues — Pebble and lego both merged. Let's Encrypt published
a blog post in February announcing implementation plans. Can anyone from Let's
Encrypt share where staging stands? Fastly committed for 2026. Amazon
Trust Services assessing. Thank you."

*Context for LE question:* The February blog post targeted staging late Q1 and
production Q2 2026. If no LE representative is present or declines, state:
"Per their February blog post, staging was targeted for late Q1 and production
for Q2."

---

## Slide 9: Questions (remaining ~5 minutes)

Open floor. If no questions, prompt:
- "Are there concerns about the adn approach we did not address?"
- "Feedback on the path toward WGLC?"

---

## Likely Questions

These came up at IETF 124 or on the mailing list and may resurface.

**"How does cert renewal work with dns-persist?"**
The CA re-queries the DNS record at renewal time. The persistent record remains
in place, so renewal succeeds without subscriber interaction. For CAs
implementing JIT validation, the renewal may auto-validate without a challenge
round-trip.

**"Can someone still use dns-01 or http-01 alongside dns-persist?"**
Yes, they are independent challenge methods. CAA can constrain which methods a
CA may use.

**"What about applying adn to dns-01?"**
Interesting idea but out of scope for this draft. That would be an RFC 8555
update — a separate effort.

**"Should DNSSEC be SHOULD or MUST?"**
The question has two forms:

*"MUST deploy DNSSEC":* No. Mandating DNSSEC deployment would make dns-persist
unusable for most domains. Closed in issue #1.

*"MUST use a DNSSEC validating resolver":* SHOULD is the right level. The
current draft text has a bare SHOULD without conditions, which Deb Cooley (AD)
flagged at IETF 124 as an RFC 2119 problem. The fix for -02 states the
consequence of deviation: "A CA that does not use a DNSSEC validating resolver
cannot detect DNSSEC failures and will silently accept forged responses for
DNSSEC-signed zones."

As supporting evidence: WebPKI CAs already MUST do this via the BRs — SC-085
(authored by Birge-Lee, now effective as of March 15, 2026) requires DNSSEC
validating resolvers for all DCV lookups. But the IETF spec should stand on its
own rationale, not depend on external policy documents.

Tracked in issue #44.

**"What about persistent token security?"**
Security considerations were expanded significantly in -00 and -01. Account
binding, multi-perspective validation, and persistUntil provide defense in
depth. Happy to discuss specifics on the list.

**"I don't buy the IoT use case."**
The primary use cases are multi-tenant platforms and enterprise domain
management. Aaron Gable described the two core problems during adoption:
webserver challenge operational difficulties, and dns-01 API key exposure risk.
IoT is a secondary motivation.
