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

## Time Discipline (lessons from IETF 124)

At IETF 124, the dns-persist slot ran over time. The chair had to cut the
queue. With 4 discussion items in a shorter slot, strict time management is
critical.

**Hard rule:** If the subdomain discussion (slides 3–4) runs past 6:00
cumulative, compress everything from TTL onward to `[IF SHORT]` versions.
Subdomain is the highest-value discussion — protect that time; defer remaining
topics to the mailing list.

## Likely Tangential Questions

These came up at IETF 124 or on the mailing list and may resurface:

- **"How does cert renewal work with dns-persist?"** → "The CA re-queries the
  DNS record at renewal time. The persistent record remains in place, so
  renewal succeeds without subscriber interaction."
- **"Can someone still use dns-01 or http-01 alongside dns-persist?"** →
  "Yes, they are independent challenge methods. CAA can constrain which
  methods a CA may use."
- **"Should DNSSEC be SHOULD or MUST?"** → This needs a nuanced answer.
  The question has two forms:

  *Form 1 — "MUST deploy DNSSEC":* No. Mandating DNSSEC deployment would
  make dns-persist unusable for most domains. Closed in issue #1.

  *Form 2 — "MUST use a DNSSEC validating resolver" (Sheth/Verisign, Kaduk):*
  **We agree with this.** SC-085 (authored by Birge-Lee) already requires
  this for all DCV lookups by the primary network perspective, effective
  March 15, 2026. The BRs now say: "DNSSEC validation back to the IANA
  DNSSEC root trust anchor MUST be performed on all DNS queries associated
  with the validation of domain authorization or control." This is already
  in force. Domains without DNSSEC still work fine — the resolver finds
  no signatures to validate. Domains *with* DNSSEC get fail-closed behavior.

  *Action:* Change §7.7.1 from "SHOULD validate DNSSEC signatures" to
  "MUST use a DNSSEC validating resolver" in -02, aligning with SC-085
  and the dnsop domain-verification-techniques BCP.

  *If Kaduk or Sheth raise this:* "You're right. SC-085 is now effective
  and our co-author wrote it. We will align the draft with the BRs in -02.
  Private PKIs can profile this down if needed."
