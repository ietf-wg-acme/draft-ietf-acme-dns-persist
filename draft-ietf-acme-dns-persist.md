---
title: "Automated Certificate Management Environment (ACME) Challenge for Persistent DNS TXT Record Validation"
abbrev: "ACME Persistent DNS Challenge"
category: std
submissionType: IETF
docname: draft-ietf-acme-dns-persist-latest
ipr: trust200902
area: "Security"
workgroup: "Automated Certificate Management Environment"
keyword:
 - acme
 - dns
 - validation
 - persistent

venue:
  group: "Automated Certificate Management Environment"
  type: "Working Group"
  mail: "acme@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/acme/"
  github: "ietf-wg-acme/draft-ietf-acme-dns-persist"

author:
 -
   name: "Shiloh Heurich"
   org: Fastly
   email: "sheurich@fastly.com"
 -
   name: "Henry Birge-Lee"
   org: "Crosslayer Labs, Inc."
   email: "henry@crosslayerlabs.com"
 -
   name: "Michael Slaughter"
   org: "Amazon Trust Services"
   email: "slghtr@amazon.com"

normative:
  FIPS180-4:
    title: "Secure Hash Standard (SHS)"
    author:
      org: "National Institute of Standards and Technology (NIST)"
    date: 2015-08
    seriesinfo:
      "FIPS PUB": "180-4"
    target: https://doi.org/10.6028/NIST.FIPS.180-4

informative:
  birgelee-sc082-security:
    title: "Security of SC-082 Redux"
    author:
      -
        ins: H. Birge-Lee
        name: Henry Birge-Lee
        org: "Princeton University"
    date: 2025
  cabf-br:
    title: "Baseline Requirements for the Issuance and Management of Publicly-Trusted TLS Server Certificates"
    target: https://cabforum.org/baseline-requirements-documents/
    date: 2025

--- abstract

This document specifies "dns-persist-01", an Automated Certificate Management Environment (ACME) challenge that verifies domain or IP address control using a persistent DNS TXT record. The record identifies a certification authority (CA) and binds validation to an ACME account. It can be published before certificate requests and reused without per-request DNS changes, including when different parties perform DNS publication and ACME operation. Use under a certificate policy or trust program remains subject to that policy's requirements.

--- middle

# Introduction {#introduction}

The Automated Certificate Management Environment (ACME) protocol {{!RFC8555}} defines mechanisms for automating certificate issuance and domain validation. For a new "http-01" or "dns-01" challenge, the ACME client publishes a response derived from a per-challenge token. Preparing that response can be difficult when the client cannot promptly update the domain's HTTP or DNS infrastructure.

Examples include:

- Edge compute and multi-tenant hosting platforms where the entity managing the DNS zone is distinct from the tenant subscribing to the certificate.
- Organizations that wish to prepare DNS validation records before later certificate requests.
- Environments with strict change management processes where DNS modifications require approval workflows.
- Deployments requiring repeated wildcard issuance without repeated DNS changes.
- Internet of Things (IoT) deployments where devices may not be able to host an HTTP service or coordinate DNS updates during challenge fulfillment.

This document defines a new ACME challenge type, "dns-persist-01". This method proves control over a fully qualified domain name (FQDN) by confirming the presence of a persistent DNS TXT record containing certification authority (CA) and account identification information. Because the record's value is derived from account, domain, and CA parameters rather than from a per-challenge token, the record can be published before any certificate request and by a party other than the ACME client (see {{pre-provisioning-records}}).

The record format is based on the "issue-value" syntax of Certification Authority Authorization (CAA) records {{!RFC8659}}, incorporating an `issuer-domain-name` and a mandatory `accounturi` parameter whose value is defined in {{validation-record-format}}. In `dns-persist-01`, the `accounturi` value is a hashed Uniform Resource Identifier (URI) that cryptographically binds the account to the domain being validated.

## Policy Applicability {#robustness-and-alignment}

The dedicated label and the account binding do not by themselves establish compliance with a CA's certificate policy or applicable trust program requirements. Those requirements determine whether the method is permitted and which additional controls apply.

# Conventions and Definitions {#conventions-and-definitions}

{::boilerplate bcp14-tagged}

**DNS TXT Record Persistent DCV Domain Label**
:   The label "_validation-persist" as specified in this document. In this term, DCV means domain control validation.

**Validation Domain Name**
:   The domain name that the CA queries for the validation TXT record. For `dns` identifiers, it is formed by prepending the DNS TXT Record Persistent DCV Domain Label to the FQDN being validated (`_validation-persist.<R>`). For `ip` identifiers ({{ip-address-validation}}), it is formed by prepending `"_ip-validation-persist"` to the normalized Reverse Zone Domain Name (`_ip-validation-persist.<R>`, see {{ip-reverse-zone-mapping}}).

    This term follows the precedent set by {{!RFC8555}}, Section 8.4, which uses "validation domain name" for the analogous `_acme-challenge.<FQDN>` label in the dns-01 challenge. This document avoids the term "Authorization Domain Name" because the CA/Browser Forum Baseline Requirements {{cabf-br}} define it as the FQDN used to obtain authorization, without any label prepended.

**Issuer Domain Name**
:   A domain name disclosed by the CA in Section 4.2 of the CA's Certificate Policy and/or Certification Practices Statement to identify the CA for the purposes of this validation method.

    The directory and challenge-object fields that carry Issuer Domain Names, and their relationship to `caaIdentities`, are defined in {{directory-issuer-domain-names}} and {{challenge-object}}.

**Validation Data Reuse Period**
:   The period during which a CA may rely on validation data, as defined by the CA's practices and applicable requirements.

**persistUntil**
:   An optional parameter in the validation record that specifies the timestamp after which CAs no longer consider the validation record valid for new validation attempts. The value MUST be a base-10 encoded integer representing a UNIX timestamp (the number of seconds since 1970-01-01T00:00:00Z ignoring leap seconds). Client pre-flight guidance appears in {{client-implementation-guidelines}}.

This document follows {{!RFC8555}} in attributing DNS provisioning actions to the ACME client. A DNS action attributed to the client MAY instead be performed by another party acting on behalf of the domain owner, such as a DNS operator. This permission does not extend to a requirement that explicitly involves an ACME message exchange or the account's private key. {{pre-provisioning-records}} describes the values such a party needs. Requirements on the content of `_validation-persist` records apply to the published record regardless of which party publishes it.

# The "dns-persist-01" Challenge {#dns-persist-01-challenge}

The `dns-persist-01` challenge demonstrates control using a TXT record at the Validation Domain Name. The record identifies the CA and requesting account and can be reused for later validations.

## Challenge Object {#challenge-object}

In addition to the common challenge object fields defined in {{!RFC8555}}, Section 8 (including the optional `validated` and `error` fields), the challenge object for "dns-persist-01" contains the following fields:

- **type** (required, string): The string "dns-persist-01"
- **url** (required, string): The URL to which a response can be posted
- **status** (required, string): The status of this challenge
- **issuerDomainNames** (required, array of strings): A list of one or more Issuer Domain Names. The client MUST choose one of these domain names to include in the DNS TXT record. The challenge is successful if a valid TXT record is found that uses any one of the provided domain names.

  Each string in the array MUST be a domain name that complies with the following normalization rules:

  1.  The domain name MUST be represented in A-label format (Punycode, {{!RFC5890}}).
  2.  All characters MUST be lowercase.
  3.  The domain name MUST NOT have a trailing dot.

  The server MUST ensure the array is not empty. Servers MUST NOT send more than 10 Issuer Domain Names. This limit serves as a practical measure to prevent denial-of-service vectors against clients. Clients MUST consider a challenge malformed if the `issuerDomainNames` array is empty or if it contains more than 10 entries, and MUST reject such challenges. Each domain name MUST NOT exceed 253 octets in length.

The following shows an example challenge object:

