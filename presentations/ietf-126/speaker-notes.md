<!-- markdownlint-disable MD013 MD060 -->

# Speaker Notes: draft-ietf-acme-dns-persist — IETF 126

**Total slot: 10 minutes.** This is a short slot, closer to a status report than
IETF 125's discussion-heavy deck. Speak for at most 8 minutes and leave 2 for
questions; push detailed discussion to the mailing list and GitHub.

## Timing Summary

| Slide | Topic | Time | Cumulative |
|-------|-------|------|------------|
| 1 | Title | 0:15 | 0:15 |
| 2 | Since IETF 125 | 1:00 | 1:15 |
| 3 | Substitution gap (#64) | 1:15 | 2:30 |
| 4 | PR #67: offline binding | 1:30 | 4:00 |
| 5 | Mandatory hashed-URI binding | 1:30 | 5:30 |
| 6 | Final construction review | 0:45 | 6:15 |
| 7 | What lands in `-02` | 0:45 | 7:00 |
| 8 | Path forward | 1:00 | 8:00 |
| 9 | Questions | 2:00 | 10:00 |

## Time Discipline

At IETF 124 the dns-persist slot ran over and the chair cut the queue. This is a
10-minute slot, not 15. **Hard rule:** if slide 5 (the core-package slide) is
not on screen by 4:15 cumulative, use the `[IF SHORT]` versions for slides 3 and
4 and deliver slide 6 in one sentence. Slides 4 and 5 are the point of the talk.
Everything else can move to the list.

Remote delivery removes room-read, so plan to land the talk near 7:15, not
8:00, and let Q&A absorb the slack. The consensus questions (author agreement,
silence versus consensus) will use all of it. Deliver slide 6 as its
`[IF SHORT]` version by default and expand only if ahead of the clock; keep
the privacy sentence either way. Budget slide 8 at 1:15 in practice.

The goal: report the chosen core package (drop 3a and 3c, mandatory hashed URI),
state the two exact extension proposals still under author review, and invite WG
feedback before -02 publishes.

---

## Slide 1: Title (0:15)

"dns-persist, third WG presentation. Short update today: most of the activity
since Shenzhen has been on one issue, the substitution hardening in #64."

For newcomers (the slide subtitle now carries this; reinforce verbally):
"dns-persist lets a domain owner place a
long-lived DNS TXT record that authorizes a specific CA account to issue
certificates, without per-issuance interaction."

---

## Slide 2: From Alternatives to PR #67 (1:00)

"One security-text correction merged on its own. The Security Considerations
tell CAs to protect a DNS name with DNSSEC. The old sentence named
issuer-domain-name, an opaque identifier carried in the TXT record; clients
never resolve it. What clients resolve and connect to is the ACME directory
URL, so PR #66 moved the DNSSEC guidance there.

For the substitution problem in #64, we considered three approaches. PR #62
used an authenticated POST to the supplied account URI. That detects
substitution, but requires a live endpoint and an online client. PR #65 bound
the record directly to a public-key hash. That works offline, but introduces a
new fixed wire format and requires a DNS update when the account key rolls over.

PR #67 combines the useful properties. The client computes the value offline,
the account URL preserves account identity, and prior-key matching supports
rollover continuity. That is the design we selected. PR #68 has merged into
the PR #67 candidate, which remains an open draft, so nothing is locked. The selection criterion was
non-transferability without giving up offline provisioning or workable key
rollover."

**[IF SHORT]:** "PR #66 moved the CA DNSSEC guidance from issuer-domain-name to
the ACME directory URL. For substitution, #62
was an online proof, #65 was a key-specific DNS binding, and #67 is the current
choice because it combines offline verification with account and rollover
continuity." (30 sec)

**Transition:** "Let me state the problem these proposals address."

---

## Slide 3: The Substitution Gap (1:15)

"The property at stake is non-transferability. A validation performed by an
honest domain owner should not become authorization for a different account at
an honest CA. RFC 8555 preserves this for the built-in challenges by binding the
account key into the response, computed offline by both client and server,
sections 8.1 and 10.2.

Mode 3a of dns-persist does not preserve it. The record carries only a
server-provided accounturi. If a malicious, compromised, or impersonated ACME
endpoint poisons what the client believes its own account URL is, at account
creation, then the client's self-check and the CA's later Simple String
Comparison both pass against the same poisoned reference; nothing detects the
swap. The victim ends up publishing a record that authorizes the attacker's
account at a real CA. If pressed, note the flip side: the victim's own
issuance then fails, which is a detection channel.

I want to be precise about the threat: this is not 'protecting a client that
chose to trust a bad server.' The narrow, real property is non-transferability."

**Threat clarification if asked** (from the thread tail): the target is a
malicious, compromised, or impersonated ACME endpoint: a typosquatted or
DNS-redirected API hostname, or a compromised serving edge (CDN), as in the
acme.sh RCE precedent; a TLS-defeating MITM on the client link is the same
case, not the motivating one. In scope: whatever serves the directory and
account responses. Out of scope: a compromised client host and the CA's
issuance/signing backend. If pressed on the CDN point: the serving edge is
CA-side infrastructure, and the mechanism's job is that even a lying edge
fails closed. A poisoned `accountHashPrefix` or account URL yields a record
that validates for no account, because the CA recomputes only over the
requesting account's proven keys and URL.

**[IF SHORT]:** Read the right column only. "Record carries only a server-given
accounturi; a bad server poisons the client's own URL, so the self-check passes
and the victim authorizes the attacker. RFC 8555 stops this by binding the key;
mode 3a doesn't." (45 sec)

**Transition:** "Here's what the thread converged on to close that gap."

---

## Slide 4: PR #67 Offline Binding with Account Continuity (1:30, protect)

"PR #67 combines client-computed proof with a stable account identity. The
client hashes a one-octet length prefix, the domain being validated, the account
key, and the account URL. It base64url-encodes the digest under a CA-published
prefix with a hash-algorithm token, producing a record like the example shown on screen.

Each input has a job. The domain makes the value name-specific and prevents
cross-name reuse. The RFC 7638 account-key thumbprint supplies the
client-computed proof. The account URL ties that proof to one ACME account. The
length prefix plus the fixed-length thumbprint make every field boundary
unambiguous. The `sha-256` token identifies the
current algorithm, while `accountHashPrefix` lets the client construct the URI
from directory metadata without another request.

The CA independently recomputes the digest over the account's proven keys and
uses Simple String Comparison. That restores non-transferability. Prior-key
matching provides rollover continuity. This closes the substitution gap without
requiring the account URI to be a live endpoint or defining a public-key URI
scheme."

**[IF SHORT]:** "The client and CA independently hash the name, account key, and
account URL. That gives offline proof, binds one account and name, and supports
rollover through prior-key matching." (45 sec)

**Transition:** "That's the construction. Here's the policy consequence."

---

## Slide 5: Mandatory Hashed-URI Binding (1:30, the point, protect)

"The published -01 had a raw account URL plus optional alternative URIs that
identified one account. The post-01 working copy then grew the 3a, 3b, and 3c
labels, including the subscriber-account form. The current work redefines 3b as
the client-computed hash and removes the other two.

The current -02 direction collapses to one mode. We drop 3a because an unbound
raw URL next to a bound hashed URI leaves a downgrade path. We drop 3c because
the subscriber-account capability remains available under CA/B Forum method
3.2.2.4.22 and a CA's CP/CPS. The hashed URI is the sole accounturi form: every
CA advertises accountHashPrefix, every client computes the hash, and there is no
raw-URL path.

My ask to the room is narrow: any objection before we put this in -02?"

Say the qualifier aloud, "this is the authors' current -02 direction", so the
slide title is not minuted bare. If the room is silent: silence is not
consensus; we summarize to the list and confirm there before publishing.

Present the hashed-only core as the authors' current direction. Do not include the
`keyRolloverWindow` proposal or mandatory support for the `domain_name = "*"`
form in the core package; those remain separate design questions.

**[IF SHORT]:** Show the table. "Drop 3a and 3c; make the hashed URI mandatory.
Any objection before -02?" (1:00)

**Transition:** "The default construction addresses the cryptographic concerns
raised in review. The domain-omitted path deliberately trades away one of those
properties and remains a separate decision."

---

## Slide 6: Final Construction Review (0:45)

"The core properties hold. Exact recomputation over a fixed field structure
makes length extension irrelevant, the account URL gives the RFC 8657 Section
5.4 uniqueness property, and the record's FQDN makes the default value
name-specific.

Four review edits remain. Use the unpadded base64url thumbprint ACME libraries
already produce. Make sha-256 mandatory and use RFC 6920 names for future
algorithms. Bound prior-key acceptance by time without putting a key hint in
the record. This does not cap the number of keys rotated within the window;
CAs still need operational rollover-rate controls to bound worst-case work.
Retention is its own CA obligation: account-to-key associations are kept and
disclosed in the CP/CPS, separate from the acceptance window.
Finally, say plainly that HTTP-01 exposes the account-key thumbprint on the
CA's cleartext validation fetch, so this is best-effort anti-correlation, not
strong unlinkability."

**[IF SHORT]:** "The crypto core holds. Four final edits: thumbprint encoding,
algorithm tokens with a ≥256-bit non-truncated floor, time-bounded prior-key
acceptance, and the privacy limit: HTTP-01 exposes the thumbprint, so hashing
resists bulk correlation but does not give unlinkability." (20 sec)

**Transition:** "That leaves a small, explicit issue disposition."

---

## Slide 7: What Lands in `-02`, and Why (0:45)

"Four settled, scoped fixes land now. Issue 38 stops clients from provisioning
an already-expired record. Issue 42 separates DNS cache lifetime from the
validation's authorization lifetime. Issue 56 aligns the challenge object with
ACME JSON naming. Issue 60 lets clients discover issuer names before creating an
order.

Three additions defer past -02. IP validation is now permitted by CA/B Forum
policy but stays out to keep this revision focused. Subdomain prompting needs a
mechanism choice, and dedicated errors need a registry-scope decision. Issues 59
and 64 close when PR #67 lands. Issue 61 stays open because the privacy analysis
is not finished."

**[IF SHORT]:** "Four scoped corrections land. Three additions defer.
Issues 59 and 64 close with PR #67; privacy issue 61 stays open." (15 sec)

---

## Slide 8: Path Forward (1:00)

"PR #68 has merged into the PR #67 candidate. The candidate now includes the
rotation-text rewrite, the RFC 8657 distinction, the four construction-review
edits, and the four ready issue fixes. IANA early review has been requested for
the new directory metadata.

The two unsettled extensions now have exact proposals under author review
rather than open-ended options. For key rotation, the proposal would replace
keyRotationPeriod with keyRolloverWindow, a required CA directory field: an
exact acceptance window after a key rotates out; record removal, persistUntil,
and account deactivation end it sooner, and certificate validity does not
extend it. For the domain-omitted star form, keep it with client-MAY, CA-MUST
semantics and the cost stated directly: the same value in every record lets
a DNS observer link the account's domains; records evaluate independently, so
no precedence rule is needed.

The draft cutoff has passed. We finish the text now, take feedback Thursday,
and publish -02 when submissions reopen. The question for the room is whether
anyone has input or objections on the two pending proposals. These are
proposals under author review, not a settled author position; we will confirm
the outcome on the list."

---

## Slide 9: Questions (2:00)

Open the floor. First, the standing chair ask: "Chairs, please minute the
core package as the authors' -02 direction with WG feedback invited, not a
consensus call; we will summarize today's input to `acme@ietf.org` and confirm
there before publishing." If an affirmative signal would help, ask the chairs
for a hum on the narrow slide 5 question. Do not treat silence as endorsement.

If quiet, ask only:

- "Any objection to dropping 3a and 3c and making the hashed URI mandatory?"
- "Any objections to the proposed keyRolloverWindow semantics or to keeping the
  domain-omitted star form with client-MAY, CA-MUST support?"

If CA/Browser Forum alignment comes up, the slide lists four open
cabforum/servercert issues tracking the Method 3.2.2.4.22 side: 674
(one-to-many accounturis), 675 (parameter-key case-sensitivity), 676
(adopting policy=wildcard), and 677 (extracting dns-persist-01 into its
own validation method). BR reconciliation routes through CABF, not this
draft.

---

## Likely Questions

### Do all three authors agree on keyRolloverWindow and the star form?

No. The settled fixes are merged, and Henry is good with them. On rollover, he
argues that records need not break and wants to explore changing the behavior if
the authors agree. The exact rollover semantics and the star form remain under
author review. We will use WG input to continue that discussion and confirm the
outcome on the list.

### Why not just use the pubkey directly in the record (PR #65)?

That baked SHA-256 into a new consumer-visible `pubkey:` wire format and broke
key-rotation durability. The PR #67 construction instead puts a hash-algorithm token
under the CA's HTTPS `accountHashPrefix`, so the CA and client can migrate
algorithms without defining a public-key URI scheme.

### Why not an online proof-of-possession POST (PR #62)?

Sound idea, and it survives key rotation, but it requires accounturi to be a
live POST-able endpoint, which only holds for mode 3a, and it breaks
pre-provisioning where there's no reachable endpoint before first issuance. The
thread preferred an offline computed value over a live exchange. Mode 3c is no
longer part of the draft.

### Does binding the key break key rotation?

Binding changes the hash when the account key changes. The PR #67 candidate
preserves continuity by matching retained prior keys, with account deactivation
as the hard override. The `keyRolloverWindow` proposal would set an exact upper
bound on prior-key acceptance; clients would republish after rotation, and the old
record would stop validating when the window ends.

Henry has raised a different direction in which the DNS record remains valid
across account key rollover without a bounded prior-key window. That direction
must preserve the non-transferability property supplied by key binding, so the
mechanism needs more design work. We will take WG input and continue the author
discussion.

### I steal a rotated-out account key. What does the window let me do before T + P?

Nothing without the current key. A rotated-out key can no longer authenticate
ACME requests (RFC 8555 Section 7.3.5), so it cannot act as the victim's
account. Registering a new account with the stolen key yields a different
account URL, so the CA's recomputation never matches the victim's published
digest, because the hash binds key and account URL together. The window preserves
validation continuity for the legitimate account only; account deactivation
remains the hard stop for a current-key compromise.

### The slide says certificate validity does not extend the window, but the PR #67 candidate still has certificate-backed retention. Which is it?

The PR #67 candidate still carries the -01-era certificate-backed exception.
The replacement is a drafted follow-up patch, deliberately unapplied until the
co-authors agree and the WG has seen the proposal. The exception is being
removed because it made the claimed upper bound ineffective: a renewal using
the same old-key hash could extend acceptance again. If agreement is not
reached, -02 carries the current keyRotationPeriod text and the proposal goes
to the list.

### You bound prior-key age but not count. What is the worst-case verification work?

Per retained key the CA performs two SHA-256 computations over short inputs,
so CPU is negligible. Key rollover is an authenticated, CA-mediated operation,
so a CA can rate-limit it and bound the retained set. The real operational
cost is retention storage plus CP/CPS disclosure of the window and retention
policy.

### Why can the outer account-URI hash change while the JWK thumbprint stays on SHA-256?

RFC 8555 Section 8.1 defines the ACME key-authorization component using the
unpadded base64url SHA-256 RFC 7638 JWK Thumbprint. This construction reuses
that existing component. Making the inner thumbprint follow `<hash-alg>` would
define another thumbprint profile for every optional outer hash and prevent
clients from reusing ACME's existing value. Any migration should be coordinated
across ACME rather than happen implicitly when the outer hash changes.

### Why isn't the rollover window an RFC 8555 mechanism?

The related list discussion is about authorization lifetime and termination
under 8555. This window bounds a property only dns-persist has: its records
embed the account key, so the DNS authorization needs explicit prior-key
acceptance after rollover. RFC 8555 authorizations bind to the account rather
than a particular account key, so they raise no equivalent rollover question.
Keep the field here, note the relationship in Security Considerations, and let
a future RFC 8555 termination mechanism compose with it.

### What if a domain-bound record and a star record disagree on wildcard policy?

Records are independent grants evaluated atomically: a request succeeds only
if a single record's accounturi, persistUntil, and policy all cover it. Only
the record carrying policy=wildcard can authorize a wildcard request; the
other record is not overridden, it simply is not a grant for that shape.
Grants union, as with multiple CAA issue records, and parameters never
combine across records. Restricting means removing the broad record or
letting its persistUntil retire it. Precedence also cannot work mechanically:
hashes are opaque, and with one-to-many accounturis one account's narrow
grant must not veto another account's wildcard grant.

### Why must every CA accept the star form rather than MAY?

There is no negotiation mechanism for a client to discover per-CA acceptance,
and the draft defines none, so an optional form would fail unpredictably across
CAs. The opt-out is chosen by the client, per record, and its privacy cost
falls only on the account that opts out. The use case is one record reused
across many names, such as a hosting fleet under one account.

### Where are the hash-algorithm tokens registered?

The URI remains under the CA's HTTPS `accountHashPrefix`, so no new URI scheme is
created. The PR #67 candidate requires `sha-256` and draws future token names
from the RFC 6920 Named Information Hash Algorithm Registry, restricted to non-truncated digests of
at least 256 bits.

### HTTP-01 already leaks the thumbprint in cleartext. What stops an attacker who knows it?

Knowing the thumbprint lets an attacker compute digests, not validate with
them. The CA recomputes only over keys the requesting account has proven at
the ACME layer, and the published digest embeds the victim's account URL, so
it can never equal a digest for the attacker's account. The leak costs
correlation resistance, a privacy loss; it does not weaken non-transferability.

### What happens to records provisioned under -01 semantics?

-02 is a breaking change to the accounturi form; the changelog states the raw
form is no longer accepted. This is a post-adoption working-group draft before
standardization: no production deployment base is assumed, so there is no
compatibility path to preserve; early deployments republish with the hashed
form. If a transition were ever needed, the dual-accept pattern already
specified for accountHashPrefix changes would extend to the form change.

### Is accountHashPrefix stable? What if the CA changes it?

The draft says CAs SHOULD keep the prefix and account-URL structure stable,
document the guarantee, and run dual acceptance during any transition, so
pre-provisioned records survive a change.

### How does this interact with EAB / External Account Binding?

With mode 3c dropped, subscriber-account grouping is out of the draft, so the
EAB interaction no longer arises here. The subscriber model remains under the
BRs and a CA's CP/CPS.

### How does dropping the raw account URL reconcile with RFC 8657's `accounturi`?

RFC 8657 Section 3 requires a URI the CA recognizes as identifying the account;
Section 3.1 requires ACME CAs to recognize the account object URI, but does not
forbid an additional CA-recognized URI. Our draft-side fix says the CAA
`accounturi` is the account URL matched during CAA processing, while the
dns-persist `accounturi` is a hashed URI matched by recomputation in a different
record. They need not share a value. The BR and CP/CPS reconciliation remains a
CABF work item.

### Where is the draft version?

The working copy is post-01 with the PR #66 merge; -02 will carry the
substitution mechanism. No new version is posted to the Datatracker since -01,
during the submission blackout. -02 goes out after the meeting, incorporating
WG feedback.