- **"What about persistent token security?"** (Ben Kaduk's concern) →
  "We expanded security considerations significantly in -00 and -01. Account
  binding, multi-perspective validation, and persistUntil provide defense in
  depth. We can discuss specifics on the mailing list."
- **"I don't buy the IoT use case"** (Michael Richardson's concern) →
  "The primary use cases are multi-tenant platforms and enterprise domain
  management. Aaron Gable described the two core problems during adoption:
  webserver challenge operational difficulties, and dns-01 API key exposure
  risk. IoT is a secondary motivation."

If these consume discussion time, redirect: "We addressed that between
meetings and can continue on the mailing list. Let's return to the subdomain
question."

## People Likely in the Room

- **Aaron Gable** (Let's Encrypt) — Co-filed #42 (TTL), #32 (IP).
  Previously suggested tree-climbing at order creation time (Oct 2025).
  Likely supportive; may press on timing or mechanism details.
- **Ben Kaduk** — Insisted on strong security considerations during adoption.
  Endorsed Sheth's "MUST use DNSSEC validating resolver" position on the list.
  May raise persistent token risks or DNSSEC. Rigorous reviewer.
- **Michael Richardson** — Skeptical of some use cases at adoption. Asked about
  http-01 coexistence at IETF 124. May raise tangential concerns about
  interaction with other challenge types.
- **Ilari Liusvaara** — Noted LE caps TTL to 1 min. Concise, technical.
- **Sergey Frolov** (Google Trust Services) — Supports same-draft for IP
  validation. May speak for it if prompted.
- **Fabien Hochstrasser** (Google Trust Services, remote) — GTS's designated
  participant for this session. Gurleen (GTS) emailed the day before asking
  authors to raise #32. Call on Fabien directly during the IP validation slide.
- **Swapneel Sheth** (Verisign) — Supported adoption. Recommended "MUST use
  DNSSEC validating resolver" aligning with dnsop BCP. May raise this if
  DNSSEC comes up.
- **Chair (Ounsworth)** — Asked authors to "canvas implementer community and
  report back." Our implementation slide addresses this directly.

---

## Slide 1: Title (15 seconds)

"dns-persist-01 — our second WG presentation. Substantial changes since
Montreal."

**For newcomers** (verbal only, not on the slide): "Quick context for
anyone new: dns-persist lets domain owners place a persistent DNS TXT record
that authorizes a specific CA to issue certificates, without requiring
per-issuance interaction. It was adopted as a WG document last October."

---

## Slide 2: Since IETF 124 (1:30)

**Key points:**

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
that term differently. Renamed to "Validation Domain Name" per RFC 8555 §8.4.
Unblocks the subdomain work we'll discuss next.

Pebble PR merged in February — the spec is implementable. lego client support
also merged, giving subscribers a tool to experiment with. cert-manager has a
community feature request with multiple +1s, and a work-in-progress
implementation behind a feature gate.

-01 hasn't been published yet. We found editorial issues during review and want
to fix those before tagging.

**[IF SHORT]:** "Three PRs merged. Biggest: accounturi decoupled from the
challenge object, enabling pre-provisioning. Pebble, lego, and cert-manager
implementations underway. -01 coming soon." (30 sec)

**Transition:** "The accounturi decoupling enables pre-provisioning, and that
leads directly to our primary discussion topic..."

---

## Slide 3: Subdomain Validation — The Gap (1:00)

**Walk through the diagram:**

"Imagine you manage example.com. You've set up a validation-persist record with
policy=wildcard. You want a cert for server.dept.example.com — three labels
down.

Today, ACME has no way for you to tell the CA 'check at example.com.' The CA
looks at the FQDN level, finds no record, and fails. Without the adn field, the
CA either does an expensive fan-out — querying at every ancestor — or only
validates exact names.

Fan-out is problematic: expensive, ambiguous when records exist at multiple
levels, creates MPIC amplification risk."

**Why this matters:** Enterprise demand exists. Let's Encrypt forum users have
raised this. Birge-Lee identifies it as a significant ACME gap. The Let's
Encrypt blog post (Feb 2026) already describes `policy=wildcard` as covering
"subdomains whose suffix matches the validated FQDN" — they expect this to
work. The `adn` field is the mechanism that makes it work.

**[IF SHORT]:** Show the diagram, two sentences. "No mechanism to say 'check
the parent.' Here's what we propose." (30 sec)

**Transition:** "Here's what we're proposing..."

---

## Slide 4: Proposed adn Field (2:30 — primary discussion)

**Walk through the mechanism:**

"RFC 8555 §7.5.1 designed this extension point. The client POSTs an adn field
to the challenge URL with the domain to check. The CA confirms it's a proper
suffix of the requested FQDN, looks up the record there, and validates.

Two open questions."

**Question 1 — Depth:**

"Should we limit how many labels can be pruned? The BRs allow pruning to the
base domain name. We could match that."

*Our position:* Match BR depth rules.

*Fallback:* Defer to BR semantics unless someone objects.

**Question 2 — Where:**

"We'd like to add this to dns-persist directly — same mechanism, ships in -02.
Any objections?"

**Mailing list context (Aaron, Oct 29 2025):** Aaron suggested tree-climbing
would happen at *order creation time* — "the CA already has a cached
dns-persist-01 validation for example.com, they may populate the order object
with that pre-validated authorization." Our `adn` field puts it in the
*challenge response* instead, giving the client explicit control. These are
complementary: order-creation caching is a CA optimization; the `adn` field is
the client's way to tell the CA which level to validate at when no cached
validation exists. If Aaron raises this, acknowledge it: "Your order-creation
approach works for CAs that proactively cache. The `adn` field covers the case
where the client knows the answer and the CA doesn't."

**If someone asks "what about dns-01?":** "Applying this to dns-01 is an
interesting idea but out of scope for this draft. That would be an RFC 8555
update — a separate effort. We're focused on dns-persist-01."

**Anticipated objections:**

- *"This is scope creep."* → "Without subdomain validation, dns-persist is
  incomplete for its primary use case — enterprise domain management. The
  extension point exists in 8555. We're adding one field."
- *"CA fan-out is simpler."* → "Fan-out creates MPIC amplification, is
  ambiguous when records exist at multiple levels, and is expensive. The client
  knows which domain it validated; it should specify it."
- *"What about security — arbitrary domain pointers?"* → "The CA confirms
  the adn is a proper suffix of the requested FQDN. Pointing to an
  unrelated domain is not possible."
- *"Tree-climbing should happen at order creation, not challenge response."*
  (Aaron's Oct 2025 position) → "Complementary. Order-creation caching is a
  CA optimization for pre-validated domains. The `adn` field is for the case
  where the client is telling the CA where to look for the first time."

**[IF SHORT]:** Show the mechanism, ask depth question only. "Should we match
BR pruning rules?" (1:30)

**Transition:** "Next discussion topic — a question about validation reuse..."

---

## Slide 5: TTL as Validation Ceiling (2:00)

**Context from IETF 124:** At Montreal, I described the three-ceiling model as a
feature — "the DNS TTL must be respected." Mailing list analysis since then
convinced us the TTL ceiling is counterproductive. Acknowledge this evolution
if anyone cites the previous position.

**Present the model:**

"Current text defines three ceilings on how long a CA can reuse validation data.
The table shows all three. We propose removing the third — DNS TTL.

Issue 42 argues this is layer confusion. TTL governs DNS caching — a hint to
resolvers. It was never designed to control CA business logic.

The practical problem is worse: when you query a recursive resolver, you don't
see the authoritative TTL — you see remaining cache time. If someone else
queried 29 seconds ago and the TTL is 30, you see 1 second. Your validation
reuse period becomes 1 second. Clearly not the intended behavior.

Operationally: many DNS providers default to 30–60 second TTLs. A subscriber
who sets up dns-persist without adjusting TTL would find validation expires
before the CA can issue. The subscriber's intent was persistent validation."

**Counterargument to air:**

"One argument for keeping TTL-as-ceiling: it gives subscribers a way to force
revalidation by lowering their TTL, without re-provisioning the record.
Removing TTL means persistUntil is the only subscriber-controlled ceiling."

**Response:**

"persistUntil is a much better mechanism. Explicit, visible, not subject to
caching artifacts. The TTL 'mechanism' is accidental and unreliable. If you
want to force revalidation, update persistUntil."

**Mailing list support for removal:**

- *Ilari Liusvaara (Oct 10, 2025):* "Resolvers can cap the DNS TTL. AFAIK,
  Let's Encrypt caps the TTL to 1 minute." If LE already caps to 1 min, the
  TTL ceiling in §9.5 is already meaningless in practice for the largest CA.
- *Aaron Gable (Oct 10, 2025):* Validation reuse should be "addressed at
  the PKI policy level, but not necessarily something that should be baked
  into the normative text." Removing TTL-as-ceiling aligns with this.
- *Ben Kaduk (Oct 10, 2025):* Called for security considerations discussion
  but said he was "merely soliciting discussion on what (if any) hard
  protocol requirements we need to include" — not insisting on keeping TTL.

**Important context:** Only 2 people have weighed in on the GitHub issue. But
the mailing list discussion shows broader comfort with handling reuse at the
policy level rather than in protocol text.

**Ask:** "Does any implementation use TTL to force revalidation?" If
silence → consensus to remove.

**Fallback:** If someone objects, offer to weaken from MUST to MAY ("CA MAY
consider TTL"). Push for removal.

**[IF SHORT]:** Show table, one sentence per "why remove" bullet, ask the
question. (1:00)

**Transition:** "One more discussion topic, then quick items..."

---

## Slide 6: IP Validation via Reverse DNS (1:00)

"Aaron Gable filed this. CA/Browser Forum is voting on a ballot that adds
dns-persist for IP addresses using reverse DNS. The label would be
_ip-validation-persist, distinct from _validation-persist.

Same mechanism, same parameters, different validation target.

The question: same draft or companion document? Trade-offs on the slide."

*Our preference:* Same draft — less overhead, same mechanism. But if the CA/BF
ballot timeline creates uncertainty, a companion is fine.

**Mailing list context (Sergey Frolov, Google Trust Services, Dec 17 2025):**
Frolov explicitly wrote: "Should the current dns-persist-01 draft also define a
validation method satisfying SC-091's requirements? ... This would eliminate the
need for a separate, nearly duplicate IETF stream." Google Trust Services
offered to help draft text. Strong signal for "same draft."

If someone asks "who wants this?": "Google Trust Services wrote to the list in
December explicitly supporting same-draft and offering to help with text.
They also emailed yesterday asking us to raise this — Fabien Hochstrasser
is on the call for GTS. Fabien, anything to add?"

**[IF SHORT]:** "IP validation via reverse DNS — same draft or companion?
GTS supports same-draft. Fabien, anything to add? Otherwise we defer to
the list." (30 sec)

**Transition:** "Two quick items..."

---

## Slide 7: Quick Items (1:00)

**Error types (30 sec):**

"Three failure modes all return 'unauthorized' — the client can't tell if its
accounturi is wrong, its persistUntil expired, or the issuer doesn't match.
Should we register ACME error types? No extension has done this yet. Our
suggestion: wait until the adn field settles — it adds more failure modes —
then design the error space once. Does this block other work?"

**Client persistUntil check (30 sec):**

"Should clients check persistUntil before responding to a challenge? Enterprise
clients behind split-horizon DNS cannot see the validation record. Is a SHOULD
useful or confusing?"

**[IF SHORT]:** "Two quick items — error types and client-side checks. Both
deferred pending larger design. Objections?" (15 sec)

**Transition:** "Wrapping up..."

---

## Slide 8: Path Forward (30 seconds)

"Four priorities for -02: resolve TTL, spec the adn field, decide IP validation
scope, and editorial cleanup.

WGLC target is after -02 or -03, depending on how today's discussions resolve.

Implementation continues — Pebble and lego both merged. Let's Encrypt published
a blog post in February announcing implementation: staging late Q1, production
Q2 2026. Fastly/Certainly committed for 2026. Amazon Trust Services assessing.
Thank you."

---

## Slide 9: Questions (remaining ~5 minutes)

Open floor. If no questions, prompt:
- "Are there concerns about the adn approach we did not address?"
- "Feedback on the path toward WGLC?"