~~~json
{
  "type": "dns-persist-01",
  "url": "https://ca.example/acme/chall/5678",
  "status": "pending",
  "issuerDomainNames": ["authority.example", "ca.example.net"]
}
~~~
{: #fig-challenge-object title="Example dns-persist-01 Challenge Object"}

## Directory Metadata for Issuer Domain Names {#directory-issuer-domain-names}

A CA offering the `dns-persist-01` challenge type MUST advertise an `issuerDomainNames` array in the `meta` object of its directory ({{!RFC8555}}, Section 7.1.1). This directory-level array lets a client discover Issuer Domain Names before opening any authorization, so it can pre-provision `_validation-persist` records per {{pre-provisioning-records}} without an interactive order.

The directory `issuerDomainNames` array MUST use the same normalization and length rules as the challenge object's `issuerDomainNames` array ({{challenge-object}}): each entry MUST be a lowercase A-label domain name with no trailing dot and MUST NOT exceed 253 octets. The array MUST NOT be empty and MUST NOT contain more than 10 entries.

Let D be the set of values in the directory `issuerDomainNames` array, and let I be the set of values in the directory `caaIdentities` array ({{!RFC8555}}, Section 7.1.1), represented as lowercase A-label domain names with no trailing dot. This document does not require a CA to advertise `caaIdentities`; a CA that does not perform CAA validation has no names to disclose there. For each `dns-persist-01` challenge object the CA issues, let C be the set of values in that challenge's `issuerDomainNames` array. The CA MUST satisfy these requirements:

- Every value in D MUST appear in C. A client that pre-provisions a record using a name from D can therefore use that name to satisfy the `issuer-domain-name` requirement of later `dns-persist-01` challenges from that CA while the name remains advertised, subject to the stability requirement and migration-period guidance in {{pre-provisioning-records}}.
- C MAY include values beyond D while remaining within the 10-entry maximum in {{challenge-object}}, so a CA can offer challenge-specific names in addition to its pre-provisioning baseline.
- If the CA advertises `caaIdentities`, every value in C MUST also appear in I. This prevents a CA that participates in CAA validation from accepting an `issuer-domain-name` it has not also disclosed through `caaIdentities`.

For these subset comparisons, the CA MUST compare `caaIdentities` values, when advertised, after applying the same lowercase A-label and trailing-dot rules used for D and C.

CA disclosure identifies the Issuer Domain Names a CA recognizes; it does not establish that each name is recognized by only one CA. For example, {{!RFC8657}}, Section 5.4, discusses a parent CA and a subsidiary that recognize a shared name and requires account URIs to be unambiguous within the relevant recognition scope. The `issuer-domain-name` check therefore does not replace the account-binding check in {{verification-procedure}}.

A client MUST treat a missing or nonconforming directory `issuerDomainNames` array as unavailable for pre-provisioning. A missing or nonconforming `caaIdentities` array does not affect pre-provisioning; it only makes the I comparisons unavailable. A client MAY still process an interactive `dns-persist-01` challenge. When D is available, the client MUST consider a challenge malformed if any value in D is absent from C. When I is available, the client MUST consider a challenge malformed if any value in C is absent from I.

# Challenge Response and Verification {#challenge-response-and-verification}

To respond to the challenge, the ACME client ensures that a DNS TXT record is provisioned at the Validation Domain Name of the domain being validated. Let `A` be the domain name in the `identifier.value` field of the ACME authorization object ({{!RFC8555}}, Section 7.1.4). For wildcard authorizations, `A` has the `*.` prefix removed per {{!RFC8555}}, Section 7.1.3. Let `R` be the FQDN whose Validation Domain Name the CA queries (the "domain being validated"). By default, `R` is `A`. When validating a subdomain or deeper wildcard authorization using a persistent record at a parent domain ({{subdomain-certificate-validation}}), `R` is the ancestor domain selected by the client. The Validation Domain Name is formed by prepending the label "_validation-persist" to `R`.

For example, if `R` is "example.com", the Validation Domain Name is "_validation-persist.example.com".

The client indicates it is ready for validation by POSTing a JSON response object to the challenge URL ({{!RFC8555}}, Section 7.5.1):

- To validate at the authorization's own identifier (`R = A`), the client POSTs an empty JSON object (`{}`).
- To validate against a persistent record provisioned at a proper ancestor of `A`, the client includes the optional `ancestorDomain` field (string) in the response payload (e.g., `{"ancestorDomain": "example.com"}`), setting `R` to that ancestor domain subject to the requirements in {{subdomain-certificate-validation}}.

The `ancestorDomain` field is defined only for the client's challenge response payload; the server does not add `ancestorDomain` to the challenge object resource ({{challenge-object}}). The first valid challenge-response POST accepted for a challenge fixes `R` for that validation attempt. While that attempt is outstanding (`pending` after acceptance or `processing`), a repeated POST carrying the identical `ancestorDomain` value (or identical `{}`) is idempotent. A CA MUST reject with an HTTP `malformed` error ({{!RFC8555}}, Section 6.7) any POST to an accepted `pending`, `processing`, or already-finalized challenge that attempts to change, add, or remove `ancestorDomain`. The CA MUST NOT overwrite the domain `R` associated with an in-flight or completed validation attempt. When the CA receives a challenge-response POST to a challenge that is already `valid` or `invalid`, it starts no new validation attempt and changes neither `R` nor the challenge state. The CA's response to such a POST is otherwise as specified in {{!RFC8555}}.

## Validation Record Format {#validation-record-format}

The RDATA of this TXT record MUST fulfill the following requirements:

1.  The RDATA value MUST conform to the issue-value syntax defined in {{!RFC8659}}, Section 4.2. CAs MUST recognize the parameter names `accounturi`, `policy`, and `persistUntil` using ASCII case-insensitive comparison. This comparison applies only to parameter names; the value-comparison rules below are unchanged. Each of these recognized parameters MUST occur at most once within a single issue-value. If a recognized parameter occurs more than once, including with different casing, the CA MUST treat the record as malformed and reject it. To ensure forward compatibility, the server MUST ignore parameters whose names it does not recognize, including repeated occurrences of such parameters.

2.  The CA MUST parse the issue-value according to {{!RFC8659}}, Section 4.2, before comparing the issuer name. Surrounding whitespace permitted by that syntax is not part of the issuer name. The parsed `issuer-domain-name` MUST match an entry in the challenge object's `issuerDomainNames` array using Simple String Comparison ({{!RFC3986}}, Section 6.2.1). The CA MUST NOT apply case folding or other normalization when comparing these strings. If no entry matches, the CA MUST reject the record.

3.  The issue-value MUST contain an `accounturi` parameter whose value is a hashed URI identifying the ACME account requesting validation. A hashed URI cryptographically binds the account to the domain being validated without publishing the account URL in cleartext. It is formed by concatenating the CA's `accountHashPrefix`, a hash-algorithm identifier, a "/" separator, and a base64url hash value:

    ~~~
    <accountHashPrefix><hash-alg>/<base64url hash value>
    ~~~

    A CA that advertises support for the dns-persist-01 challenge type MUST advertise an `accountHashPrefix` string in the `meta` object of its directory ({{!RFC8555}}, Section 7.1.1). This prefix places hashed URIs under the CA's infrastructure and lets a client construct them from the directory without a per-provisioning request to the CA. The CA MUST choose a prefix such that, for each hash algorithm it accepts, appending the remaining components yields both a valid URI ({{!RFC3986}}) and a value conforming to the `value` production of the issue-value grammar in {{!RFC8659}}, Section 4.2. An example of such a prefix is `https://ca.example/account-hash/`. Clients and CAs MUST use the advertised prefix unchanged when constructing the hashed URI, including any percent-encoded octets. Before constructing a hashed URI, a client and any party computing or publishing a record on its behalf MUST verify that `accountHashPrefix` is present and contains only octets allowed by the `value` production of {{!RFC8659}}, Section 4.2. Each such party MUST also verify that the prefix contains only `%` and characters in the `reserved` and `unreserved` sets of {{!RFC3986}}, Sections 2.2 and 2.3; this excludes `\` and `"`, which DNS master-file syntax ({{!RFC1035}}, Section 5.1) treats as escape and quoting characters. If the prefix is missing or fails either check, dns-persist-01 is unavailable at that CA. The client MUST NOT construct a record, and it MUST NOT substitute another value, such as the account URL. A prefix containing `;` would otherwise make the client's own digest parse as a separate parameter that the CA ignores (item 1).

    `<hash-alg>` MUST be the exact Hash Name String registered in the "Named Information Hash Algorithm Registry" defined by {{!RFC6920}}, Section 9.4. Both clients and CAs implementing dns-persist-01 MUST support the registered `sha-256` Hash Name String, which identifies SHA-256 {{FIPS180-4}}. A client MUST be able to compute, and a CA MUST accept, a hashed URI that uses this token, so that every conforming implementation interoperates without prior negotiation. A CA MAY accept other registered algorithms and MUST document each additional algorithm it accepts. An additional algorithm MUST have a registered digest length of at least 256 bits and MUST remain collision resistant at the time of use. A client MAY use an optional algorithm only after establishing from the CA's documentation that the CA accepts it. This document defines no separate algorithm-negotiation mechanism.

    A CA MUST reject with a `malformed` error ({{!RFC8555}}, Section 6.7) any `_validation-persist` record whose `<hash-alg>` token identifies a hash algorithm the CA no longer accepts, including records the CA previously validated under that algorithm. This rejection applies to the individual record; when several records are retrieved, the challenge-level error follows {{error-handling}}. If a CA deprecates or discontinues acceptance of a hash algorithm, domain owners MUST re-provision records using an accepted hash algorithm.

    `<base64url hash value>` is the base64url encoding {{!RFC4648}}, with trailing padding (`=`) omitted, of the hash digest computed with the identified algorithm over the following octet string:

    ~~~
    H(length_of_domain || domain_name || key || account_URL)
    ~~~

    where `H` is the identified hash algorithm and `||` denotes octet-string concatenation. The inputs are:

    1. `length_of_domain` is a single octet giving the length in octets of `domain_name`. A domain name is limited to 255 octets ({{!RFC1035}}, Section 2.3.4), so a single octet always suffices. This length prefix delimits `domain_name` from the fields that follow.
    2. `domain_name` is, by default, the FQDN being validated: the Validation Domain Name with its leading `_validation-persist` label removed and no trailing dot, in the lowercase ASCII presentation form produced by {{normalization-algorithm}}, using validated A-labels for internationalized labels. This is `R`, the FQDN whose Validation Domain Name the CA queries ({{normalization-algorithm}}), not the FQDN of the certificate ultimately requested; binding the record to its own domain is what makes the hashed URI domain-specific (see {{hashed-uri-security}}).

       As an exception, a client MAY set `domain_name` to the single US-ASCII octet `*` (0x2a; `length_of_domain` is then 0x01) to opt out of the domain-correlation mitigation. The resulting hashed URI does not depend on the domain and MAY therefore be reused across domains. A CA MUST accept a hashed URI computed with `domain_name` set to `*`. The privacy trade-off of this opt-out is described in {{hashed-uri-security}}.
    3. `key` is the 43 ASCII octets of the unpadded base64url encoding {{!RFC4648}} of the SHA-256 JSON Web Key (JWK) Thumbprint {{!RFC7638}} of a public key associated with the ACME account. This is the encoding used for the thumbprint component of an ACME key authorization ({{!RFC8555}}, Section 8.1), so a client can reuse that component without decoding it. When provisioning a `_validation-persist` record, the client MUST use the account's current key. A CA MUST accept a hashed URI generated with the current key associated with the account. A CA MUST also accept a hashed URI generated with a prior key previously associated with the account that is known to the CA per the key retention obligation in {{verification-procedure}}, subject to the deactivation and retention rules stated there. This allows a record provisioned before a key rotation to continue to validate. The rationale for these requirements is given in {{hashed-uri-security}}.
    4. `account_URL` is the ACME account URL ({{!RFC8555}}, Section 7.3). A URI is US-ASCII per {{!RFC3986}}, so it is encoded as those US-ASCII octets.

    {{fig-hashed-uri-construction}} illustrates the hash preimage layout and resulting `accounturi` structure:

    ~~~
    |<------------------------ Hash Preimage ------------------------>|
    +------------------+--------------------+------------+------------+
    | length_of_domain | domain_name        | key        | account_URL|
    | 1 octet          | 1..253 ASCII octets| 43 ASCII   | Variable   |
    | (0x01..0xfd)     | (A-label or "*")   | octets     | ASCII      |
    +------------------+--------------------+------------+------------+
                                     |
                                     v  H(...) (e.g., sha-256) +
                                     |  unpadded base64url
                                     v
        <accountHashPrefix><hash-alg>/<base64url hash value>
    ~~~
    {: #fig-hashed-uri-construction title="Hashed-URI Preimage and Construction"}

    The CA MUST verify that the hashed URI in the DNS record authorizes the ACME account making the request, by recomputing it as described in {{verification-procedure}}; if no recomputation matches, the CA MUST reject the record.

4.  The issue-value MAY contain a `policy` parameter. If present, this parameter modifies the validation scope. The `policy` parameter follows the `tag=value` syntax from {{!RFC8659}}. Comparison of the values defined for this parameter MUST be case-insensitive (e.g., "WILDCARD" and "wildcard" are equivalent).

    Note: The requirement to ignore unrecognized parameters (item 1) ensures forward compatibility, allowing future extensions without breaking existing implementations, consistent with ACME's extensibility model ({{!RFC8555}}). Item 1 governs unrecognized parameter names. The rule below governs unrecognized `policy` values.

    The following value for the `policy` parameter is defined with respect to subdomain and wildcard validation:

    - `policy=wildcard`: If this value is present, the CA MAY consider this validation sufficient for issuing certificates for the validated FQDN, for specific subdomains of the validated FQDN (as covered by wildcard scope or specific subdomain validation rules), and for wildcard certificates (e.g., `*.example.com`). See {{wildcard-certificate-validation}} and {{subdomain-certificate-validation}}.

    If the `policy` parameter is absent, or if its value is anything other than `wildcard`, the CA MUST proceed as if the `policy` parameter were not present (i.e., the validation applies only to the specific FQDN).

5.  The issue-value MAY contain a `persistUntil` parameter. If present, the value MUST consist of one or more ASCII digits (`0` through `9`), with no sign or other characters, and represents a UNIX timestamp (the number of seconds since 1970-01-01T00:00:00Z ignoring leap seconds). If the value does not have this form, or exceeds 9223372036854775807 (2^63 - 1), the CA MUST treat the record as malformed and reject it.

    When a CA validates a `_validation-persist` record containing a `persistUntil` parameter with timestamp T, the CA MUST NOT set the `expires` field of the resulting authorization object ({{!RFC8555}}, Section 7.1.4) later than T, and MUST NOT rely on that validation to issue certificates after T. The record is valid for a validation attempt performed at or before T; a CA MUST NOT consider it valid for a validation attempt performed after T. Already-issued certificates are unaffected by the expiration of a `persistUntil` timestamp.

    When more than one record satisfies all requirements, the CA relies on one of them and applies only that record's `persistUntil`; the CA MUST NOT combine parameters from different records. Relying on the record that permits the latest expiration (one without `persistUntil`, otherwise the one with the greatest T) makes the result independent of record order.

This specification defines the following case-handling rules for parameter values in dns-persist-01 records:

- `accounturi`: The value is a hashed URI. CAs MUST compare `accounturi` values using Simple String Comparison per {{!RFC3986}}, Section 6.2.1, with no case-folding or other normalization.
- `policy`: Case-insensitive, as specified in item 4 above.
- `persistUntil`: The value is a base-10 integer. Case does not apply.

The following TXT record validates `example.com` using Issuer Domain Name `authority.example` and the hashed URI computed in {{hashed-uri-example}}:

~~~ dns
_validation-persist.example.com. IN TXT ("authority.example;"
" accounturi=https://ca.example/account-hash/"
"sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw")
~~~
{: #ex-basic-validation title="Basic Validation TXT Record"}

## Verification Procedure {#verification-procedure}

The ACME server determines the domain being validated (`R`) from the challenge response as specified in {{challenge-response-and-verification}} and {{subdomain-certificate-validation}}, then verifies the challenge by performing a DNS lookup for TXT records at the Validation Domain Name (`_validation-persist.<R>`). It iterates through the returned records to find one that conforms to the required structure. For a record to be considered valid, its `issuer-domain-name` value MUST match one of the values provided in the `issuerDomainNames` array from the challenge object. To be considered valid, the record's `accounturi` parameter MUST also be a hashed URI that identifies the requesting account. {{validation-record-format}} defines both requirements. The server MUST compare Issuer Domain Names using the Simple String Comparison specified in {{validation-record-format}}. The server also enforces any `policy` parameter requirements according to {{wildcard-certificate-validation}} and {{subdomain-certificate-validation}}. In particular, the record MUST contain `policy=wildcard` in each of the following cases:

- the authorization object has `wildcard` set to `true` ({{!RFC8555}}, Section 7.1.4);
- `subdomainAuthAllowed` is set to `true` ({{!RFC9444}}, Section 4.1); or
- `ancestorDomain` was supplied in the challenge response (`R` is a proper ancestor of `A`).

If no record meeting all requirements is found, the server MUST treat the challenge as failed.

To verify a hashed URI `accounturi` value, the CA uses the requesting account's URL and the domain being validated (`R`). The CA computes the expected hashed URI as defined in {{validation-record-format}} over each ACME public key it retains for that account (see below). For each such key, it computes the value twice: once using `R` as `domain_name` and once using the domain-omitted opt-out form (`domain_name` set to `*`). The CA treats the record as satisfying the requirement if its `accounturi` value equals any of these computed hashed URIs under Simple String Comparison ({{!RFC3986}}, Section 6.2.1). Retaining an account's prior key thumbprints allows records provisioned before a key rotation to continue validating.

To recompute these hashed URIs, a CA has a data-retention obligation: it MUST retain an association between each ACME account and the SHA-256 JWK Thumbprints of all public keys that account has used since the CA began accepting key-bound `dns-persist-01` records. These keys are the key bound to the account when the CA began accepting key-bound `dns-persist-01` records (or when the account was created, if later) and every key bound to the account afterward (for example, by key rollover; {{!RFC8555}}, Section 7.3.5); the CA need not reconstruct keys retired before that time. The following rules govern this retained data and the acceptance of the keys behind it, listed in order of precedence with earlier rules overriding later ones:

1. **Deactivation overrides retention.** A CA MUST NOT accept a hashed URI computed from a key of an ACME account whose status is not `valid` (deactivated or revoked; {{!RFC8555}}, Section 7.1.2), regardless of any retention rule below. Deactivating an account ({{!RFC8555}}, Section 7.3.6) stops all of that account's `_validation-persist` records from validating immediately, which is the intended behavior for a suspected key or account compromise. The CA MUST evaluate account status from live account state at each validation attempt; an account proof or status observed at record publication time MUST NOT substitute for live validation-time account evaluation. Once an account is deactivated, the CA MAY prune all retained key thumbprints for that account.
2. **Witnessed key retention.** Subject to rule 1, a CA MUST retain the SHA-256 JWK Thumbprint of every public key an account has used, bounded only by the lifetime of the account, so that a record computed under any of those keys continues to validate across routine key rotations.

A CA MAY cache computed or accepted hashed URIs; such caching MUST NOT change any acceptance outcome.

The CA MUST evaluate `persistUntil`, `policy`, `issuer-domain-name`, and any unrecognized parameters from fresh DNS data retrieved during each validation attempt. No previously observed or cached value of these fields overrides the values read during the current validation.

When recomputing expected hashed URIs over retained keys, the CA MUST evaluate the account's current key first, and SHOULD evaluate remaining retained keys in reverse chronological order of their use.

## Multiple Issuer Support {#multiple-issuer-support}

A domain MAY authorize multiple CAs by provisioning a separate `_validation-persist` TXT record for each issuer.

### Coexistence of Records {#multiple-issuer-coexistence}

When multiple TXT records are present at the same DNS label (e.g., `_validation-persist.example.com`), each record functions as an independent authorization for the specified issuer. This follows a similar pattern to CAA records {{!RFC8659}}, where multiple records at the same label are permissible.

### CA Verification Process {#multiple-issuer-verification}

When a CA performs validation for a domain with multiple `_validation-persist` TXT records, it MUST follow these steps:

1.  **Query DNS**: Retrieve all TXT records from the Validation Domain Name.
2.  **Filter Records**: Iterate through the returned records to find those where the `issuer-domain-name` value matches, using the comparison specified in {{validation-record-format}}, one of the Issuer Domain Names the CA is configured to use for this validation. The CA MUST ignore all other records.
3.  **Validate Record**: For each matching record, the CA proceeds to validate it according to the requirements in this specification, including verifying the `accounturi` and `persistUntil` parameters. If any matching record satisfies all requirements, the validation succeeds.
4.  **Handle No Match**: If no record with a matching `issuer-domain-name` is found, or if no matching record satisfies all validation requirements, the validation attempt MUST fail.

### Security and Management Considerations {#multiple-issuer-management}

Domain owners authorizing multiple issuers should consider:

**Auditing**
:   Regularly audit DNS records so that only intended CAs remain authorized. Remove records for CAs that are no longer in use.

**Issuer and Account Checks**
:   Each record is checked against the validating CA's Issuer Domain Names and the requesting account, as specified in {{verification-procedure}}. These checks do not establish that the authorized CAs have independent infrastructure.

**Weakest Link**
:   The domain's overall security posture is influenced by the security practices of all authorized CAs. Domain owners should consider the practices of each CA they authorize.

**Authorization Removal**
:   To de-authorize a CA, the domain owner (or its delegated DNS operator) MUST delete the corresponding TXT record from the DNS zone.

### Example: Authorizing Two CAs {#multiple-issuer-example}

This example shows a domain owner authorizing two CAs, "ca1.example" and "ca2.example", to issue certificates for `example.org`. Each record uses `domain_name` `example.org` and the account key thumbprint from {{hashed-uri-example}}. The account URLs are `https://ca1.example/acct/123` at CA1 and `https://ca2.example/acct/456` at CA2.

~~~dns
_validation-persist.example.org. 3600 IN TXT ("ca1.example;"
" accounturi=https://ca1.example/account-hash/"
"sha-256/qGV7z_u4K-mcpnqe1n9rforNnuQWLcNZTidyrtOhWac;"
" policy=wildcard")
_validation-persist.example.org. 3600 IN TXT ("ca2.example;"
" accounturi=https://ca2.example/account-hash/"
"sha-256/FoiSuymBL2Pk3iIpv5oPODm9wrZGRzLXBohP1QHVc2Y;"
" persistUntil=1924992000")
~~~
{: #ex-multiple-ca-auth title="Multiple CA Authorization Records"}

Each CA retrieves both records and ignores the other CA's record. CA1 evaluates its record with the `policy=wildcard` scope; CA2 evaluates its record with the `persistUntil` restriction. Each CA verifies `accounturi` and all other validation requirements as specified in {{verification-procedure}}.

## Just-in-Time Validation {#just-in-time-validation}

When processing a new authorization request, a CA MAY perform an immediate DNS lookup for `_validation-persist` TXT records at the Validation Domain Name corresponding to the authorization object's own domain identifier (`A`), without walking ancestor domains.

If one or more such records exist at `_validation-persist.<A>`, the CA MUST evaluate them according to the requirements specified in {{verification-procedure}} and {{multiple-issuer-support}}, including requiring `policy=wildcard` if the authorization object has `wildcard` or `subdomainAuthAllowed` ({{!RFC9444}}, Section 4.1) set to `true`. If at least one record meets all validation requirements for that authorization's scope, the CA MAY transition the authorization to the "valid" status, subject to item 7 of {{subdomain-implementation-requirements}}, without returning a "pending" challenge to the client. When a CA makes an authorization `valid` through Just-in-Time validation, it includes the `dns-persist-01` challenge object with `status` `valid` and a `validated` time in the authorization's `challenges` array ({{!RFC8555}}, Section 7.1.4). This mechanism is an optimization and does not alter the ACME state machine defined in {{!RFC8555}}. The `newOrder` response is an order object containing authorization URLs; the client retrieves the referenced authorization objects using POST-as-GET requests ({{!RFC8555}}, Sections 7.4 and 7.5). If the CA offers the optional `newAuthz` resource, its response contains the authorization object directly ({{!RFC8555}}, Section 7.4.1).

If no DNS TXT record at `_validation-persist.<A>` meets the validation requirements, or if the records are absent, the CA MUST proceed with the standard authorization flow with a "pending" authorization and an associated `dns-persist-01` challenge object. This includes the case in which the domain owner has provisioned the record only at a parent domain. The standard flow allows the client to supply `ancestorDomain` in its challenge response ({{subdomain-certificate-validation}}).

CAs implementing Just-in-Time (JIT) validation SHOULD rate-limit JIT DNS lookups per domain identifier, independent of the requesting account, to prevent amplification attacks where multiple accounts trigger excessive queries against a target domain. CAs implementing Just-in-Time validation SHOULD provide account activity notifications or logging, as this path eliminates the challenge-response interaction that might otherwise provide a detection window for account compromise.

## Pre-Provisioning Records {#pre-provisioning-records}

Domain owners MAY provision `_validation-persist` TXT records before requesting any certificates, provided they can compute the `accounturi` value. The earliest point at which this value can be computed is ACME account creation, because the account URL returned in the newAccount response ({{!RFC8555}}, Section 7.3) is an input to the hash. No order, authorization, or challenge object is required to compute or publish the record. A client, or a party publishing on its behalf, MUST use the following values when constructing a record without a challenge object:

- **accounturi**: The hashed URI, computed as specified in {{validation-record-format}}. A client knows its own ACME account URL (as returned in the `Location` header of the account creation response, {{!RFC8555}}, Section 7.3), the FQDN it is provisioning, and its current account key. It can therefore compute and publish the hashed URI itself from the CA's `accountHashPrefix` without a per-provisioning request to the CA. A client that wishes to reuse a single record across multiple domains MAY instead use the domain-omitted opt-out form (`domain_name` set to `*`, see {{validation-record-format}}). The client can publish the resulting value at each Validation Domain Name or publish it once and alias each Validation Domain Name to it with a CNAME record; a value computed over a specific `domain_name` cannot be shared this way, because it validates only for that `R` ({{normalization-algorithm}}).
- **issuer-domain-name**: A CA offering the dns-persist-01 challenge type advertises its Issuer Domain Names in the `issuerDomainNames` array of its directory `meta` object (see {{directory-issuer-domain-names}}). Clients pre-provisioning records SHOULD use one of these values; a value obtained elsewhere is not guaranteed to appear in a later challenge and can leave the record unusable.

The client obtains the `accountHashPrefix` from the CA's directory over HTTPS ({{!RFC8555}}, Sections 6.1 and 7.1.1), so that the record is placed under the intended CA's infrastructure. Because the hashed URI cryptographically binds the account key (see {{hashed-uri-security}}), a record computed with the client's genuine account key cannot be made to validate for a different account, provided the `accountHashPrefix` passes the checks in {{validation-record-format}}, item 3. This key binding lets a domain owner safely pre-provision without the challenge-response exchange.

Computing the hashed URI requires no private-key operation. A party that computes or publishes a record on the domain owner's behalf needs the following values: the account URL, the account key's public JWK Thumbprint, the `domain_name` input (`R` or `*`), the CA's `accountHashPrefix`, the hash-algorithm identifier, and the selected Issuer Domain Name, together with the `policy` and `persistUntil` parameters the domain owner chooses. The account holder can share all of these values with another party. A domain owner MAY therefore delegate computation and publication of a `_validation-persist` record to a party that does not operate the ACME client and holds no ACME credentials, such as a DNS operator. In this arrangement, the ACME client itself needs no DNS credentials.

Before delegating the computation or publication of a `_validation-persist` record, the account holder MUST verify the account URL by sending an authenticated POST request to the exact account URL used in the digest. Per {{!RFC8555}}, Section 6.3, the payload of the JWS MUST be the empty octet string `""`. The client MUST follow no redirects and MUST verify that the returned account object has a `status` of `valid` ({{!RFC8555}}, Section 7.1.2). A JWS payload of `"{}"` MUST be rejected as non-conforming for this check. The account holder MAY reuse a successful proof for an unchanged (account URL, account key) pair to satisfy later pre-publication checks.

This proof authenticates that the requesting key controls the exact account URL used in the digest. It detects URL or key misconfiguration before an unverified account URL is disclosed to a third party or an unusable record is published in DNS. Cross-account substitution resistance itself is enforced by the key-bound digest in {{hashed-uri-security}}. The proof does not authenticate the account to the domain name being validated; the domain owner establishes that by publishing the record in their DNS zone.

The domain owner and the delegated party MUST use an authenticated channel to transmit the values listed above outside the ACME channel. Sharing these values with such a party discloses the account URL, which the hashed form otherwise keeps out of public DNS, and enables that party to correlate the account's records across domains (see {{account-uri-privacy}}). The key binding described in {{hashed-uri-security}} is unaffected by which party performs the computation.

A domain owner that delegates computation or publication SHOULD also publish a CAA `accounturi` parameter ({{!RFC8657}}) specifying its account URL as an independent reinforcing authorization. A CAA `accounturi` discloses the account URL in public DNS and lets observers link domains that share it, so domain owners relying on the correlation mitigation in {{hashed-uri-security}} should weigh that disclosure or use separate accounts.

{{fig-delegated-preprovisioning}} illustrates the message sequence when a domain owner delegates `_validation-persist` record computation and publication to a DNS operator prior to submitting an order:

~~~
ACME Client                  DNS Operator               ACME Server
     |                             |                         |
     |--- 1. GET directory --------------------------------->|
     |<-- accountHashPrefix, issuerDomainNames (D) ----------|
     |                             |                         |
     |--- 2. POST newAccount ------------------------------->|
     |<-- 201 Location: acct_URL ----------------------------|
     |                             |                         |
     |--- 3. POST-as-GET "" -------------------------------->|
     |<-- 200 status: "valid" -------------------------------|
     |                             |                         |
     |--- 4. Share acct_URL, ----->|                         |
     |    key thumbprint, domain,  |                         |
     |    prefix, algorithm,       |                         |
     |    issuer, and any policy   |                         |
     |    or persistUntil          |                         |
     |    (authenticated channel)  |                         |
     |                             |-- [5. Publish TXT]      |
     |                             |   (_validation-persist) |
     |                             |                         |
     |--- 6. POST newOrder --------------------------------->|
     |<-- Order (URLs for valid or pending authorizations) --|
~~~
{: #fig-delegated-preprovisioning title="Delegated Pre-Provisioning and Order Sequence"}

Organizations pre-provisioning records SHOULD maintain an inventory of `_validation-persist` records and the ACME accounts they reference. Domain owners MAY include a `persistUntil` parameter to bound a record's effective lifetime (see {{persist-until-parameter-considerations}}). Domain owners SHOULD audit `_validation-persist` records after any DNS infrastructure security incident, as pre-provisioned records persist beyond the window of compromise.

CAs implementing `dns-persist-01` MUST maintain a stable `accountHashPrefix` and stable account URLs for the lifetime of an account once advertised, and MUST document their stability guarantees. This stability is required because the hashed URI is computed over the account URL and served under the prefix. A CA MUST NOT assign an account URL that has previously identified a different account, including a deactivated or revoked one. If a CA must change its `accountHashPrefix` or account-URL structure, it MUST provide a migration period during which hashed URIs under both the old and new forms are accepted for validation. Similarly, while a name remains advertised in the directory `issuerDomainNames` array ({{directory-issuer-domain-names}}), the CA MUST continue to include that name in the `issuerDomainNames` array of every `dns-persist-01` challenge object it issues. If the CA removes a name from the directory, it SHOULD provide a migration period during which every `dns-persist-01` challenge object's array still includes the name; without a migration period, records pre-provisioned using that name fail validation as soon as it leaves challenge objects. During any migration period, the retained name still counts toward the 10-entry maximum in {{challenge-object}} and, if the CA advertises `caaIdentities`, still appears there ({{directory-issuer-domain-names}}).

## Combining Persistent DNS Validation with Certificate-Key Attestation {#certificate-key-attestation}

Some deployments, such as internal network devices and enterprise workloads that lack DNS write credentials and inbound HTTP connectivity, combine domain authorization via `dns-persist-01` with cryptographic proof that the certificate private key resides on an approved hardware device or module (for example, using `device-attest-01` as described in {{?I-D.ietf-acme-device-attest}}, Sections 5, 7.4, and 8).

DNS records alone cannot enforce that conjunction across CAs. Within a single ACME authorization object, listed challenges are alternatives ({{!RFC8555}}, Section 7.1.4). Similarly, a CAA `validationmethods` list ({{!RFC8657}}, Section 4) is disjunctive and authorizes issuance if any one listed method is used, and CAs ignore unrecognized parameters in `_validation-persist` TXT records ({{validation-record-format}}, item 1).

Instead, a deployment that requires both checks relies on CA order policy ({{!RFC8555}}, Sections 7.1.3 and 7.4). Under {{!RFC8555}}, Section 7.1.3, the authorizations required for an order are determined by server policy and need not have a one-to-one relationship with the requested certificate identifiers (for example, when a hardware identifier is validated for the order but omitted from the certificate signing request (CSR) and issued certificate; see {{?I-D.ietf-acme-device-attest}}, Section 7.4). In such a deployment, the CA includes both the domain authorization and a non-waivable attestation authorization on each order in scope (including orders for wildcard or subordinate names authorized via `policy=wildcard`). At finalization ({{!RFC8555}}, Section 7.4), the CA checks that all order authorizations are `valid` and that the CSR public key matches the attested public key even when `dns` validation used Just-in-Time validation ({{just-in-time-validation}}) or authorization reuse. When a renewal order uses a new CSR public key, the CA requires fresh attestation for that key. Because `_validation-persist` binds authorization to a specific `issuer-domain-name` and `accounturi`, a domain owner needing this guarantee provisions `_validation-persist` (and any reinforcing CAA `accounturi` / `issue`) records only for CAs and accounts whose policy enforces both checks.

# Wildcard Certificate Validation {#wildcard-certificate-validation}

The `policy=wildcard` parameter extends a record's scope to wildcard certificates and to subdomains of the validated FQDN.

## Scope of `policy=wildcard` {#wildcard-scope}

When a DNS TXT record includes the `policy=wildcard` parameter value, it authorizes certificate issuance for:

1. **The validated FQDN itself** - The base domain for which the TXT record exists (e.g., `example.com`)
2. **Wildcard certificates** - Wildcard certificates for the validated FQDN or any of its subdomains (e.g., `*.example.com`, `*.dept.example.com`)
3. **Specific subdomains** - Any specific subdomain of the validated FQDN (e.g., `www.example.com`, `app.example.com`, `server.dept.example.com`)

For example, a TXT record at `_validation-persist.example.com` containing `policy=wildcard` can validate certificates for `example.com`, `*.example.com`, `www.example.com`, and any other subdomain of `example.com`.

If the `policy` parameter is absent, or if its value is anything other than `wildcard`, the validation applies only to the specific FQDN being validated. CAs MUST NOT consider such validation sufficient for wildcard certificates or subdomains.

# Subdomain Certificate Validation {#subdomain-certificate-validation}

When the `policy=wildcard` parameter is present (as described in {{wildcard-certificate-validation}}), CAs MAY validate an authorization for a subdomain or deeper wildcard identifier using a persistent TXT record provisioned at an ancestor domain. This section specifies the `ancestorDomain` challenge-response selector, its boundary and verification rules, and its interaction with {{!RFC9444}}.

## Selecting a Parent Validation Record with `ancestorDomain` {#selecting-ancestor-domain}

An ACME client that wishes to fulfill a `dns-persist-01` challenge for an authorization with `identifier.value` `A` using a persistent record at an ancestor domain `R` includes the `ancestorDomain` field in its challenge response payload ({{challenge-response-and-verification}}). The field lets the CA avoid blind multi-label DNS queries across the hierarchy. For example:

~~~json
{
  "ancestorDomain": "example.com"
}
~~~
{: #fig-ancestor-domain-response title="Challenge Response Selecting an Ancestor Record"}

For wildcard certificate requests, {{!RFC8555}}, Section 7.1.3 removes the `*.` prefix when constructing the authorization's `identifier.value` `A` and sets `wildcard` to `true` on the authorization object:

- For a base-level wildcard request `*.example.com` validated via `_validation-persist.example.com`, `A` is already `example.com`. The client MUST omit `ancestorDomain` and send `{}` ({{wildcard-certificate-validation}}).
- For a deeper wildcard request `*.dept.example.com` (or a non-wildcard subdomain request `server.dept.example.com`) validated via `_validation-persist.example.com`, `A` is `dept.example.com` (or `server.dept.example.com`), and the client sends `"ancestorDomain": "example.com"`.

{{tab-domain-scope-summary}} summarizes how the requested certificate identifier, the authorization object's `identifier.value` (`A`) and scope flags, the challenge response payload, and the required `policy` parameter relate when validating against a persistent record at `_validation-persist.example.com` (`R` = `example.com`):

| Identifier | Authorization A | Challenge Response | Required policy |
| :--- | :--- | :--- | :--- |
| `example.com` | `example.com` | `{}` | none or `wildcard` |
| `*.example.com` | `example.com` (`wildcard: true`) | `{}` | `wildcard` |
| `a.example.com` | `a.example.com` | `{"ancestorDomain": "example.com"}` | `wildcard` |
| `*.a.example.com` | `a.example.com` (`wildcard: true`) | `{"ancestorDomain": "example.com"}` | `wildcard` |
| `a.example.com` (RFC 9444) | `example.com` (`subdomainAuthAllowed: true`) | `{}` | `wildcard` |
{: #tab-domain-scope-summary title="Summary of Identifier, Authorization, and Record Domains"}

## Implementation Requirements {#subdomain-implementation-requirements}

CAs and clients MUST enforce the following requirements when `ancestorDomain` is used in a `dns-persist-01` challenge response (items 1 through 6), when `_validation-persist.<A>` would exceed the DNS length limits (item 2), or when the authorization itself covers names below a public suffix (item 7):

1. **Mandatory recognition:** Every CA implementing `dns-persist-01` MUST recognize the `ancestorDomain` field in a `dns-persist-01` challenge response and MUST NOT silently ignore it to query `_validation-persist.<A>`.
2. **Canonical syntax and DNS length limits:** The `ancestorDomain` value MUST be a lowercase A-label domain name ({{!RFC5890}}) with no trailing dot and no `*.` wildcard prefix. Both `ancestorDomain` and the complete constructed Validation Domain Name (`_validation-persist.<ancestorDomain>`) MUST conform to the 63-octet label limit and the 253-octet presentation / 255-octet wire-format total length limit of {{!RFC1035}}, Section 2.3.4. If `ancestorDomain` is not a JSON string or violates these syntax or length rules, the CA MUST reject the challenge with a `malformed` error ({{!RFC8555}}, Section 6.7). The same label and total-length limits apply to the Validation Domain Name for every `R`, including `R = A` when `ancestorDomain` is omitted. If `_validation-persist.<A>` would exceed them, the CA MUST reject the `{}` response with a `malformed` error without changing the challenge state. The CA MUST NOT perform a Just-in-Time lookup ({{just-in-time-validation}}) for that overlong name. The CA MUST NOT query that overlong name or any shortened form of it. The client can select an ancestor instead.
3. **Strict whole-label proper ancestor:** `ancestorDomain` (`R`) MUST be an Ancestor Domain of the authorization's `identifier.value` (`A`, normalized per {{normalization-algorithm}}) as defined in {{!RFC9444}}, Section 2: `R` MUST be a whole-label proper suffix of `A` containing strictly fewer labels than `A`. A domain is not an ancestor of itself; if `R` equals `A` or is not a whole-label suffix of `A`, the CA MUST reject the challenge with a `malformed` error.
4. **Public-suffix boundary:** The CA SHOULD NOT accept an `ancestorDomain` (`R`) that is a public suffix ({{!RFC9499}}, Section 9), or that is an Ancestor Domain of a public suffix `P` where `P` equals `A` or is an Ancestor Domain of `A`, as determined by the CA from a source such as the Public Suffix List. A qualifying record at or above such a suffix could be used to validate the bound account for names held by unrelated registrants. A CA can reasonably accept such a domain `R` where a single organization controls every name below `R`, such as a single-organization top-level domain (TLD) or a namespace used by a private public key infrastructure (PKI). If the CA declines an `ancestorDomain` under this item, or its policy does not permit subdomain validation at the requested `ancestorDomain`, the CA MUST reject the challenge with an `unauthorized` error ({{!RFC8555}}, Section 6.7).
5. **Record scope and digest binding:** When validating at `_validation-persist.<ancestorDomain>`, a record satisfies validation only if it also includes `policy=wildcard`. If the only records that would otherwise satisfy validation lack `policy=wildcard` and the attempt is not eligible for `persistExpired` ({{error-handling}}), the CA MUST fail the validation attempt with an `unauthorized` error; other failure reasons are reported as described in {{error-handling}}. Both the client (or delegated DNS publisher) and the CA MUST compute the expected `accounturi` digest ({{validation-record-format}}) using the normalized `ancestorDomain` (`R`) as `domain_name` (or the `*` opt-out), not the child identifier `A`.
6. **Preservation of authorization scope:** Validating a challenge with `ancestorDomain` satisfies only that authorization object for `A`; it MUST NOT alter the authorization object's `identifier` field or its `subdomainAuthAllowed` value ({{!RFC9444}}, Section 4.1).
7. **Suffix-level authorizations:** The guidance in item 4 also applies when the authorization's `identifier.value` (`A`) is itself a public suffix and the authorization has `wildcard` or `subdomainAuthAllowed` set to `true`, so that validation at `R = A` would cover names below the suffix. In that case, the CA SHOULD NOT validate the authorization via `dns-persist-01` except in the circumstances item 4 describes. If the CA declines such an authorization, a Just-in-Time lookup ({{just-in-time-validation}}) MUST NOT transition it to `valid`; the CA proceeds with a `pending` challenge instead. If the CA declines a challenge response for such an authorization, it MUST reject the response with an `unauthorized` error ({{!RFC8555}}, Section 6.7).

Rejections of a challenge response under items 2 through 4 and 7 are HTTP errors on the challenge-response request ({{!RFC8555}}, Section 7.5.1) that leave the challenge's existing state unchanged; a challenge that has not yet accepted a response remains `pending`. In contrast, a failure under item 5 is a validation failure that transitions the challenge to `invalid` ({{!RFC8555}}, Section 7.1.6).

Note: Because `ancestorDomain` is a request-only challenge-response field, the resulting challenge object does not echo `ancestorDomain`. A CA implementing an earlier draft of this specification ignores `ancestorDomain`, as {{!RFC8555}}, Section 7.5.1 requires for unrecognized response fields, and validates at `_validation-persist.<A>`. The result fails closed: validation succeeds only if a record at `_validation-persist.<A>` independently authorizes `A`. A client cannot tell this outcome from the challenge object.

See {{subdomain-validation-risks}} for important security implications of enabling subdomain validation.

## Interaction with RFC 9444 {#rfc9444-interaction}

{{!RFC9444}} defines an optional ACME extension in which a client requests an authorization object whose `identifier.value` is itself an ancestor domain and whose `subdomainAuthAllowed` field is set to `true` ({{!RFC9444}}, Sections 4.1 through 4.3). This extension allows a single validated ancestor authorization to be reused across subsequent orders for subordinate subdomains without additional challenge flows.

When a CA implements {{!RFC9444}} alongside `dns-persist-01`, the two mechanisms operate at different protocol layers and interact as follows:

- If an authorization object has `subdomainAuthAllowed` set to `true` for identifier `A`, a CA validating that authorization via `dns-persist-01` MUST require `policy=wildcard` on the `_validation-persist` record even when the client omits `ancestorDomain` from the challenge response (`R = A`). This requirement also applies to Just-in-Time validation ({{just-in-time-validation}}). Item 7 of {{subdomain-implementation-requirements}} applies when `A` is a public suffix. When `A` is not a public suffix, whether an authorization with `subdomainAuthAllowed` set to `true` covers subordinate names at or below a public suffix beneath `A` is governed by CA policy ({{!RFC9444}}, Section 7.3).
- Under {{!RFC9444}}, Section 4.3, `ancestorDomain` in a `newOrder` identifier object specifies an upper control boundary; this field is distinct from the challenge-response `ancestorDomain` field defined in {{selecting-ancestor-domain}}. The boundary allows the server to create the authorization object for any intermediate ancestor `A_mid` up to and including that boundary. If the domain owner pre-provisioned the `_validation-persist` record only at a higher ancestor `R`, a lookup at `_validation-persist.<A_mid>` will find no record. The client MAY then supply `{"ancestorDomain": "<R>"}` in the `dns-persist-01` challenge response for the `A_mid` authorization, since `R` is a proper ancestor of `A_mid`. The resulting authorization object's scope remains `A_mid` and its subordinate names.

## Example: Subdomain Validation {#subdomain-validation-example}

For a persistent TXT record provisioned at `_validation-persist.example.com` with `policy=wildcard`:

- Permitted: `example.com` and `*.example.com` (with challenge response `{}`), as well as `www.example.com`, `app.example.com`, `server.dept.example.com`, and `*.dept.example.com` (with challenge response `{"ancestorDomain": "example.com"}`, or under an {{!RFC9444}} ancestor authorization for `example.com` with `subdomainAuthAllowed` set to `true`)
- Not permitted without additional validation: `example.org`, `example.net`

# IP Address Validation via the Reverse Namespace {#ip-address-validation}

The `dns-persist-01` validation method MAY also be used to validate control of an IPv4 or IPv6 address represented as an ACME `ip` identifier ({{!RFC8738}}, Section 3) by provisioning a persistent DNS TXT record in the reverse DNS namespace.

## Reverse-Zone Mapping and Validation Domain Name {#ip-reverse-zone-mapping}

When an ACME authorization object has `identifier.type` set to `ip`, the client and CA construct the normalized Reverse Zone Domain Name (`R`) for the IP address in lowercase US-ASCII with no trailing dot. {{!RFC1035}}, Section 3.5 and {{!RFC3596}}, Section 2.5 write `IN-ADDR.ARPA` and `IP6.ARPA` in uppercase; per {{normalization-algorithm}}, this specification uses lowercase `.in-addr.arpa` and `.ip6.arpa`. The construction proceeds as follows:

- **IPv4 ({{!RFC1035}}, Section 3.5):** Represent the 32-bit IPv4 address as four decimal octet labels (0 to 255, with leading zeros omitted except for a single `0` octet) in reverse order, followed by the suffix `.in-addr.arpa`. For example, `192.0.2.1` maps to `1.2.0.192.in-addr.arpa`.
- **IPv6 ({{!RFC3596}}, Section 2.5):** Expand the 128-bit IPv6 address to all 32 hexadecimal digits (expanding any `::` zero compression). Represent them as 32 single-character lowercase hexadecimal (`0` through `9` and `a` through `f`) labels separated by dots in reverse order (low-order nibble first), followed by the suffix `.ip6.arpa`. The resulting Reverse Zone Domain Name is always 72 octets in length. For example, `2001:db8::1` maps to the Reverse Zone Domain Name used in {{ex-ipv6-validation}}.

For an `ip` identifier, the Validation Domain Name is formed by prepending the label `_ip-validation-persist` to the Reverse Zone Domain Name (`_ip-validation-persist.<R>`). This distinct underscore label separates IP-address validation from `dns`-identifier validation.

A CA MUST query `_ip-validation-persist.<R>` only when validating an authorization with `identifier.type` set to `ip`, and MUST query `_validation-persist` only when validating an authorization with `identifier.type` set to `dns`. When validating an `ip` identifier with `dns-persist-01`, the CA MUST NOT perform a `PTR` lookup or validate the target domain name of a `PTR` record in place of `_ip-validation-persist.<R>`.

## Record Format, Digest Input, and Scope for IP Validation {#ip-record-and-scope}

The following requirements also apply to `ip` authorizations:

- Record format: {{validation-record-format}}.
- Verification and key retention: {{verification-procedure}}.
- Multiple issuers: {{multiple-issuer-support}}.
- Just-in-Time validation: {{just-in-time-validation}}.
- Pre-provisioning: {{pre-provisioning-records}}, excluding the `dns`-only CAA `accounturi` guidance because CAA does not govern IP address issuance.
- Certificate-key attestation: {{certificate-key-attestation}}.
- Validation-data reuse: {{validation-data-reuse}}.
- Revocation and invalidation: {{revocation-and-invalidation}}.
- Implementation requirements: {{implementation-considerations}}.

They apply by substituting `_ip-validation-persist.<R>` for `_validation-persist.<FQDN>` and the normalized Reverse Zone Domain Name `R` for the validated FQDN, subject to the following rules:

1. **Digest `domain_name` input:** By default, both the client (or delegated DNS publisher) and the CA MUST set `domain_name` in `H(length_of_domain || domain_name || key || account_URL)` ({{validation-record-format}}) to the normalized lowercase US-ASCII Reverse Zone Domain Name `R`. This is the Validation Domain Name with its leading `_ip-validation-persist.` removed and no trailing dot. As specified in {{validation-record-format}}, a client MAY instead set `domain_name` to the single US-ASCII octet `*` (`0x2a`; `length_of_domain` `0x01`) to opt out of per-address domain binding, and a CA MUST accept both forms.

   A CNAME record at `<R>` (such as a classless `in-addr.arpa` `PTR` alias) redirects only the owner name `<R>` and not child labels beneath it. Delegating `_ip-validation-persist.<R>` via CNAME therefore requires an explicit alias at `_ip-validation-persist.<R>` itself.

   Where `_ip-validation-persist.<R>` is aliased via CNAME to a target in a delegated zone, both the publisher and the CA MUST still compute `domain_name` using the normalized `in-addr.arpa` or `ip6.arpa` Reverse Zone Domain Name `R` (or `*`), never the CNAME target. That is, `R` is the reverse-mapping domain name of the IP address prior to following any CNAME aliases, not the DNS canonical name of the alias target in the sense of {{!RFC1035}}, Section 3.3.1.
2. **Exact-address scope:** Validation of `_ip-validation-persist.<R>` authorizes certificate issuance only for the single IPv4 or IPv6 address in the `ip` authorization's `identifier.value`. It grants no authority over any IP prefix, Classless Inter-Domain Routing (CIDR) range, or reverse-zone ancestor. Clients and DNS publishers SHOULD omit the `policy` parameter on `_ip-validation-persist` records. If `policy=wildcard` (or any other `policy` value) is present on an `_ip-validation-persist` record, the CA MUST ignore it and treat the validation as applying only to the single IP address.
3. **Challenge response for `ip` authorizations:** To respond to a `dns-persist-01` challenge for an `ip` authorization, the client POSTs an empty JSON object (`{}`) to the challenge URL ({{!RFC8555}}, Section 7.5.1). Ancestor selection does not apply to `ip` identifiers; if a client includes an `ancestorDomain` field in a challenge response for an `ip` authorization, the CA MUST reject the challenge with a `malformed` error ({{!RFC8555}}, Section 6.7). This is an HTTP error on the challenge-response request that leaves the challenge's existing state unchanged.
4. **Identifier-typed caching:** Any CA caching of computed hashed URIs or validation outcomes ({{verification-procedure}}) MUST remain scoped to the authorization's identifier type (`ip` versus `dns`) so that a digest or record evaluated for one identifier type is never treated as satisfying an authorization for the other identifier type.

## IPv4 and IPv6 Validation Examples {#ip-validation-examples}

Using the same CA `accountHashPrefix` (`https://ca.example/account-hash/`), Issuer Domain Name (`authority.example`), account URL (`https://ca.example/acct/123`), and account key thumbprint (`NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs`) as {{hashed-uri-example}}:

- **IPv4 example (`192.0.2.1`):**
  - `length_of_domain`: `0x16` (22 octets)
  - `domain_name`: `1.2.0.192.in-addr.arpa`
  - Unpadded base64url SHA-256 digest: `CnwSwHOHJbE-X_FSkydPTw6dGMovluSdRsY3_tmA-0E`

~~~ dns
_ip-validation-persist.1.2.0.192.in-addr.arpa. IN TXT (
  "authority.example;"
  " accounturi=https://ca.example/account-hash/"
  "sha-256/CnwSwHOHJbE-X_FSkydPTw6dGMovluSdRsY3_tmA-0E")
~~~
{: #ex-ipv4-validation title="IPv4 Validation TXT Record"}

- **IPv6 example (`2001:db8::1`):**
  - `length_of_domain`: `0x48` (72 octets)
  - `domain_name`: the Reverse Zone Domain Name for `2001:db8::1`, formed by appending the `$ORIGIN` below to the relative owner name and removing the leading `_ip-validation-persist.` label and the trailing dot.
  - Unpadded base64url SHA-256 digest: `9RLd7iwi0v30KV01dsPwGvSBucGEYgsJhDNCxk8Nb9g`

~~~ dns
$ORIGIN 0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa.
_ip-validation-persist.1.0.0.0.0.0.0.0 IN TXT (
  "authority.example;"
  " accounturi=https://ca.example/account-hash/"
  "sha-256/9RLd7iwi0v30KV01dsPwGvSBucGEYgsJhDNCxk8Nb9g")
~~~
{: #ex-ipv6-validation title="IPv6 Validation TXT Record"}

If the client instead opts out of per-address binding by setting `domain_name` to `*` (`length_of_domain` `0x01`), both records use the same hashed URI as the domain-omitted example in {{hashed-uri-example}} (`https://ca.example/account-hash/sha-256/NpDnSwUthQK8zCgFdefYxAdAVPnygMLbs9US6oO-5ug`).

# Implementation Status {#implementation-status}
{:removeinrfc}

This section records the status of known implementations of the protocol defined by this specification at the time of posting of this Internet-Draft, and is based on a proposal described in {{?RFC7942}}. The description of implementations in this section is intended to assist the IETF in its decision processes in progressing drafts to RFCs. Please note that the listing of any individual implementation here does not imply endorsement by the IETF. Furthermore, no effort has been spent to verify the information presented here that was supplied by IETF contributors. This is not intended as, and must not be construed to be, a catalog of available implementations or their features. Readers are advised to note that other implementations may exist.

According to {{?RFC7942}}, "this will allow reviewers and working groups to assign due consideration to documents that have the benefit of running code, which may serve as evidence of valuable experimentation and feedback that have made the implemented protocols more mature. It is up to the individual working groups to use this information as they see fit".

The entries below describe draft support and planned work. Conformance to the -03 revision has not been verified.

Boulder (CA server):
: Organization: Internet Security Research Group (ISRG). Boulder is the ACME server used by Let's Encrypt. Boulder includes support for draft-ietf-acme-dns-persist-01, added in [PR #8725](https://github.com/letsencrypt/boulder/pull/8725). An update to -02 is tracked in [issue #9127](https://github.com/letsencrypt/boulder/issues/9127). Source: https://github.com/letsencrypt/boulder

Pebble (test CA server):
: Organization: ISRG. Pebble is a small ACME test server. It has supported `dns-persist-01` since release [v2.10.0](https://github.com/letsencrypt/pebble/releases/tag/v2.10.0), added in [PR #536](https://github.com/letsencrypt/pebble/pull/536). Source: https://github.com/letsencrypt/pebble

lego (client):
: lego is an ACME client and library written in Go. It has supported `dns-persist-01` since release [v5.0.0](https://github.com/go-acme/lego/releases/tag/v5.0.0), added in [PR #2871](https://github.com/go-acme/lego/pull/2871). Source: https://github.com/go-acme/lego

cert-manager (client):
: cert-manager is a certificate controller for Kubernetes. A design proposal for `dns-persist-01` is under review as draft [PR #9487](https://github.com/cert-manager/cert-manager/pull/9487). Source: https://github.com/cert-manager/cert-manager

Implementations of earlier revisions of this document use the same challenge type name but do not interoperate with this revision. For example, a client implementing this revision does not use a CA whose directory omits `accountHashPrefix`, as directories implementing revision -01 do.

RFC Editor: please remove this section, together with the reference to {{?RFC7942}}, before publication.

# Security Considerations {#security-considerations}

The requirement for CAs to ignore unknown parameter tags means that a future extension must be designed so that a CA that ignores it does not create a security vulnerability. Extensions that require strict enforcement should use alternative mechanisms, such as separate record types or explicit version negotiation.

## Persistent Record Risks {#persistent-record-risks}

The persistence of validation records creates extended windows of vulnerability compared to traditional ACME challenge methods. If an attacker gains control of a DNS zone containing persistent validation records, they can potentially obtain certificates for the validated domains until the validation records are removed or modified.

Clients SHOULD protect validation records with DNS security measures, including:

- Using DNS providers with strong authentication and access controls
- Implementing DNS Security Extensions (DNSSEC) where possible
- Monitoring DNS zones for unauthorized changes
- Regularly reviewing and rotating validation records

## Account Binding Security {#account-binding-security}

The `accounturi` parameter binds domain validation to the ACME account identified by the hashed URI. This binding is only as secure as the account itself.

The security of this method depends on protecting the ACME account's current private key, which authenticates requests to the CA. An attacker who compromises that key can authenticate as the account and use qualifying persistent records without further DNS access, including dormant records computed with retained prior key thumbprints (see {{verification-procedure}}). The attacker does not need the corresponding old private keys. Still-valid authorizations can also support issuance within their existing scope and expiration. Issuance remains subject to the CA's authorization and policy checks; account-key compromise does not guarantee immediate issuance. The persistence of the records makes secure account-key management important throughout their effective lifetime.

CAs SHOULD protect accounts with measures including:

- Strong authentication requirements for ACME accounts
- Account activity monitoring and anomaly detection
- Rapid account revocation capabilities
- Regular account security reviews
- Account key rotation policies and procedures

Clients SHOULD protect their ACME account keys with the same level of security as they would protect private keys for high-value certificates.

### Account Key Rotation {#account-key-rotation}

When an account key rotates following the procedures defined in {{!RFC8555}}, Section 7.3.5, the ACME account URL remains unchanged. Recomputing the hashed URI with the new account key produces a different value because `key` is an input to the hash. Published records remain unchanged until republication. Existing records remain valid under the retention rules in {{verification-procedure}}. Clients SHOULD republish `_validation-persist` records with the hashed URI recomputed under the account's new key promptly after a rotation. Republication keeps records in step with current credentials, which simplifies auditing.

### Account URI Privacy {#account-uri-privacy}

Because `_validation-persist` TXT records are publicly queryable and long-lived, the `accounturi` value is visible to any party that queries DNS. With a cleartext account identifier, the same value appearing in records for multiple domains would let third parties infer that those domains share the same ACME account and likely share infrastructure; this correlation risk is noted in {{!RFC8657}}, Section 5.9. The hashed URI defined in this document mitigates that risk by binding `domain_name` into the digest, so a given account produces a different `accounturi` value for each domain (see {{hashed-uri-security}}).

Domain owners who require stronger privacy can additionally use separate ACME accounts for domains that should not be correlated. The domain-omitted opt-out form (`domain_name` set to `*`, see {{validation-record-format}}) deliberately forgoes the per-domain mitigation: the same value then appears in every record that uses it, so a bulk observer can link those domains. A domain owner who uses the opt-out to reuse one record across domains accepts that linkage.

### Hashed Account URIs {#hashed-uri-security}

The hashed URI is the sole `accounturi` form ({{validation-record-format}}). It serves two purposes: it cryptographically binds the requesting account to the domain so that a record cannot be transferred to authorize a different account, preserving the non-transferability property of {{!RFC8555}}; and it reduces the correlation exposure described in {{account-uri-privacy}}. It does not hide the account from the CA, which necessarily learns the account URL when it recomputes and validates the record.

**Source of the authorization binding.** The account holder commits to the account URL by publishing a digest computed over `account_URL`. Because the hash is collision resistant, this commitment is comparable to publishing the account URL directly, while the digest also incorporates the account key and the domain, as described below.

**Non-transferability and substitution resistance.** A raw account URL is only a URL commitment: its integrity is only as good as the channel that delivered the URL to the domain owner. If that channel is poisoned, the record binds to an account the owner did not intend, so RFC 8555 non-transferability is not preserved. The hashed URI closes this gap by additionally binding the account key. A CA recomputes the digest over the requesting account's own key, so a record validates for an account only if it was computed with a key that account controls. An attacker therefore cannot make an honest CA accept, for the attacker's account, a digest the victim published, because the attacker does not hold the victim's private key. An attacker who substitutes a different account URL still cannot produce a validating record without the corresponding key. This substitution resistance is the reason this document defines only the hashed URI and does not permit a raw account URL. It depends on the checks of `accountHashPrefix` ({{validation-record-format}}, item 3); a prefix carrying a `;` could otherwise cause the client's own digest to be parsed as an ignored parameter.

**Role of the account key (privacy).** ACME account URLs are frequently low in entropy (e.g., a short, sequential account identifier of only a handful of digits). An adversary who knows a CA's account-URL structure could enumerate all plausible account URLs, hash each against a target domain, and recover the account URL behind a published digest, defeating the privacy goal. The account key's JWK Thumbprint is included as a high-entropy input that raises the cost of this enumeration.

**HTTP-01 key exposure.** The key authorization for the `http-01` challenge ({{!RFC8555}}, Section 8.3) is transmitted over cleartext HTTP and contains the same unpadded base64url SHA-256 JWK Thumbprint used as the `key` input above. An on-path observer of an `http-01` validation for the account (or a party that obtains the thumbprint from a stateless `http-01` responder or published client logs) therefore learns the thumbprint. Only the account URL then remains unknown when evaluating a published hashed URI. The observer must still enumerate candidate account URLs and recompute the digest, so the remaining protection for that observer is limited to the entropy of the account URL. A hashed URI under the CA's `accountHashPrefix` is a comparison identifier, even when it has an HTTPS URI form. It is not dereferenced during validation, and CAs MUST NOT serve account-identifying information at the constructed hashed URI.

**Binding to the record's own domain.** The digest binds `domain_name`, which is `R`, the FQDN whose Validation Domain Name the CA queries, not the FQDN of the certificate ultimately requested. This has two effects. First, the same account produces a different `accounturi` value in every domain's record, so a bulk observer cannot correlate domains by matching identical strings, as it could with a shared cleartext account URL. Second, the digest binds the pre-alias `R` ({{normalization-algorithm}}). The CA can therefore use a record computed over a specific `domain_name` only for validation attempts using that `R`, regardless of the CNAME target, and the record cannot be lifted to a different domain. A domain owner who deliberately wants one record to authorize many names uses the domain-omitted opt-out (`domain_name` set to `*`), accepting the correlation trade-off described in {{account-uri-privacy}}.

**Digest construction.** The digest is compared for exact equality against a value the CA recomputes over a fixed field structure ({{validation-record-format}}), so there is no attacker-controlled trailing input, and length-extension of the digest yields nothing an attacker can use. The `length_of_domain` prefix and the fixed 43-octet `key` make every field boundary unambiguous regardless of which octets appear in `domain_name` or `account_URL`, so no field can be shifted into another to construct a colliding preimage. The `key` uses the unpadded base64url thumbprint representation already present as the second component of an ACME key authorization ({{!RFC8555}}, Section 8.1), avoiding an unnecessary decode step.

**Uniqueness of the binding.** What {{!RFC8657}}, Section 5.4 requires of an `accounturi` is reverse-direction uniqueness: a single `accounturi` value must not identify two different accounts. This holds because `account_URL` is one of the hash inputs and the hash is collision resistant, so each hashed URI uniquely identifies one account. The forward direction is intentionally one-to-many: one account yields a different value per domain, which is the privacy property above.

**Acceptance of prior keys.** A CA accepts digests generated with an account's prior keys so that records provisioned before a rotation survive without republication ({{!RFC8555}}, Section 7.3.5), subject to the deactivation and retention rules in {{verification-procedure}}. This acceptance aligns with {{!RFC8555}}, Section 7.3.5, which establishes that changing an account key should not have collateral impact on existing authorizations. Such a record was committed to the account's URL when it was provisioned, so even if the key in the record is one that was later rotated out, the domain owner's intent to authorize that account is preserved. Retaining historical public key thumbprints does not allow a rotated-out key alone to authenticate subsequent requests under {{!RFC8555}}, Section 7.3.5. Registering a fresh account with that key yields a different account URL and therefore a different hashed URI. However, a requester authenticating with the original account's current key can use a qualifying record computed with any retained prior thumbprint, including a dormant record that was never previously validated, without possessing the corresponding old private key. This continuity also applies after compromise of the current account key. A key rollover alone does not establish whether control of the account remains with the intended holder; lifetime retention preserves record usability rather than providing takeover containment. Account deactivation overrides prior-key acceptance, while DNS record removal and `persistUntil` have the timing and reuse effects described in {{revocation-and-invalidation}} and {{validation-data-reuse}}.

**Limits of the privacy protection.** This construction is a best-effort obfuscation, not a cryptographically strong privacy mechanism. It resists bulk correlation and casual enumeration of account URLs, but a determined adversary with a priori knowledge of candidate account URLs and their account keys can recover the true account URL. Domain owners who require stronger unlinkability SHOULD use separate ACME accounts for domains that must not be correlated.

## Subdomain Validation Risks {#subdomain-validation-risks}

The `policy=wildcard` value serves as the explicit mechanism for domain owners to opt in to broader validation scopes. Organizations using this feature SHOULD carefully control subdomain delegation and monitor for unauthorized subdomains.

Risks include:

- Subdomain takeover attacks where abandoned subdomains are claimed by attackers
- Unauthorized certificate issuance for subdomains controlled by different organizations
- Confusion about which entity has authority over specific subdomains

Organizations considering the use of subdomain validation SHOULD:

- Maintain strict control over subdomain delegation
- Implement monitoring for subdomain creation and changes
- Consider limiting subdomain validation to specific, controlled scenarios
- Provide clear governance policies for subdomain certificate authorization

Operators of shared parent domains whose provisioning interfaces let tenants create arbitrary child labels SHOULD reserve the `_validation-persist` and `_ip-validation-persist` labels so that no tenant can create them. A tenant-created `policy=wildcard` record at a shared parent would otherwise authorize that tenant's account for the parent and its other tenants.

## IP Address Validation Risks {#ip-validation-risks}

When `dns-persist-01` is used for an `ip` identifier ({{ip-address-validation}}), authorization derives from control of the `_ip-validation-persist.<R>` node in the `in-addr.arpa` or `ip6.arpa` reverse DNS tree. In many deployments, reverse-zone delegation is held by the upstream network provider, hosting operator, or address-space registrant rather than the operator of the individual host. {{!RFC8738}}, Section 7 prohibits `dns-01` for IP identifiers (see also Section 9.1 of that RFC). Unlike `dns-01`, `_ip-validation-persist` uses a dedicated underscore label (`_ip-validation-persist`), binds both the CA (`issuer-domain-name`) and the ACME account (`accounturi`), forbids `PTR` target indirection, and ignores `policy=wildcard` so a single authorization covers only one IP address.

Because the reverse-zone operator controls all nodes within its delegated zone, it can still synthesize TXT responses for any address in that zone. For example, it can publish a DNS wildcard TXT record such as `*.2.0.192.in-addr.arpa` carrying a `domain_name=*` hashed URI, which can answer both `_ip-validation-persist.<R>` and `_validation-persist.<R>` queries where no closer node intervenes.

If an IP address is reassigned while a persistent `_ip-validation-persist.<R>` record (or CNAME delegation) remains in place, the prior holder's account could continue to validate the address until the record is removed and any existing authorizations expire. Operators SHOULD remove `_ip-validation-persist.<R>` records upon address reassignment. Operators SHOULD also use `persistUntil` ({{persist-until-parameter-considerations}}) to bound persistent records to the expected address allocation period. Address holders and CAs SHOULD account for reverse-zone delegation boundaries in their threat models.

## Cross-CA Validation Reuse {#cross-ca-validation-reuse}

The persistent nature of validation records raises concerns about potential reuse across different CAs. While the `issuer-domain-name` parameter is designed to prevent such reuse, CAs MUST validate that the `issuer-domain-name` in the DNS record matches, using the comparison specified in {{validation-record-format}}, one of the CA's disclosed Issuer Domain Names.

## Record Tampering and Integrity {#record-tampering-and-integrity}

DNS records are generally not authenticated end-to-end, making them potentially vulnerable to tampering. CAs SHOULD implement additional integrity checks where possible and consider the overall security posture of the DNS infrastructure when relying on persistent validation records.

Additionally, CAs SHOULD protect their `ACME directory URL` with appropriate security measures. Using DNSSEC to protect the CA's `ACME directory URL` is RECOMMENDED. An attacker who compromises the DNS for a CA's `ACME directory URL` could disrupt validation or, in some cases, impersonate the CA. While this is a systemic DNS security risk that extends beyond this specification, it is amplified by any mechanism that relies on DNS for identity.

## Issuer Domain Name Normalization and Limits {#issuer-domain-name-limits}

The `issuerDomainNames` field requires lowercase ASCII domain names, with internationalized labels represented as A-labels and no trailing dot. Clients use these advertised names unchanged when constructing records. CAs compare the parsed issuer name using Simple String Comparison, as specified in {{validation-record-format}}. An issuer name that differs in case from an advertised name does not match. The issue-value syntax prohibits Unicode input and trailing dots in the issuer name. The order of names in the array has no significance.

The 10-entry limit on `issuerDomainNames` bounds the size of the advertised list and the work needed to process it.

## CAA Interaction {#caa-interaction}

The `dns-persist-01` validation method is a valid `validationmethods` parameter value for CAA records per {{!RFC8657}}. Although `_validation-persist` reuses the `accounturi` parameter name from {{!RFC8657}}, Section 3, the two parameters follow different rules. Under {{!RFC8657}}, Section 3.1, a CAA `accounturi` for an ACME account is the URI of the ACME account object (the account URL); when the parameter is present, the CA matches that URI as part of CAA processing. The `dns-persist-01` `accounturi` is instead the hashed URI defined in {{validation-record-format}}, which the CA matches by recomputing it while validating the `_validation-persist` record. These parameters occur in different DNS records and need not have the same value. They are independent reinforcing checks: both can restrict issuance, and the CA evaluates them separately. A comma-separated `validationmethods` list in CAA authorizes issuance when any one listed method is used ({{!RFC8657}}, Section 4). Combining `dns-persist-01` with a second mandatory check such as certificate-key attestation is therefore enforced through CA order policy rather than a CAA `validationmethods` list (see {{certificate-key-attestation}}).

## DNS Security Measures {#dns-security-measures}

### DNSSEC {#dnssec}

DNS Security Extensions (DNSSEC) {{!RFC9364}} provide cryptographic authentication of DNS data, ensuring that the validation records retrieved by a CA are authentic and have not been tampered with. CAs SHOULD use a DNSSEC-validating resolver when querying `dns-persist-01` TXT records. Without DNSSEC validation, the CA cannot detect invalid DNSSEC signatures and may accept forged responses even when the zone is signed. If a CA performs DNSSEC validation, it MUST treat validation failure (e.g., expired signatures, broken chain of trust) as a challenge failure and MUST NOT use the record for domain validation. This requirement is stricter than the general DNSSEC guidance in {{!RFC8555}} because `dns-persist-01` records are long-lived and a reusable record makes authentic DNS responses important throughout its lifetime.

### Multi-Perspective Validation {#multi-perspective-validation}

Multi-Perspective Issuance Corroboration (MPIC) is a technique to validate domain control from multiple network vantage points. This is a critical defense against localized network attacks, such as Border Gateway Protocol (BGP) hijacking and DNS spoofing, which could otherwise lead to certificate mis-issuance.

For CAs subject to requirements like the CA/Browser Forum Baseline Requirements, MPIC is essential to domain validation. For private PKI systems where the network topology is well-known and such localized attacks are not part of the threat model, operators might reasonably judge MPIC unnecessary.

## Validation Data Reuse {#validation-data-reuse}

This validation method is designed for persistence and reuse. The period for which a CA may rely on validation data is its `Validation Data Reuse Period` (as defined in {{conventions-and-definitions}}), bounded by the CA's own policy, applicable root program requirements, and any `persistUntil` constraint on the record ({{validation-record-format}}). The DNS TTL of the `_validation-persist` record governs caching at the DNS layer only; it is not a validation data reuse limit, and a CA MUST NOT derive the effective validation data reuse period from the record's observed TTL.

CAs MAY reuse validation data obtained through this method for the duration of their Validation Data Reuse Period, subject to the `expires` constraint established by any `persistUntil` parameter on the validated record (see {{validation-record-format}}). For a validation attempt that queries DNS, removing or changing the TXT record takes effect after resolver caches expire, as described in {{revocation-and-invalidation}}. Record removal or later modification of `persistUntil` in DNS does not retroactively invalidate or shorten previously obtained validation data before its established authorization expiration.

## persistUntil Parameter Considerations {#persist-until-parameter-considerations}

The `persistUntil` parameter provides domain owners with direct control over the validity period of their validation records. CAs and clients should be aware of the following considerations:

- Domain owners should set expiration dates for validation records that balance security and operational needs. To avoid unexpected validation failures during certificate renewal, domain owners are advised to:
   - Align `persistUntil` values with certificate lifetimes or planned maintenance intervals
   - Monitor or set reminders for `persistUntil` expirations
   - Document `persistUntil` practices in certificate management procedures
   - Automate updates to validation records with new `persistUntil` values during certificate renewal workflows
- CAs MUST parse and apply the `persistUntil` timestamp as specified in {{validation-record-format}}.

## Revocation and Invalidation of Persistent Authorizations {#revocation-and-invalidation}

The persistent nature of `dns-persist-01` authorizations means that a valid DNS TXT record can grant control for an extended period, potentially even if the domain owner's intent changes or if the associated account key is compromised.

There are two primary revocation and invalidation actions:

1. **Remove DNS record:** Removing the corresponding DNS TXT record from the Validation Domain Name prevents new `dns-persist-01` validations after resolver caches expire; it does not invalidate an existing authorization object. This action is available to the domain owner and does not require CA involvement.
2. **Deactivate ACME account:** If account-key compromise is suspected, the account bound by the record can be deactivated as specified in {{!RFC8555}}, Section 7.3.6. Per the deactivation rule in {{verification-procedure}}, a CA MUST NOT accept a hashed URI computed from any key of a deactivated account; new validations fail without waiting for DNS caches to expire, and deactivation overrides witnessed key retention.

   If an attacker who compromised the account key has already performed a key rollover ({{!RFC8555}}, Sections 7.3.5 and 11.1), the legitimate holder's prior key can no longer authenticate an in-band deactivation request. In that case, CAs SHOULD provide an out-of-band account deactivation mechanism ({{account-binding-security}}), subject to identity verification commensurate with the account's issuance capability. DNS record removal (item 1) remains directly available to the domain owner.

   RFC 8555 prohibits further requests authorized by the deactivated account's key, but cancellation of already-pending operations, such as certificate orders, is a SHOULD rather than an unconditional requirement. Deactivation therefore does not guarantee that every pending issuance operation is canceled. RFC 8555 also recommends against revoking already-issued certificates solely because the account was deactivated.

{{tab-revocation-actions}} summarizes the applicability and timing of these actions:

| Action | Effect |
|---|---|
| Remove DNS record | New validations fail after resolver cache expiry; domain-owner action, no CA involvement |
| Deactivate ACME account | New validations fail immediately; overrides key retention (see {{verification-procedure}}) |
{: #tab-revocation-actions title="Effects of DNS Record Removal and Account Deactivation"}

ACME clients SHOULD provide clear mechanisms for users to:

- Remove the `_validation-persist` DNS TXT record.
- Monitor the presence and content of their `_validation-persist` records to confirm that they reflect the intended authorizations.

CAs implementing this method MUST:

- During a validation attempt, fail the validation if the corresponding DNS TXT record is no longer present or if its content does not meet the requirements of this specification (e.g., incorrect `issuer-domain-name`, missing `accounturi`, altered `policy`).

- Respect the `persistUntil` constraint as specified in {{validation-record-format}}, setting authorization expiration to no later than the specified timestamp and rejecting new validation attempts after the timestamp elapses.

- Handle failed challenges as specified in {{verification-procedure}} and {{ca-implementation-guidelines}}; apply the fallback in {{just-in-time-validation}} when a Just-in-Time lookup finds no qualifying record.

While this method provides a persistent signal of control, the fundamental ACME authorization object (as defined in {{!RFC8555}}) remains subject to its own lifecycle, including expiration. A persistent DNS record can support repeated validation attempts. Each valid authorization object has its own `expires` timestamp; retaining the DNS record does not automatically extend that timestamp.

# Privacy Considerations {#privacy-considerations}

This section addresses the privacy considerations described in {{?RFC6973}}. A `_validation-persist` record is public and long-lived. Anyone who can query DNS can observe it for as long as it is published.

Correlation of domains through account identifiers:
: A cleartext account URL in many records would let an observer link the domains that share an ACME account. By default, the hashed URI binds each value to its record domain, so a single account publishes a different value at each domain name, even when those names are in the same DNS zone ({{account-uri-privacy}}). Distinct values prevent linking by string comparison only. Records can still be linked through other shared features, such as identical `persistUntil` values or simultaneous republication after a key rotation ({{account-key-rotation}}). Removed records can remain visible in passive DNS archives. The mitigation is best-effort and has limits; see {{hashed-uri-security}}.

Choices that weaken the mitigation:
: The domain-omitted opt-out ({{validation-record-format}}) publishes the same value in every record that uses it. A CAA `accounturi` parameter ({{caa-interaction}}) publishes the account URL itself. Sharing the account URL with a DNS operator or other third party lets that party link the account's records ({{pre-provisioning-records}}). Domain owners who need domains to stay unlinked can use separate ACME accounts.

Information revealed by the record's presence:
: A record reveals an intended CA authorization for its domain. Its optional `persistUntil` value reveals the record's stated cutoff for new validation attempts. Authorization objects created from earlier validations have their own expiration times ({{validation-data-reuse}}). Records under `_ip-validation-persist` reveal which addresses an operator has prepared for certificates. These facts may expose deployment structure that the domain or address holder considers private.

Information held by the CA:
: The CA learns which domains and addresses have authorized each account when it validates them, as with other ACME challenges. Because the CA holds each account's URL and key thumbprints, it can also match any published record under its `accountHashPrefix` to an account, including a pre-provisioned record for which no order was placed. It can repeat that check at any time without a new action by the account holder. The CA retains account-key thumbprints for the account's lifetime under {{verification-procedure}}; exposure of those associations can weaken the correlation mitigation described in {{hashed-uri-security}}.

Information returned in errors:
: The `persistExpired` error is returned only for a record that would otherwise validate for the requesting account and CA, so it does not reveal the state of another account's records ({{error-handling}}).

# IANA Considerations {#iana-considerations}

## ACME Validation Methods Registry {#acme-validation-methods-registry}

IANA is requested to register the following entries in the "ACME Validation Methods" registry:

| Label | Identifier Type | ACME | Reference |
|---|---|---|---|
| dns-persist-01 | dns | Y | This document |
| dns-persist-01 | ip | Y | This document |
{: title="ACME Validation Methods Registrations"}

## Underscored and Globally Scoped DNS Node Names Registry {#dns-node-names-registry}

IANA is requested to register the following entries in the "Underscored and Globally Scoped DNS Node Names" registry defined in {{!RFC8552}}:

| RR Type | _NODE NAME | Reference |
|---|---|---|
| TXT | _validation-persist | This document |
| TXT | _ip-validation-persist | This document |
{: title="Underscored and Globally Scoped DNS Node Names Registrations"}

## ACME Directory Metadata Fields {#acme-directory-metadata-field}

IANA is requested to register the following entries in the "ACME Directory Metadata Fields" registry defined in {{!RFC8555}}, Section 9.7.6:

| Field Name | Field Type | Reference |
|---|---|---|
| accountHashPrefix | string | This document |
| issuerDomainNames | array of string | This document |
{: title="ACME Directory Metadata Fields Registrations"}

## ACME Error Types Registry {#acme-error-types-registry}

IANA is requested to register the following entry in the "ACME Error Types" registry defined in {{!RFC8555}}, Section 9.7.4:

- **Type**: persistExpired
- **Description**: A persistent validation record matching the CA and requesting account has an expired `persistUntil` timestamp
- **Reference**: This document

# Implementation Requirements and Guidance {#implementation-considerations}

When designing future extensions to this specification, new parameters SHOULD be designed to degrade gracefully when ignored by CAs that do not recognize them. Parameters that fundamentally change the security properties of the validation SHOULD NOT be introduced without a version negotiation mechanism.

## DNS Record Size Considerations {#dns-record-size}

The RDATA of the TXT record, which contains the `issue-value`, may become large, particularly if the `accounturi` is long. While the total size of a TXT record's RDATA can be up to 65,535 octets, it must be formatted as a sequence of one or more character-strings, where each string is limited to 255 octets in length.

**CA Implementation Guidelines:**

- CAs SHOULD endeavor to keep the `accounturi` values they generate reasonably concise to minimize the final record size.

**Client Implementation Guidelines:**

- Clients MUST properly handle the creation of TXT records where the RDATA exceeds 255 octets. As specified in {{!RFC1035}}, Section 3.3.14, the RDATA MUST be published as one or more character-strings, each containing at most 255 octets; the issue-value is the concatenation of their contents in order, with no separators added. The example below uses quoted master-file syntax ({{!RFC1035}}, Section 5.1); the quotation marks are not part of the data. For example:

~~~ dns
_validation-persist.example.com. IN TXT ("first-255-bytes..."
" ...remaining-bytes")
~~~
{: #ex-long-txt-record title="Multi-String TXT Record Format"}

## Domain Name Normalization Algorithm {#normalization-algorithm}

Clients and CAs MUST derive `domain_name` from `R`: the Validation Domain Name that the CA queries, with its leading `_validation-persist` label removed. A CA follows CNAME aliases at the Validation Domain Name as for any TXT query; `domain_name` remains `R`, never a CNAME target. For `ip` identifiers, they MUST instead use the Reverse Zone Domain Name `R` specified in {{ip-record-and-scope}}. The input MUST be a valid DNS name in US-ASCII, with internationalized labels represented as valid A-labels ({{!RFC5890}} and {{!RFC5891}}). An optional trailing root-label dot is permitted.

Name selection and validation MUST occur before this procedure. Implementations MAY reuse validation performed when accepting an ACME identifier or selecting a name for provisioning. Any conversion of user-supplied Unicode to A-labels occurs during name selection, outside this procedure. Implementations MUST reject non-ASCII input to this procedure.

Clients and CAs MUST apply the following steps in order:

1. Replace each ASCII uppercase letter `A` through `Z` with its lowercase equivalent. Leave all other octets unchanged.
2. Remove the trailing root-label dot, if present.

The resulting ASCII octets are the `domain_name` input to {{validation-record-format}}. Clients and CAs MUST NOT perform Unicode mapping, normalization, or conversion during this procedure. It does not apply to the explicit `*` opt-out defined in {{validation-record-format}}.

For example, `EXAMPLE.com.` produces `example.com`, and `XN--FA-HIA.example.` produces `xn--fa-hia.example`. The names `xn--fa-hia.example` and `fass.example` remain distinct.

## CA Implementation Guidelines {#ca-implementation-guidelines}

CAs implementing this validation method should consider:

- Establishing clear policies for Issuer Domain Name disclosure in Certificate Policies and Certification Practice Statements
- Creating account security monitoring and incident response procedures
- Providing clear documentation for clients on proper record construction

### Error Handling {#error-handling}

A CA implementing the "dns-persist-01" validation method SHOULD return the following ACME error types so that clients can tell validation failures apart:

- CAs SHOULD return a `malformed` error (as defined in {{!RFC8555}}) when the challenge response or TXT record has invalid syntax. Examples include:
   - an `ancestorDomain` value that is not a valid lowercase A-label proper ancestor within the DNS length limits ({{subdomain-implementation-requirements}});
   - a POST attempting to modify `ancestorDomain` on an accepted `pending`, `processing`, or finalized challenge ({{challenge-response-and-verification}});
   - a `{}` response for which `_validation-persist.<A>` would exceed the DNS length limits ({{subdomain-implementation-requirements}}, item 2);
   - duplicate occurrences of a recognized parameter;
   - invalid timestamp format in the `persistUntil` parameter;
   - missing mandatory `accounturi` parameter;
   - an `accounturi` that is not a well-formed hashed URI (for example, one not placed under the CA's `accountHashPrefix`, one using an algorithm identifier the CA does not accept, or one whose hash value is not valid base64url of the expected length); and
   - other syntactic violations of the formats specified in this document.

   For syntax errors detected in retrieved DNS TXT records (rather than the client's HTTP request payload) where this specification does not mandate a specific error type, a CA MAY return `unauthorized` instead of `malformed`. This choice lets the CA maintain a uniform authorization-failure disclosure policy.

- A retrieved record at the queried Validation Domain Name is otherwise qualifying for a validation attempt if it meets all of the following conditions:
   - it has valid syntax;
   - it uses an accepted hash algorithm;
   - its `issuer-domain-name` matches one of the challenge's `issuerDomainNames` ({{validation-record-format}});
   - it has an `accounturi` matching the live requesting account ({{verification-procedure}}); and
   - it satisfies any applicable `policy=wildcard` requirement.

   When a challenge validation attempt fails and at least one retrieved record is otherwise qualifying for that attempt but fails solely because its well-formed `persistUntil` timestamp has passed, the CA SHOULD return a `persistExpired` error (`urn:ietf:params:acme:error:persistExpired`, {{acme-error-types-registry}}). A CA MUST NOT return `persistExpired` if any record at the queried name satisfies validation, or if no otherwise qualifying record for the CA and the requesting account failed solely due to an expired `persistUntil` timestamp. The order in which records are evaluated does not affect this restriction. This mandatory eligibility restriction ensures that `persistExpired` never discloses the expiration state of a record belonging to a different account or CA. A CA MAY return an `unauthorized` error instead of `persistExpired` even when the eligibility condition holds (for example, to maintain a uniform authorization-failure disclosure policy). If it does so, clients and operators cannot distinguish record expiration from other authorization failures using the error type alone.

- CAs SHOULD return an `unauthorized` error (as defined in {{!RFC8555}}) when validation fails due to other authorization or policy issues (or when the CA declines to return `persistExpired` as described above), including:
   - the `accounturi` hashed URI in the DNS TXT record does not, when recomputed by the CA over the requesting account's retained keys (see {{verification-procedure}}), match any value the CA accepts for that account;
   - the `persistUntil` timestamp has expired and `persistExpired` is not returned;
   - the `issuer-domain-name` in the DNS TXT record does not match any of the values provided in the `issuerDomainNames` array of the challenge object;
   - the record lacks `policy=wildcard` when required for a wildcard authorization, an {{!RFC9444}} `subdomainAuthAllowed` authorization, or a challenge response specifying `ancestorDomain`; and
   - the request is declined under item 4 or 7 of {{subdomain-implementation-requirements}}, or is otherwise disallowed by the CA's policy.

These error types apply to validation attempts on individual challenges. When multiple records in the same resource record set (RRset) fail validation for different reasons and no record satisfies validation, reporting `persistExpired` when eligible (as specified above) and otherwise choosing between `unauthorized` and `malformed` according to CA policy (where this document does not mandate a specific error type) gives operators actionable feedback without requiring a fixed evaluation order between `unauthorized` and `malformed`. Records that cannot be parsed to a matching `issuer-domain-name` are ignored as specified in {{multiple-issuer-support}}. In the case of Just-in-Time validation (see {{just-in-time-validation}}), a CA that finds a pre-existing DNS TXT record that does not meet validation requirements does not return an error. Instead, the CA proceeds with the standard authorization flow by issuing a new `pending` challenge.

{{tab-error-handling-summary}} summarizes the challenge-response, DNS validation, and Just-in-Time outcomes for common failure conditions:

| Failure Condition | Stage | Challenge Outcome | Just-in-Time Outcome |
| :--- | :--- | :--- | :--- |
| Invalid `ancestorDomain` syntax, length, non-ancestor, or `R == A`; `ancestorDomain` sent for `ip` authorization; modifying `ancestorDomain` after acceptance; `{}` when `_validation-persist.<A>` is too long | Challenge POST (HTTP) | `malformed` (challenge state unchanged) | N/A |
| Request declined under the public-suffix guidance, or CA policy disallows ancestor | Challenge POST (HTTP) | `unauthorized` (challenge state unchanged) | No `valid` transition; `pending` challenge issued |
| Discontinued `<hash-alg>` token in TXT record ({{validation-record-format}}, item 3) | DNS Validation | `malformed` (`invalid`) | Fall back to `pending` challenge |
| Other TXT syntax violation (duplicate recognized parameter, missing `accounturi`, invalid `persistUntil` integer, malformed hashed URI) | DNS Validation | `malformed` or `unauthorized` (`invalid`) | Fall back to `pending` challenge |
| Record matches CA, live account, syntax, and scope, and fails solely due to expired `persistUntil` (no other record validates) | DNS Validation | `persistExpired` or `unauthorized` (`invalid`) | Fall back to `pending` challenge |
| `issuer-domain-name` mismatch, `accounturi` digest mismatch, or missing required `policy=wildcard` | DNS Validation | `unauthorized` (`invalid`) | Fall back to `pending` challenge |
{: #tab-error-handling-summary title="Summary of Validation Failure Conditions and ACME Outcomes"}

To mitigate potential denial-of-service risks from accounts with extensive key rollover history, CAs MUST rate-limit failed validation attempts per domain and account.

Failed-validation rate limits do not bound the computation of an individual validation attempt. Matching a historical thumbprint can require examining the full retained-key history, even when validation succeeds. Because the size of an account's retained-key history is determined by the account holder through `keyChange` requests, CAs SHOULD also apply rate limits or quotas to `keyChange` operations or to admitted validation attempts per account. Within an individual validation attempt, implementations can avoid repeated hashing across multiple TXT records in an RRset by computing the candidate hashed URIs once per retained key and comparing them across records. Retained prior-key thumbprints are used solely to recompute candidate hashed URIs during `dns-persist-01` verification; a CA MUST NOT use retained prior-key thumbprints to authenticate ACME requests or to resolve a JWK to an account.

## Client Implementation Guidelines {#client-implementation-guidelines}

ACME clients implementing this validation method should consider:

- Implementing secure DNS record management practices
- Providing clear user interfaces for managing persistent validation records
- Implementing validation record monitoring and alerting
- Handling validation failures, accommodating both `persistExpired` and generic `unauthorized` responses when a record's `persistUntil` timestamp has passed
- Considering the security implications of persistent records in their threat models

Clients that delegate record publication to third parties (such as DNS providers) MUST perform the pre-publication account check specified in {{pre-provisioning-records}} before transmitting inputs.

Clients that manage or provision `_validation-persist` records SHOULD inspect their own local provisioning state (rather than relying on a DNS lookup) before initiating a new order. Such clients SHOULD warn the operator if a record's `persistUntil` value has already expired or is likely to expire before validation completes. A client that skips this check risks initiating a validation attempt that the CA will reject with a `persistExpired` or `unauthorized` error (see {{ca-implementation-guidelines}}) instead of reporting an actionable, locally diagnosed cause.

When using `ancestorDomain` for a `dns` authorization ({{selecting-ancestor-domain}}), using the record domain `R` recorded in configuration or local provisioning state avoids extra DNS queries and ambiguity when multiple ancestors carry records. A client that instead searches DNS for an existing `_validation-persist` record can walk upward from the immediate parent of `A` (most-specific ancestor first) with a small step limit, avoiding queries at or above known public suffixes, single-label TLDs, and the DNS root unless configured for a single-organization suffix. Checking locally before submitting the challenge response that a candidate record's `issuer-domain-name` matches an entry in the challenge's `issuerDomainNames` ({{validation-record-format}}), includes `policy=wildcard`, and has not passed any `persistUntil` timestamp (and, when the current account key provisioned the record, matches the expected `accounturi` for `R` or `*`) avoids failing the challenge against unrelated or planted records on shared parent domains ({{subdomain-validation-risks}}). Note, however, that a record provisioned under an earlier account key can still be accepted by the CA ({{verification-procedure}}) even though it will not match a hash computed from the client's current key alone. Ancestor discovery applies only to `dns` authorizations ({{ip-record-and-scope}}).

## DNS Provider Considerations {#dns-provider-considerations}

DNS providers supporting this validation method should consider:

- Restricting who can create, change, or delete validation records
- Providing audit logging for validation record changes
- Considering dedicated interfaces or APIs for ACME validation record management
- Supporting workflows in which the provider computes and publishes `_validation-persist` records on a domain owner's behalf from the shareable values listed in {{pre-provisioning-records}}, none of which include private key material
- Never requesting or storing ACME account private keys, which no record-management operation described in this document requires
- Providing inventory and audit views of `_validation-persist` records to support the inventory practices described in {{pre-provisioning-records}}

# Examples {#examples}

## Basic Validation Example (FQDN Only) {#basic-validation-example}

For validation of "example.com" by a CA using "authority.example" as its Issuer Domain Name, where the validation should only apply to "example.com":

1.  CA provides challenge object with a list of valid Issuer Domain Names:

    ~~~json
    {
      "type": "dns-persist-01",
      "url": "https://ca.example/acme/chall/5678",
      "status": "pending",
      "issuerDomainNames": ["authority.example", "ca.example.net"]
    }
    ~~~

2.  Client chooses one of the provided Issuer Domain Names (e.g., "authority.example"), computes its hashed URI as shown in {{hashed-uri-example}}, and provisions a DNS TXT record (note the absence of a `policy` parameter for scope):

    ~~~ dns
    _validation-persist.example.com. IN TXT ("authority.example;"
    " accounturi=https://ca.example/account-hash/"
    "sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw")
    ~~~

3.  CA validates the record through DNS queries. This validation is sufficient only for "example.com".


## Hashed-URI Example {#hashed-uri-example}

This example validates "example.com" using a hashed URI ({{validation-record-format}}) and serves as a test vector. The hash inputs are:

- `length_of_domain`: `0x0b` (the domain `example.com` is 11 octets)
- `domain_name`: `example.com`
- `key`: the 43 ASCII octets of `NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs`, the unpadded base64url encoding of the account's current SHA-256 JWK Thumbprint {{!RFC7638}} (the example thumbprint from {{!RFC7638}}, Section 3.1)
- `account_URL`: `https://ca.example/acct/123`

Computing `SHA-256(length_of_domain || domain_name || key || account_URL)` and applying unpadded base64url encoding to the 32-octet digest yields:

~~~
5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw
~~~

With a CA `accountHashPrefix` of `https://ca.example/account-hash/` and the `sha-256` algorithm identifier, the hashed URI is `https://ca.example/account-hash/sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw`. Using Issuer Domain Name "authority.example", the client provisions:

~~~ dns
_validation-persist.example.com. IN TXT ("authority.example;"
" accounturi=https://ca.example/account-hash/"
"sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw")
~~~
{: #ex-hashed-uri title="Hashed-URI Validation Record"}

To instead opt out of the domain-correlation mitigation ({{validation-record-format}}), the client sets `domain_name` to the single octet `*` (so `length_of_domain` is `0x01`), keeping the same `key` and `account_URL`. Computing `SHA-256(0x01 || "*" || key || account_URL)` and applying unpadded base64url encoding to the digest yields:

~~~
NpDnSwUthQK8zCgFdefYxAdAVPnygMLbs9US6oO-5ug
~~~

The resulting hashed URI, `https://ca.example/account-hash/sha-256/NpDnSwUthQK8zCgFdefYxAdAVPnygMLbs9US6oO-5ug`, does not depend on the domain and can be reused across domains (see {{validation-record-format}}).

For a record provisioned at `_validation-persist.XN--FA-HIA.example.`, {{normalization-algorithm}} produces `domain_name` `xn--fa-hia.example`. Keeping the `key` and `account_URL` of the "example.com" example, the hash inputs are:

- `length_of_domain`: `0x12` (the domain `xn--fa-hia.example` is 18 octets)
- `domain_name`: `xn--fa-hia.example`

Computing `SHA-256(length_of_domain || domain_name || key || account_URL)` and applying unpadded base64url encoding to the 32-octet digest yields:

~~~
dc3zyKBvsc9_vyWQoZSyM4fgcN0d7jnr61730r7ekAw
~~~


## Pre-Provisioned Validation Example {#pre-provisioned-validation-example}

This example illustrates the pre-provisioning flow described in {{pre-provisioning-records}}: the `_validation-persist` record for "example.com" is computed and published before any order exists, by a party other than the ACME client.

1.  Prerequisite: the domain owner's ACME client has an account with the CA; the account URL `https://ca.example/acct/123` was returned in the `Location` header of the newAccount response. No order, authorization, or challenge object exists yet.

2.  The CA's directory `meta` object advertises the values needed to pre-provision a record:

    ~~~json
    {
      "meta": {
        "accountHashPrefix": "https://ca.example/account-hash/",
        "issuerDomainNames": ["authority.example", "ca.example.net"]
      }
    }
    ~~~
    {: #ex-preprovisioned-directory-meta title="Directory Metadata for Pre-Provisioning"}

    Per {{directory-issuer-domain-names}}, a name taken from this array satisfies the `issuer-domain-name` requirement of later `dns-persist-01` challenges from this CA while advertised, subject to the stability requirement and migration-period guidance in {{pre-provisioning-records}}.

3.  Before handing off the record inputs, the account holder satisfies the account check in {{pre-provisioning-records}} using an authenticated POST-as-GET request to the exact account URL used in the digest, `https://ca.example/acct/123`. The request has a zero-length payload: the JWS `payload` field is present and set to the empty string `""`, rather than carrying the JSON object `{}`. The client follows no redirects and verifies that the returned account object's `status` is `valid`. A successful proof for an unchanged (account URL, account key) pair can satisfy this later pre-publication check, as permitted in {{pre-provisioning-records}}. If the check is not satisfied, the handoff does not proceed.

    The domain owner then transmits the following values through an authenticated channel to the DNS operator managing the zone, a party distinct from the ACME client:

    - the account URL `https://ca.example/acct/123`;
    - the account key's public JWK Thumbprint (encoded as `NzbLsXh8uDCcd-6MNwXF4W_7noWXFZAfHkxZsRGC9Xs`, as in {{hashed-uri-example}});
    - the domain input `example.com`;
    - the `accountHashPrefix` `https://ca.example/account-hash/`;
    - the algorithm identifier `sha-256`;
    - the selected Issuer Domain Name `authority.example`; and
    - the `policy` and `persistUntil` parameters the domain owner chooses (none in this example).

    The client retains the account private key; no private key material is shared. Using the same inputs and result as {{hashed-uri-example}}, that party computes the hashed URI and publishes the TXT record:

    ~~~ dns
    _validation-persist.example.com. IN TXT ("authority.example;"
    " accounturi=https://ca.example/account-hash/"
    "sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw")
    ~~~
    {: #ex-preprovisioned title="Pre-Provisioned Validation Record"}

4.  At any later time, the client submits a newOrder request and receives an order object containing authorization URLs. The client retrieves the referenced authorization objects using POST-as-GET requests ({{!RFC8555}}, Section 7.5). If an authorization is pending with a `dns-persist-01` challenge, the client responds by POSTing `{}` to the challenge URL, and the CA validates the already-present record using fresh DNS data; no DNS change is needed at order time. A CA implementing Just-in-Time validation ({{just-in-time-validation}}) may instead complete the authorization's validation without a challenge-response exchange, subject to the restrictions in that section.


## Wildcard Validation Example {#wildcard-validation-example}

For validation of "*.example.com" by a CA using "authority.example" as its Issuer Domain Name:

1.  The CA provides a challenge object similar to the basic example, containing an `issuerDomainNames` array.

2.  Client chooses one of the provided Issuer Domain Names (e.g., "authority.example") and provisions a DNS TXT record at the base domain's Validation Domain Name, including `policy=wildcard`:

    ~~~ dns
    _validation-persist.example.com. IN TXT ("authority.example;"
    " accounturi=https://ca.example/account-hash/"
    "sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw;"
    " policy=wildcard")
    ~~~
    {: #ex-wildcard-validation title="Wildcard Policy Validation Record"}

3.  CA validates the record through DNS queries for the requested wildcard identifier. The resulting authorization has identifier value `example.com` and `wildcard` set to `true` ({{!RFC8555}}, Sections 7.1.3 and 7.1.4). The CA may also use this record to validate "example.com" and specific subdomains like "www.example.com", subject to {{wildcard-certificate-validation}} and {{subdomain-certificate-validation}}. Any required authorizations for those identifiers are handled separately; the record's applicability does not automatically complete them.

## Validation Example with persistUntil {#persist-until-example}

For validation of "example.com" with an explicit expiration date:

1.  The CA provides a challenge object similar to the basic example, containing an `issuerDomainNames` array.

2.  Client chooses one of the provided Issuer Domain Names (e.g., "authority.example") and provisions a DNS TXT record including `persistUntil`:

    ~~~ dns
    _validation-persist.example.com. IN TXT ("authority.example;"
    " accounturi=https://ca.example/account-hash/"
    "sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw;"
    " persistUntil=1924992000")
    ~~~
    {: #ex-persist-until title="Validation Record with Expiration Time"}

3.  CA validates the record. This validation is sufficient only for "example.com" and will not be considered valid after the specified timestamp (2031-01-01T00:00:00Z).

## Wildcard Validation Example with persistUntil {#wildcard-persist-until-example}

For validation of "*.example.com" with an explicit expiration date:

1.  The CA provides a challenge object similar to the basic example, containing an `issuerDomainNames` array.

2.  Client chooses one of the provided Issuer Domain Names (e.g., "authority.example") and provisions a DNS TXT record including `policy=wildcard` and `persistUntil`:

    ~~~ dns
    _validation-persist.example.com. IN TXT ("authority.example;"
    " accounturi=https://ca.example/account-hash/"
    "sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw;"
    " policy=wildcard;"
    " persistUntil=1924992000")
    ~~~
    {: #ex-wildcard-persist-until title="Wildcard Validation Record with Expiration Time"}

3.  CA validates the record for the requested wildcard identifier, with authorization scope as described in {{wildcard-validation-example}}. The record may also support validation for "example.com" and specific subdomains, but does not automatically complete authorizations for those identifiers. The validation will not be considered valid after the specified timestamp (2031-01-01T00:00:00Z).

--- back

# Acknowledgments {#acknowledgments}
{:unnumbered}

The authors acknowledge prior community work that directly informed this specification:

- The CA/Browser Forum ballot work on persistent / static DNS Domain Control Validation signals in the Baseline Requirements {{cabf-br}}. The authors acknowledge in particular Ballot SC-082 ("Clarify CA Assisted DNS Validation under 3.2.2.4.7", authored by Michael Slaughter) and Ballot SC-088 ("DNS TXT Record with Persistent Value DCV Method", also authored by Michael Slaughter). SC-088 was adopted on October 9, 2025, with an effective date of November 10, 2025. These efforts provided the policy framing and initial industry discussion motivating standardization of a reusable ACME DNS validation record.
- The formal and empirical security analysis of static / persistent DCV methods performed by Henry Birge-Lee ("Proof of static DCV security" presentation, the "Security of SC-082 Redux" paper {{birgelee-sc082-security}}, and related research). This analysis helped clarify the threat model and informed the security considerations in this document.
- The Delegated DNS Domain Validation (DDDV) Threat Modeling Tiger Team discussions and document ("Validation SC - Delegated DNS Domain Validation (DDDV) Threat Model"), whose participants contributed to broad threat enumeration. Notable contributors include Michael Slaughter (Amazon Trust Services), Corey Bonnell (DigiCert), Clint Wilson (Apple), and Martijn Katerbarg (Sectigo).

The authors also thank members of the ACME Working Group and CA/Browser Forum who provided early review, critique, and operational perspectives on persistent validation records.

# Change Log {#changelog}
{:unnumbered}

RFC Editor: please remove this section before publication.

## Since draft-ietf-acme-dns-persist-02
{:unnumbered}

- Added the request-only `ancestorDomain` challenge-response field ({{subdomain-certificate-validation}}) to select a `policy=wildcard` validation record at a proper ancestor, with interaction rules for {{!RFC9444}}, public-suffix guidance (including nested public suffixes), DNS length checks on every `R` (including rejecting `{}` with `malformed` when `_validation-persist.<A>` exceeds 253 octets), and client discovery advice ({{client-implementation-guidelines}}).
- Added IP address validation ({{ip-address-validation}}) through `_ip-validation-persist.<R>` records in the reverse DNS namespace (`in-addr.arpa` and `ip6.arpa`), with per-address scope and pre-alias reverse-name digest inputs.
- Added the `persistExpired` error type ({{acme-error-types-registry}}, {{error-handling}}) for otherwise qualifying records of the live requesting account whose `persistUntil` has passed, with an `unauthorized` fallback and informative multi-record failure guidance.
- Added informative deployment guidance ({{certificate-key-attestation}}) on combining `dns-persist-01` with certificate-key attestation through CA order policy.
- Changed domain hash canonicalization ({{normalization-algorithm}}) from Unicode/IDNA processing to ASCII lowercasing and trailing-dot removal on validated A-label inputs, specified that CAs follow CNAME aliases at the Validation Domain Name while keeping `domain_name` bound to the pre-alias `R` (or `*`), and changed `issuer-domain-name` matching ({{validation-record-format}}) to exact Simple String Comparison (adding an `XN--FA-HIA.example.` test vector in {{hashed-uri-example}}).
- Changed recognized parameter-name matching (`accounturi`, `policy`, `persistUntil`) to ASCII case-insensitive comparison while rejecting duplicate occurrences of recognized parameter names, restricted `persistUntil` to unsigned ASCII digits not exceeding `2^63 - 1` (valid at or before `T`, expired after `T`), and specified that CAs evaluate each record individually without combining parameters across records ({{validation-record-format}}).
- Required clients and any delegated publishers to verify before constructing a hashed URI that `accountHashPrefix` is present in the directory and contains only {{!RFC8659}} `value` octets and `%` plus {{!RFC3986}} `reserved`/`unreserved` characters, with no fallback to another value ({{validation-record-format}}).
- Consolidated prior-key retention rules ({{verification-procedure}}, {{account-key-rotation}}): scoped the retention start point to when the CA began accepting key-bound `dns-persist-01` records, applied immediate override on any account status other than `valid`, required that CAs never reassign an account URL to a different account ({{pre-provisioning-records}}), noted `keyChange` / attempt rate limiting ({{error-handling}}), and removed the `-02` recommendation to restrict Just-in-Time validation to accounts with a prior `dns-persist-01` validation ({{just-in-time-validation}}).
- Extended {{pre-provisioning-records}} stability rules to names advertised in directory `issuerDomainNames` (requiring inclusion in issued challenges while advertised and recommending a migration period on removal), clarified that Just-in-Time validated authorizations include the `valid` `dns-persist-01` challenge object, and recommended reserving validation labels on shared parent domains ({{subdomain-validation-risks}}).
- Added Privacy Considerations ({{privacy-considerations}}) and Implementation Status ({{implementation-status}}, noting that earlier-draft deployments sharing the `dns-persist-01` label do not interoperate with `-03`), upgraded the DNSSEC reference to {{!RFC9364}}, added explanatory figures and summary tables, corrected the IANA directory metadata registry name, and edited the prose for clarity and consistent actor attribution.

## Since draft-ietf-acme-dns-persist-01
{:unnumbered}

- The `accounturi` is now a hashed URI that cryptographically binds the ACME account to the domain being validated. In -01 the `accounturi` was the ACME account URL, or another URI that uniquely and permanently identified a single account ({{!RFC8657}}, Section 5.4), with no cryptographic binding; that form is no longer accepted.
- Defined the hashed URI as `<accountHashPrefix><hash-alg>/<base64url hash value>`, advertised via the new `accountHashPrefix` directory metadata field and carrying an in-URI hash-algorithm identifier for algorithm agility (the registered `sha-256` Hash Name String is mandatory).
- Specified the digest as `H(length_of_domain || domain_name || key || account_URL)`, with a leading length octet delimiting the domain name.
- Added a mandatory domain-omitted opt-out (`domain_name` set to `*`) for correlation opt-out and cross-domain record reuse.
- Retired the temporal key rotation window mechanisms (keyRotationPeriod directory metadata and keyRolloverWindow); prior-key acceptance is governed by CA retention.
- Specified uniform witnessed-thumbprint retention as a MUST bounded only by account lifetime, with account deactivation immediately overriding acceptance and permitting pruning.
- Replaced the -01 reuse-unaffected text with a requirement that CAs clamp authorization `expires` to `persistUntil`.
- Upgraded `accountHashPrefix` and account-URL stability to a MUST with mandatory dual acceptance during migration periods.
- Added a pre-publication account-resource proof (authenticated POST-as-GET with empty payload `""` verifying `status=valid`) before delegating record publication.
- Mitigated recomputation denial-of-service by evaluating current keys first, evaluating historical keys in reverse chronological order, and rate-limiting failed validations.
- Renamed the challenge object's plural field from `issuer-domain-names` to `issuerDomainNames` to align with ACME's camelCase convention ({{!RFC8555}}); the singular DNS `issuer-domain-name` parameter and `persistUntil` are unchanged (#56).
- Clarified that `key` in the digest is the 43-octet unpadded base64url encoding of the SHA-256 JWK Thumbprint {{!RFC7638}}, not the 32 raw digest octets, matching the encoding already used in ACME key authorizations ({{!RFC8555}}, Section 8.1).
- Required `sha-256` support in both clients and CAs for interoperability, moved `<hash-alg>` to the Hash Name String registry defined by {{!RFC6920}}, excluded truncated digests shorter than 256 bits, and removed the document-local slash-to-hyphen substitution rule.
- Clarified the relationship between the CAA `accounturi` and the `dns-persist-01` `accounturi` in the Introduction and {{caa-interaction}}, citing {{!RFC8657}}, Section 3.1.
- Added a privacy caveat noting that a cleartext `http-01` key authorization exposes the same account-key thumbprint used in the hashed URI, leaving account-URL entropy as the remaining protection.
- Added client-side guidance ({{client-implementation-guidelines}}) to check `persistUntil` against local provisioning state before initiating an order (#38).
- Removed DNS TTL as a validation data reuse limit; reuse remains subject to `persistUntil` and the CA's Validation Data Reuse Period. Renamed {{validation-data-reuse}} accordingly and removed the related TTL-specific implementation guidance (#42).
- Corrected the stale Account Key Rotation text: the account URL is stable, but the hashed `accounturi` changes with the account key, and continued acceptance of a prior key is governed by {{verification-procedure}}.
- Added the required directory `issuerDomainNames` metadata field and the rule that D is a subset of C for the challenge object's `issuerDomainNames`; the rule that C is a subset of I against `caaIdentities` applies when the CA advertises `caaIdentities` (#60).
- Added a pre-provisioned validation example and clarified that computing and publishing a `_validation-persist` record requires no private-key operation and can be performed out of band by a party other than the ACME client, at any point after account creation.
