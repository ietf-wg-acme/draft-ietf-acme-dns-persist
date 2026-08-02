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

This document specifies "dns-persist-01", a new validation method for the Automated Certificate Management Environment (ACME) protocol. This method allows a Certification Authority (CA) to verify control over a domain by confirming the presence of a persistent DNS TXT record containing CA and account identification information. This method is particularly suited for environments where traditional challenge methods are impractical, such as multi-tenant hosting platforms, enterprise DNS environments, and IoT deployments. The validation method is designed with a strong focus on security and robustness, incorporating widely adopted industry best practices for persistent domain control validation. This design aims to make it suitable for Certification Authorities operating under various policy environments, including those that align with the CA/Browser Forum Baseline Requirements.

--- middle

# Introduction {#introduction}

The Automated Certificate Management Environment (ACME) protocol {{!RFC8555}} defines mechanisms for automating certificate issuance and domain validation. The existing challenge methods, "http-01" and "dns-01", require real-time interaction between the ACME client and the domain's infrastructure during the validation process. While effective for many use cases, these methods present challenges in certain deployment scenarios.

Examples include:

- Edge compute and multi-tenant hosting platforms where the entity managing the DNS zone is distinct from the tenant subscribing to the certificate.
- Organizations that wish to pre-validate domains and batch issuance operations offline or at a later time.
- Environments with strict change management processes where DNS modifications require approval workflows.
- Scenarios requiring wildcard certificates where domain control is proven once and reused over an extended period.
- Internet of Things (IoT) deployments where devices may not be able to host an HTTP service or coordinate DNS updates in real-time.

This document defines a new ACME challenge type, "dns-persist-01". This method proves control over a Fully Qualified Domain Name (FQDN) by confirming the presence of a persistent DNS TXT record containing CA and account identification information.

The record format is based on the "issue-value" syntax from {{!RFC8659}}, incorporating an `issuer-domain-name` and a mandatory `accounturi` parameter whose value is defined in {{validation-record-format}}. The parameter name is reused from the CAA `accounturi` parameter defined in {{!RFC8657}}, Section 3; {{caa-interaction}} explains the relationship between them. The `accounturi` is a hashed URI that cryptographically binds the account to the domain being validated. This design provides strong binding between the domain, the CA, and the entity requesting validation.

## Robustness and Alignment with Industry Best Practices {#robustness-and-alignment}

This validation method is designed to provide a robust and persistent mechanism for domain control verification within the ACME protocol. Its technical design incorporates widely adopted security principles and best practices for domain validation, ensuring high assurance regardless of the specific CA policy environment. These principles include, but are not limited to:

1.  The use of a well-defined, unique DNS label (e.g., "_validation-persist") for persistent validation records, minimizing potential conflicts.
2.  Explicit binding of the domain validation to an ACME account through a unique identifier, establishing clear accountability and enhancing security against unauthorized use.

Certification Authorities operating under various trust program requirements will find this technical framework suitable for their domain validation needs, as its design inherently supports robust and auditable validation practices.

# Conventions and Definitions {#conventions-and-definitions}

{::boilerplate bcp14-tagged}

**DNS TXT Record Persistent DCV Domain Label**
:   The label "_validation-persist" as specified in this document. This label is consistent with industry practices for persistent domain validation.

**Validation Domain Name**
:   The domain name at which the validation TXT record is provisioned. It is formed by prepending the DNS TXT Record Persistent DCV Domain Label to the FQDN being validated.

    This term follows the precedent set by {{!RFC8555}}, Section 8.4, which uses "validation domain name" for the analogous `_acme-challenge.<FQDN>` label in the dns-01 challenge. This document avoids the term "Authorization Domain Name" because the CA/Browser Forum Baseline Requirements {{cabf-br}} define it as the FQDN used to obtain authorization, without any label prepended.

**Issuer Domain Name**
:   A domain name disclosed by the CA in Section 4.2 of the CA's Certificate Policy and/or Certification Practices Statement to identify the CA for the purposes of this validation method.

    The directory and challenge-object fields that carry Issuer Domain Names, and their relationship to `caaIdentities`, are defined in {{directory-issuer-domain-names}} and {{challenge-object}}.

**Validation Data Reuse Period**
:   The period during which a CA may rely on validation data, as defined by the CA's practices and applicable requirements.

**persistUntil**
:   An optional parameter in the validation record that specifies the timestamp after which the validation record should no longer be considered valid by CAs. The value MUST be a base-10 encoded integer representing a UNIX timestamp (the number of seconds since 1970-01-01T00:00:00Z ignoring leap seconds). Client pre-flight guidance appears in {{client-implementation-guidelines}}.

# The "dns-persist-01" Challenge {#dns-persist-01-challenge}

The "dns-persist-01" challenge allows an ACME client to demonstrate control over an FQDN by proving it can provision a DNS TXT record containing specific, persistent validation information. The validation information links the FQDN to both the Certificate Authority performing the validation and the account requesting the validation.

When an ACME client accepts a "dns-persist-01" challenge, it proves control by provisioning a DNS TXT record at the Validation Domain Name. Unlike the existing "dns-01" challenge, this record is designed to persist and may be reused for multiple certificate issuances over an extended period.

## Challenge Object {#challenge-object}

The challenge object for "dns-persist-01" contains the following fields:

- **type** (required, string): The string "dns-persist-01"
- **url** (required, string): The URL to which a response can be posted
- **status** (required, string): The status of this challenge
- **issuerDomainNames** (required, array of strings): A list of one or more Issuer Domain Names. The client MUST choose one of these domain names to include in the DNS TXT record. The challenge is successful if a valid TXT record is found that uses any one of the provided domain names.

  Each string in the array MUST be a domain name that complies with the following normalization rules:

  1.  The domain name MUST be represented in A-label format (Punycode, {{!RFC5890}}).
  2.  All characters MUST be lowercase.
  3.  The domain name MUST NOT have a trailing dot.

  The server MUST ensure the array is not empty. Servers MUST NOT send more than 10 issuer domain names. This limit serves as a practical measure to prevent denial-of-service vectors against clients. Clients MUST consider a challenge malformed if the `issuerDomainNames` array is empty or if it contains more than 10 entries, and MUST reject such challenges. Each domain name MUST NOT exceed 253 octets in length.

The following shows an example challenge object:

~~~json
{
  "type": "dns-persist-01",
  "url": "https://ca.example/acme/authz/1234/0",
  "status": "pending",
  "issuerDomainNames": ["authority.example", "ca.example.net"]
}
~~~
{: #fig-challenge-object title="Example dns-persist-01 Challenge Object"}

## Directory Metadata for Issuer Domain Names {#directory-issuer-domain-names}

A CA offering the `dns-persist-01` challenge type MUST advertise an `issuerDomainNames` array in the `meta` object of its directory ({{!RFC8555}}, Section 7.1.1). This directory-level array lets a client discover Issuer Domain Names before opening any authorization, so it can pre-provision `_validation-persist` records per {{pre-provisioning-records}} without an interactive order.

The directory `issuerDomainNames` array MUST use the same normalization and length rules as the challenge object's `issuerDomainNames` array ({{challenge-object}}): each entry MUST be a lowercase A-label domain name with no trailing dot and MUST NOT exceed 253 octets. The array MUST NOT be empty and MUST NOT contain more than 10 entries.

Let D be the set of values in the directory `issuerDomainNames` array, and let I be the set of values in the directory `caaIdentities` array ({{!RFC8555}}, Section 7.1.1), represented as lowercase A-label domain names with no trailing dot. This document does not require a CA to advertise `caaIdentities`; a CA that does not perform CAA validation has no names to disclose there. For each `dns-persist-01` challenge object the CA issues, let C be the set of values in that challenge's `issuerDomainNames` array. The CA MUST satisfy these requirements:

- Every value in D MUST appear in C. A client that pre-provisions a record using a name from D is therefore guaranteed that the name will satisfy the `issuer-domain-name` requirement of any `dns-persist-01` challenge that CA issues later, without first opening an authorization.
- C MAY include values beyond D while remaining within the 10-entry maximum in {{challenge-object}}, so a CA can offer challenge-specific names in addition to its pre-provisioning baseline.
- If the CA advertises `caaIdentities`, every value in C MUST also appear in I. This prevents a CA that participates in CAA validation from accepting an `issuer-domain-name` it has not also disclosed through `caaIdentities`.

For these subset comparisons, the CA MUST compare `caaIdentities` values, when advertised, after applying the same lowercase A-label and trailing-dot rules used for D and C.

`caaIdentities` values of publicly trusted CAs are kept distinct by the CA/Browser Forum disclosure and audit regime. Identities of CAs outside that regime have no equivalent protection against another CA recognizing the same name, so a domain owner who adds such an identity to a CAA policy accepts that reuse caveat.

A client MUST treat a missing or nonconforming directory `issuerDomainNames` array as unavailable for pre-provisioning. A missing or nonconforming `caaIdentities` array does not affect pre-provisioning; it only makes the I comparisons unavailable. A client MAY still process an interactive `dns-persist-01` challenge. When D is available, the client MUST consider a challenge malformed if any value in D is absent from C. When I is available, the client MUST consider a challenge malformed if any value in C is absent from I.

# Challenge Response and Verification {#challenge-response-and-verification}

To respond to the challenge, the ACME client provisions a DNS TXT record at the Validation Domain Name of the domain being validated. The Validation Domain Name is formed by prepending the label "_validation-persist" to the domain name being validated.

For example, if the domain being validated is "example.com", the Validation Domain Name would be "_validation-persist.example.com".

The client indicates it is ready for validation by POSTing an empty JSON object (`{}`) to the challenge URL, following the procedure defined in {{!RFC8555}}, Section 7.5.1.

## Validation Record Format {#validation-record-format}

The RDATA of this TXT record MUST fulfill the following requirements:

1.  The RDATA value MUST conform to the issue-value syntax defined in {{!RFC8659}}, Section 4.2. To ensure forward compatibility, the server MUST ignore any parameter within the issue-value that has an unrecognized tag.

2.  The `issuer-domain-name` portion of the issue-value MUST be one of the Issuer Domain Names provided by the CA in the `issuerDomainNames` array of the challenge object. If the `issuer-domain-name` does not match any of the provided values, the CA MUST reject the record.

3.  The issue-value MUST contain an `accounturi` parameter whose value is a hashed URI identifying the ACME account requesting validation. A hashed URI cryptographically binds the account to the domain being validated without publishing the account URL in cleartext. It is formed by concatenating the CA's `accountHashPrefix`, a hash-algorithm identifier, a "/" separator, and a base64url hash value:

    ~~~
    <accountHashPrefix><hash-alg>/<base64url hash value>
    ~~~

    A CA that advertises support for the dns-persist-01 challenge type MUST advertise an `accountHashPrefix` string in the `meta` object of its directory ({{!RFC8555}}, Section 7.1.1). This prefix places hashed account URIs under the CA's infrastructure and lets a client construct them from the directory without a per-provisioning request to the CA. The CA MUST choose a prefix such that appending the remaining components yields a valid URI (for example, `https://ca.example/account-hash/`).

    `<hash-alg>` MUST be the exact Hash Name String registered in the "Named Information Hash Algorithm Registry" defined by {{!RFC6920}}, Section 9.4. Both clients and CAs implementing dns-persist-01 MUST support the registered `sha-256` Hash Name String, which identifies SHA-256 {{FIPS180-4}}: a client MUST be able to compute, and a CA MUST accept, a hashed URI that uses this token, so that every conforming implementation interoperates without prior negotiation. A CA MAY accept other registered algorithms and MUST document each additional algorithm it accepts. An additional algorithm MUST have a registered digest length of at least 256 bits and MUST remain collision resistant at the time of use. A client MAY use an optional algorithm only after establishing from the CA's documentation that the CA accepts it. This document defines no separate algorithm-negotiation mechanism.

    `<base64url hash value>` is the base64url encoding {{!RFC4648}}, with trailing padding (`=`) omitted, of the hash digest computed with the identified algorithm over the following octet string:

    ~~~
    H(length_of_domain || domain_name || key || account_URL)
    ~~~

    where `H` is the identified hash algorithm and `||` denotes octet-string concatenation. The inputs are:

    1. `length_of_domain` is a single octet giving the length in octets of `domain_name`. A domain name is limited to 255 octets ({{!RFC1035}}, Section 2.3.4), so a single octet always suffices. This length prefix delimits `domain_name` from the fields that follow.
    2. `domain_name` is, by default, the FQDN being validated: the Validation Domain Name with its leading `_validation-persist` label removed and no trailing dot, in the normalized lowercase A-label form produced by {{normalization-algorithm}}. This form is US-ASCII. This is the FQDN at which the `_validation-persist` record is provisioned, not the FQDN of the certificate ultimately requested; binding the record to its own domain is what makes the hashed URI domain-specific (see {{hashed-uri-security}}).

       As an exception, a client MAY set `domain_name` to the single US-ASCII octet `*` (0x2a; `length_of_domain` is then 0x01) to opt out of the domain-correlation mitigation. The resulting hashed URI does not depend on the domain and MAY therefore be reused across domains. A CA MUST accept a hashed URI computed with `domain_name` set to `*`. The privacy trade-off of this opt-out is described in {{hashed-uri-security}}.
    3. `key` is the 43 ASCII octets of the unpadded base64url encoding {{!RFC4648}} of the SHA-256 JWK Thumbprint {{!RFC7638}} of a public key associated with the ACME account. This is the encoding used for the thumbprint component of an ACME key authorization ({{!RFC8555}}, Section 8.1), so a client can reuse that component without decoding it. When provisioning a `_validation-persist` record, the client MUST use the account's current key. A CA MUST accept a hashed URI generated with the current key associated with the account. Additionally, a CA MUST accept a hashed URI generated with a prior key previously associated with the account that is known to the CA per the key retention obligation in {{verification-procedure}}, subject to the deactivation and key-rotation limits stated there. This allows a record provisioned before a key rotation to continue to validate. The rationale for these requirements is given in {{hashed-uri-security}}.
    4. `account_URL` is the ACME account URL ({{!RFC8555}}, Section 7.3). A URI is US-ASCII per {{!RFC3986}}, so it is encoded as those US-ASCII octets.

    The CA MUST verify that the hashed URI in the DNS record authorizes the ACME account making the request, by recomputing it as described in {{verification-procedure}}; if no recomputation matches, the CA MUST reject the record.

4.  The issue-value MAY contain a `policy` parameter. If present, this parameter modifies the validation scope. The `policy` parameter follows the 'tag=value' syntax from {{!RFC8659}}. Comparison of the values defined for this parameter MUST be case-insensitive (e.g., "WILDCARD" and "wildcard" are equivalent).

    Note: The requirement to ignore unrecognized parameters (item 1) ensures forward compatibility, allowing future extensions without breaking existing implementations, consistent with ACME's extensibility model ({{!RFC8555}}). The explicit case-insensitivity requirement is necessary to ensure consistent behavior across implementations; without it, some CAs might reject unknown parameter values, preventing protocol evolution.

    The following value for the `policy` parameter is defined with respect to subdomain and wildcard validation:

    - `policy=wildcard`: If this value is present, the CA MAY consider this validation sufficient for issuing certificates for the validated FQDN, for specific subdomains of the validated FQDN (as covered by wildcard scope or specific subdomain validation rules), and for wildcard certificates (e.g., `*.example.com`). See {{wildcard-certificate-validation}} and {{subdomain-certificate-validation}}.

    If the `policy` parameter is absent, or if its value is anything other than `wildcard`, the CA MUST proceed as if the `policy` parameter were not present (i.e., the validation applies only to the specific FQDN).

5.  The issue-value MAY contain a `persistUntil` parameter. If present, the value MUST be a base-10 encoded integer representing a UNIX timestamp (the number of seconds since 1970-01-01T00:00:00Z ignoring leap seconds). If the value is not a valid base-10 integer, the CA MUST treat the record as malformed and reject it. After the specified timestamp, a CA MUST NOT use the record for a new validation attempt or reuse validation data previously obtained from it.

This specification defines the following case-handling rules for parameter values in dns-persist-01 records:

- `accounturi`: The value is a hashed URI. CAs MUST compare `accounturi` values using Simple String Comparison per {{!RFC3986}}, Section 6.2.1, with no case-folding or other normalization.
- `policy`: Case-insensitive, as specified in item 4 above.
- `persistUntil`: The value is a base-10 integer. Case does not apply.

For example, for validation of the FQDN "example.com" with issuer domain name "authority.example", where the CA advertises an `accountHashPrefix` of "https://ca.example/account-hash/" and the account's hashed URI (see {{hashed-uri-example}} for the full computation) is "https://ca.example/account-hash/sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw", the DNS TXT record would contain:

~~~ dns
_validation-persist.example.com. IN TXT ("authority.example;"
" accounturi=https://ca.example/account-hash/"
"sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw")
~~~
{: #ex-basic-validation title="Basic Validation TXT Record"}

## Verification Procedure {#verification-procedure}

The ACME server verifies the challenge by performing a DNS lookup for TXT records at the Validation Domain Name. It then iterates through the returned records to find one that conforms to the required structure. For a record to be considered valid, its `issuer-domain-name` value MUST match one of the values provided in the `issuerDomainNames` array from the challenge object, and its `accounturi` parameter MUST be a hashed URI that identifies the requesting account, as defined in {{validation-record-format}}. When comparing issuer domain names, the server MUST adhere to the normalization rules specified in {{challenge-object}}. The server also interprets any `policy` parameter values according to this specification. If no record meeting all requirements is found, the server MUST treat the challenge as failed.

To verify a hashed-URI `accounturi` value, the CA already knows the requesting account's URL and the FQDN being validated. The CA computes the expected hashed URI as defined in {{validation-record-format}} over each ACME public key it retains for that account (see below); for each such key it computes the value twice, once using the FQDN being validated as `domain_name` and once using the domain-omitted opt-out form (`domain_name` set to `*`). The CA treats the record as satisfying the requirement if its `accounturi` value equals any of these computed hashed URIs under Simple String Comparison ({{!RFC3986}}, Section 6.2.1). Retaining an account's hashed URIs that previously passed validation allows records provisioned before a key rotation to continue validating.

To reuse these hashed URIs, a CA has a data-retention obligation: it MUST retain an association between each ACME account and the sha256 hashes that have passed initial DNS-PERSIST-01 validation along with for which domain. The following rules govern this retained data and the acceptance of the keys behind it, listed in order of precedence with earlier rules overriding later ones:

1. **Certificate key-revocation overrides retention.** A CA MUST NOT accept a hashed URI that was successfully passed initial validation BEFORE a revocation using the certificate key as outlined in ({{!RFC8555}}, Section 7.6) using the certificate keypair. The CA MAY limit this operation to certificates issued using DNS-PERSIST-01, if the CA has this information stored. After such a revocation is made, the CA must prune any previously validated hashed URIs from the account.
2. **General retention.** Subject to the rules above, a CA SHOULD retain the hashed URI of every passed DNS-PERSIST-01 validation passed on a particular account, so that a record continues to validate even after key rotation. A CA MAY bound acceptance of a rotated-out key with a `keyRotationPeriod` (see {{key-rotation-period}}); once any applicable `keyRotationPeriod` has elapsed or a certificate was revoked using the certificate keypair as outlined in 1, the CA MAY prune that hashed UEI.

An CA MUST recheck that a stored hashed URI still is present in DNS - The stored hashed URIs only serve as comparision strings to allow an account with a rotated out key to still validate using a previously published DNS record - AS LONG as that DNS record have previously passed validation at least once.

## Account Key Rotation Period {#key-rotation-period}

A CA MAY advertise a `keyRotationPeriod` in the `meta` object of its directory ({{!RFC8555}}, Section 7.1.1). When present, its value MUST be a non-negative integer number of seconds. It bounds how long, after a public key ceases to be the current key of an ACME account (for example, because the account rotated to a new key per {{!RFC8555}}, Section 7.3.5), the CA continues to accept old hashes that was previously validated with that key.

After the period elapses, the CA MAY cease accepting previous hashes that was previously accepted, if these hashes does no longer compute using the current account details, subject to the precedence rules in {{verification-procedure}}. A CA that does not advertise a `keyRotationPeriod` retains and accepts prior hashed URIs per the general retention guidance in {{verification-procedure}}.

The `keyRotationPeriod` gives domain owners an upper bound on how long a previously accepted `_validation-persist` records computed with a since-rotated key remains usable, and gives CAs a defined point at which prior-key hashes may be pruned. Clients SHOULD republish `_validation-persist` records using the account's current key after a key rotation, and SHOULD do so before the CA's `keyRotationPeriod` elapses, so that validation does not lapse.

## Multiple Issuer Support {#multiple-issuer-support}

A domain MAY authorize multiple Certificate Authorities (CAs) by provisioning a separate `_validation-persist` TXT record for each issuer. This allows domain owners to maintain relationships with multiple CAs simultaneously, enhancing flexibility and resilience.

### Coexistence of Records

When multiple TXT records are present at the same DNS label (e.g., `_validation-persist.example.com`), each record functions as an independent authorization for the specified issuer. This follows a similar pattern to CAA records {{!RFC8659}}, where multiple records at the same label are permissible.

### CA Verification Process

When a CA performs validation for a domain with multiple `_validation-persist` TXT records, it MUST follow these steps:

1.  **Query DNS**: Retrieve all TXT records from the Validation Domain Name.
2.  **Filter Records**: Iterate through the returned records to find those where the `issuer-domain-name` value matches one of the Issuer Domain Names the CA is configured to use for this validation. The CA MUST ignore all other records.
3.  **Validate Record**: For each matching record, the CA proceeds to validate it according to the requirements in this specification, including verifying the `accounturi` and `persistUntil` parameters. If any matching record satisfies all requirements, the validation succeeds.
4.  **Handle No Match**: If no record with a matching `issuer-domain-name` is found, or if no matching record satisfies all validation requirements, the validation attempt MUST fail.

### Security and Management Considerations

When authorizing multiple issuers, domain owners MUST consider the following:

**Auditing**
:   Regularly audit DNS records to ensure that only intended CAs remain authorized. Remove records for CAs that are no longer in use.

**Independent Security**
:   Each authorized CA operates independently. The compromise of one CA's systems does not directly affect the security of other authorized CAs.

**Weakest Link**
:   The domain's overall security posture is influenced by the security practices of all authorized CAs. Domain owners should consider the practices of each CA they authorize.

**Authorization Removal**
:   To de-authorize a CA, the corresponding TXT record MUST be deleted from the DNS zone.

### Example: Authorizing Two CAs

This example demonstrates how a domain owner can authorize two different CAs, "ca1.example" and "ca2.example", to issue certificates for `example.org`.

**DNS Configuration:**

~~~dns
_validation-persist.example.org. 3600 IN TXT ("ca1.example;"
" accounturi=https://ca1.example/account-hash/"
"sha-256/wjZNvLpIgf7zrr9KQo7zCLaG2Ea67KS4o4cAwSepsls;"
" policy=wildcard")
_validation-persist.example.org. 3600 IN TXT ("ca2.example;"
" accounturi=https://ca2.example/account-hash/"
"sha-256/Mpngr2LZCN9gPArggXqEu4pKtYZvLL5IxMStzyWrrXc;"
" persistUntil=1767225600")
~~~
{: #ex-multiple-ca-auth title="Multiple CA Authorization Records"}

**Verification Flow for CA1:**

1.  CA1 queries for TXT records at `_validation-persist.example.org`.
2.  It receives both records.
3.  It filters for the record where `issuer-domain-name` is "ca1.example".
4.  It validates the request using this record, noting the `policy=wildcard` authorization.
5.  The second record for "ca2.example" is ignored.

**Verification Flow for CA2:**

1.  CA2 queries for TXT records at `_validation-persist.example.org`.
2.  It receives both records.
3.  It filters for the record where `issuer-domain-name` is "ca2.example".
4.  It validates the request using this record, noting the `persistUntil` constraint.
5.  The first record for "ca1.example" is ignored.

## Just-in-Time Validation {#just-in-time-validation}

When processing a new authorization request, a CA MAY perform an immediate DNS lookup for `_validation-persist` TXT records at the Validation Domain Name corresponding to the requested domain identifier.

If one or more such records exist, the CA MUST evaluate them according to the requirements specified in {{multiple-issuer-support}}. If at least one record meets all validation requirements, the CA MAY transition the authorization to the "valid" status without returning a "pending" challenge to the client. This mechanism is an optimization and does not alter the ACME state machine defined in {{!RFC8555}}. The server internally transitions the authorization from "pending" through "processing" to "valid" instantaneously. From the client's perspective, it receives a "valid" authorization object directly in response to its creation request.

If no DNS TXT record meets the validation requirements, or if the records are absent, the CA MUST proceed with the standard authorization flow by returning a "pending" authorization with an associated `dns-persist-01` challenge object.

CAs implementing Just-in-Time validation SHOULD rate-limit JIT DNS lookups per domain identifier, independent of the requesting account, to prevent amplification attacks where multiple accounts trigger excessive queries against a target domain. CAs SHOULD restrict JIT validation to accounts that have previously completed a successful `dns-persist-01` validation. CAs implementing Just-in-Time validation SHOULD provide account activity notifications or logging, as this path eliminates the challenge-response interaction that might otherwise provide a detection window for account compromise.

This mechanism enables efficient reuse of persistent validation records while maintaining the security properties of the validation method.

## Pre-Provisioning Records {#pre-provisioning-records}

Domain owners MAY provision `_validation-persist` TXT records before requesting any certificates, provided they can compute the `accounturi` value. When constructing records without a challenge object, the following values MUST be used:

- **accounturi**: The hashed URI, computed as specified in {{validation-record-format}}. A client knows its own ACME account URL (as returned in the `Location` header of the account creation response, {{!RFC8555}}, Section 7.3), the FQDN it is provisioning, and its current account key, so it can compute and publish the hashed URI itself from the CA's `accountHashPrefix` without a per-provisioning request to the CA. A client that wishes to reuse a single record across multiple domains MAY instead use the domain-omitted opt-out form (`domain_name` set to `*`, see {{validation-record-format}}).
- **issuer-domain-name**: A CA offering the dns-persist-01 challenge type advertises its Issuer Domain Names in the `issuerDomainNames` array of its directory `meta` object (see {{directory-issuer-domain-names}}). Clients pre-provisioning records SHOULD use one of these values; a value obtained elsewhere is not guaranteed to appear in a later challenge and can leave the record unusable.

For a pre-provisioned record, the client computes the hashed `accounturi` itself, outside the ACME challenge-response flow, from its own ACME account URL, its current account key, and the CA's `accountHashPrefix`. The client obtains the `accountHashPrefix` from the CA's directory ({{!RFC8555}}, Section 7.1.1) and SHOULD retrieve the directory over an authenticated channel, so that the record is placed under the intended CA's infrastructure. Because the hashed URI cryptographically binds the account key (see {{hashed-uri-security}}), a record computed with the client's genuine account key cannot be made to validate for a different account; this key binding is what lets a domain owner safely pre-provision without the challenge-response exchange.

Organizations pre-provisioning records SHOULD maintain an inventory of `_validation-persist` records and the ACME accounts they reference. Records MAY include a `persistUntil` parameter to bound their effective lifetime (see {{persist-until-parameter-considerations}}). Domain owners SHOULD audit `_validation-persist` records after any DNS infrastructure security incident, as pre-provisioned records persist beyond the window of compromise.

CAs implementing `dns-persist-01` SHOULD maintain a stable `accountHashPrefix` and stable account URLs for the lifetime of an account, and SHOULD document their stability guarantees, since the hashed URI is computed over the account URL and served under the prefix. If a CA must change its `accountHashPrefix` or account-URL structure, it SHOULD provide a transition period during which hashed URIs under both the old and new forms are accepted for validation.

# Wildcard and Subdomain Certificate Validation {#wildcard-certificate-validation}

This validation method supports validation for wildcard certificates (e.g., *.example.com) and specific subdomains through the use of the `policy=wildcard` parameter.

## Scope of `policy=wildcard`

When a DNS TXT record includes the `policy=wildcard` parameter value, it authorizes certificate issuance for:

1. **The validated FQDN itself** - The base domain for which the TXT record exists (e.g., `example.com`)
2. **Wildcard certificates** - Wildcard certificates for the validated FQDN or any of its subdomains (e.g., `*.example.com`, `*.dept.example.com`)
3. **Specific subdomains** - Any specific subdomain of the validated FQDN (e.g., `www.example.com`, `app.example.com`, `server.dept.example.com`)

For example, a TXT record at `_validation-persist.example.com` containing `policy=wildcard` can validate certificates for `example.com`, `*.example.com`, `www.example.com`, and any other subdomain of `example.com`.

If the `policy` parameter is absent, or if its value is anything other than `wildcard`, the validation applies only to the specific FQDN being validated. CAs MUST NOT consider such validation sufficient for wildcard certificates or subdomains.

# Subdomain Certificate Validation {#subdomain-certificate-validation}

When the `policy=wildcard` parameter is present (as described in {{wildcard-certificate-validation}}), CAs MAY issue certificates for subdomains of the validated FQDN. This section describes the implementation details for subdomain validation.

## Determining Permitted Subdomains

To determine which subdomains are permitted, the FQDN for which the persistent TXT record exists (referred to as the "validated FQDN") MUST be a proper suffix of the FQDN for which a certificate is requested (referred to as the "requested FQDN"). For wildcard certificate requests, the proper suffix check applies to the base domain name after removing the wildcard prefix (`*.`), consistent with {{!RFC8555}}, Section 7.1.3. The base-level wildcard (e.g., `*.example.com` where the validated FQDN is `example.com`) is authorized directly by {{wildcard-certificate-validation}} and is not subject to this proper suffix requirement.

For example, if `dept.example.com` is the validated FQDN, a certificate for `server.dept.example.com` is permitted because `dept.example.com` is its suffix. A certificate for `*.server.dept.example.com` is also permitted: after removing the wildcard prefix, `server.dept.example.com` has `dept.example.com` as a proper suffix.

## Implementation Requirements

- The persistent DNS TXT record MUST include `policy=wildcard` for subdomain validation to be permitted.
- CAs MUST verify that the validated FQDN is a proper suffix of the requested FQDN. For wildcard requests, this check applies to the base domain after removing the `*.` prefix. The base-level wildcard is exempt per {{wildcard-certificate-validation}}.
- If the `policy` parameter is absent or has any value other than `wildcard`, subdomain validation MUST NOT be permitted.

See {{subdomain-validation-risks}} for important security implications of enabling subdomain validation.

## Example: Subdomain Validation

For a persistent TXT record provisioned at `_validation-persist.example.com` with `policy=wildcard`:

- Permitted: `example.com`, `www.example.com`, `app.example.com`, `server.dept.example.com`, `*.example.com`, `*.dept.example.com`
- Not permitted without additional validation: `otherexample.com`, `example.net`

# Security Considerations {#security-considerations}

The requirement for CAs to ignore unknown parameter tags means that future extensions must be carefully designed to ensure that being ignored does not create security vulnerabilities. Extensions that require strict enforcement should use alternative mechanisms, such as separate record types or explicit version negotiation.

## Persistent Record Risks {#persistent-record-risks}

The persistence of validation records creates extended windows of vulnerability compared to traditional ACME challenge methods. If an attacker gains control of a DNS zone containing persistent validation records, they can potentially obtain certificates for the validated domains until the validation records are removed or modified.

Clients SHOULD protect validation records through appropriate DNS security measures, including:

- Using DNS providers with strong authentication and access controls
- Implementing DNS Security Extensions (DNSSEC) where possible
- Monitoring DNS zones for unauthorized changes
- Regularly reviewing and rotating validation records

## Account Binding Security {#account-binding-security}

The `accounturi` parameter provides strong binding between domain validation and the ACME account identified by the hashed URI. However, this binding depends on the security of the account itself.

The security of this method is fundamentally bound to the security of the ACME account's private key. Because the `accounturi` is a hashed URI computed in part over the account key (see {{hashed-uri-security}}), if this key is compromised, an attacker can immediately use any pre-existing `dns-persist-01` authorizations associated with that account to issue certificates, without needing any further access to the domain's DNS infrastructure. This elevates the importance of secure key management for ACME clients far above that required for transient challenge methods, as the window of opportunity for an attacker is tied to the lifetime of the persistent authorization, not a momentary challenge.

CAs SHOULD implement robust account security measures, including:

- Strong authentication requirements for ACME accounts
- Account activity monitoring and anomaly detection
- Rapid account revocation capabilities
- Regular account security reviews
- Account key rotation policies and procedures

Clients SHOULD protect their ACME account keys with the same level of security as they would protect private keys for high-value certificates.

### Account Key Rotation {#account-key-rotation}

The ACME account URL is the stable identifier for the account and persists across key rotations ({{!RFC8555}}, Section 7.3). The `accounturi` parameter, however, is a hashed URI computed in part over the account's current key ({{validation-record-format}}). When a client rotates its account key following the procedures defined in {{!RFC8555}}, Section 7.3.5, the hashed value in an existing DNS TXT record changes because `key` is one of its inputs, even though the account URL has not changed. An existing record continues to validate only for as long as the CA stores that hash, under the retention rules in {{verification-procedure}} and any `keyRotationPeriod` advertised per {{key-rotation-period}}; it is not a permanent exemption from republication. Clients SHOULD republish `_validation-persist` records with the hashed URI recomputed under the account's new key promptly after a rotation, as also noted in {{key-rotation-period}}; a client that does not do so risks a validation failure once the CA stops accepting the prior hash.

### Account URI Privacy {#account-uri-privacy}

Because `_validation-persist` TXT records are publicly queryable and long-lived, the `accounturi` value is visible to any party that queries DNS. With a cleartext account identifier, the same value appearing in records for multiple domains would let third parties infer that those domains share the same ACME account and likely share infrastructure; this correlation risk is noted in {{!RFC8657}}, Section 5.9. The hashed URI defined in this document mitigates that risk by binding `domain_name` into the digest, so a given account produces a different `accounturi` value for each domain (see {{hashed-uri-security}}).

Domain owners who require stronger privacy can additionally use separate ACME accounts for domains that should not be correlated. The domain-omitted opt-out form (`domain_name` set to `*`, see {{validation-record-format}}) deliberately forgoes the per-domain mitigation: the same value then appears in every record that uses it, so a bulk observer can link those domains. A domain owner who uses the opt-out to reuse one record across domains accepts that linkage.

### Hashed Account URIs {#hashed-uri-security}

The hashed URI is the sole `accounturi` form ({{validation-record-format}}). It serves two purposes: it cryptographically binds the requesting account to the domain so that a record cannot be transferred to authorize a different account, preserving the non-transferability property of {{!RFC8555}}; and it reduces the correlation exposure described in {{account-uri-privacy}}. It does not hide the account from the CA, which necessarily learns the account URL when it recomputes and validates the record.

**Source of the authorization binding.** The account holder commits to the account URL by publishing a digest computed over `account_URL`. Because the hash is collision resistant, this commitment is comparable to publishing the account URL directly, while the digest also incorporates the account key and the domain, as described below.

**Non-transferability and substitution resistance.** A raw account URL is only a URL commitment: its integrity is only as good as the channel that delivered the URL to the domain owner, and if that channel is poisoned the record binds to an account the owner did not intend, so RFC 8555 non-transferability is not preserved. The hashed URI closes this gap by additionally binding the account key. A CA recomputes the digest over the requesting account's own key, so a record validates for an account only if it was computed with a key that account controls. An attacker therefore cannot make an honest CA accept, for the attacker's account, a digest the victim published, because the attacker does not hold the victim's private key; and an attacker who substitutes a different account URL still cannot produce a validating record without the corresponding key. This substitution resistance is the reason this document defines only the hashed URI and does not permit a raw account URL.

**Role of the account key (privacy).** ACME account URLs are frequently low in entropy (e.g., a short, sequential account identifier of only a handful of digits). An adversary who knows a CA's account-URL structure could enumerate all plausible account URLs, hash each against a target domain, and recover the account URL behind a published digest, defeating the privacy goal. The account key's JWK Thumbprint is included as a high-entropy input that raises the cost of this enumeration.

**HTTP-01 key exposure.** The key authorization for the `http-01` challenge ({{!RFC8555}}, Section 8.3) is transmitted over cleartext HTTP and contains the same unpadded base64url SHA-256 JWK Thumbprint used as the `key` input above. An on-path observer of an `http-01` validation for the account therefore learns the thumbprint, leaving only the account URL unknown when evaluating a published hashed URI. The observer must still enumerate candidate account URLs and recompute the digest, so the remaining protection for that observer is limited to the entropy of the account URL.

**Binding to the record's own domain.** The digest binds `domain_name`, which is the FQDN at which the `_validation-persist` record is provisioned, not the FQDN of the certificate ultimately requested. This has two effects. First, the same account produces a different `accounturi` value in every domain's record, so a bulk observer cannot correlate domains by matching identical strings, as it could with a shared cleartext account URL. Second, because each record is bound to its own domain, there is no ambiguity under wildcard or CNAME'd provisioning about which name a given record authorizes, and a record cannot be lifted to a different domain. A domain owner who deliberately wants one record to authorize many names uses the domain-omitted opt-out (`domain_name` set to `*`), accepting the correlation trade-off described in {{account-uri-privacy}}.

**Digest construction.** The digest is compared for exact equality against a value the CA recomputes over a fixed field structure ({{validation-record-format}}), so there is no attacker-controlled trailing input, and length-extension of the digest yields nothing an attacker can use. The `length_of_domain` prefix and the fixed 43-octet `key` make every field boundary unambiguous regardless of which octets appear in `domain_name` or `account_URL`, so no field can be shifted into another to construct a colliding preimage. The `key` uses the unpadded base64url thumbprint representation already present as the second component of an ACME key authorization ({{!RFC8555}}, Section 8.1), avoiding an unnecessary decode step.

**Uniqueness of the binding.** What {{!RFC8657}}, Section 5.4 requires of an `accounturi` is reverse-direction uniqueness: a single `accounturi` value must not identify two different accounts. This holds because `account_URL` is one of the hash inputs and the hash is collision resistant, so each hashed URI uniquely identifies one account. The forward direction is intentionally one-to-many — one account yields a different value per domain — which is the privacy property above.

**Acceptance of prior keys.** A CA accepts digests that was previously accepted so that records provisioned before a rotation survive without republication ({{!RFC8555}}, Section 7.3.5), subject to the certificate key-base revocation, and key-rotation-period rules in {{verification-procedure}} and {{key-rotation-period}}. Such a record was committed to the account's URL when it was provisioned, so even if the key in the record is one that was later rotated out, the domain owner's intent to authorize that account is preserved. Certificate key-pair revocation prunes the previously accepted hashed URI list of an account, which is the emergency stop for a suspected key compromise.

**Limits of the privacy protection.** This construction is a best-effort obfuscation, not a cryptographically strong privacy mechanism. It resists bulk correlation and casual enumeration of account URLs, but a determined adversary with a-priori knowledge of candidate account URLs and their account keys can recover the true account URL. Domain owners who require stronger unlinkability SHOULD use separate ACME accounts for domains that must not be correlated.

## Subdomain Validation Risks {#subdomain-validation-risks}

Enabling subdomain validation via `policy=wildcard` creates significant security implications. Organizations using this feature SHOULD carefully control subdomain delegation and monitor for unauthorized subdomains. This policy value serves as the explicit mechanism for domain owners to opt-in to broader validation scopes.

The ability to issue certificates for subdomains of validated FQDNs creates significant security risks, particularly in environments with subdomain delegation or where subdomains may be controlled by different entities.

Potential risks include:

- Subdomain takeover attacks where abandoned subdomains are claimed by attackers
- Unauthorized certificate issuance for subdomains controlled by different organizations
- Confusion about which entity has authority over specific subdomains

Organizations considering the use of subdomain validation SHOULD:

- Maintain strict control over subdomain delegation
- Implement monitoring for subdomain creation and changes
- Consider limiting subdomain validation to specific, controlled scenarios
- Provide clear governance policies for subdomain certificate authority

## Cross-CA Validation Reuse {#cross-ca-validation-reuse}

The persistent nature of validation records raises concerns about potential reuse across different Certificate Authorities. While the `issuer-domain-name` parameter is designed to prevent such reuse, implementations MUST validate that the `issuer-domain-name` in the DNS record matches the CA's disclosed Issuer Domain Name.

## Record Tampering and Integrity {#record-tampering-and-integrity}

DNS records are generally not authenticated end-to-end, making them potentially vulnerable to tampering. CAs SHOULD implement additional integrity checks where possible and consider the overall security posture of the DNS infrastructure when relying on persistent validation records.

Additionally, CAs SHOULD protect their `ACME directory URL` with appropriate security measures. Using DNSSEC to protect the CA's `ACME directory URL` is RECOMMENDED. An attacker who compromises the DNS for a CA's `ACME directory URL` could disrupt validation or potentially impersonate the CA in certain scenarios. While this is a systemic DNS security risk that extends beyond this specification, it is amplified by any mechanism that relies on DNS for identity.

## Issuer Domain Name Normalization and Limits

The `issuerDomainNames` field requires domain names to be provided in a normalized form (lowercase A-labels, no trailing dot) to prevent errors and security issues arising from case-sensitivity differences or Unicode homograph attacks. By requiring a canonical representation, servers and clients can perform simple byte-for-byte comparisons, ensuring interoperability and deterministic validation. The order of names in the array has no significance.

The server-side limit on the number of issuer domain names provided in a single challenge (e.g., 10) helps mitigate denial-of-service vectors where a client might be forced to perform an excessive number of DNS queries or a server might be burdened by validating against a large set of domains.

## CAA Interaction {#caa-interaction}

The `dns-persist-01` validation method is a valid `validationmethods` parameter value for CAA records per {{!RFC8657}}. Under {{!RFC8657}}, Section 3.1, a CAA `accounturi` for an ACME account is the URI of the ACME account object (the account URL); when the parameter is present, the CA matches that URI as part of CAA processing. The `dns-persist-01` `accounturi` is instead the hashed URI defined in {{validation-record-format}}, which the CA matches by recomputing it while validating the `_validation-persist` record. These parameters occur in different DNS records and need not have the same value. They are independent reinforcing checks: both can restrict issuance, and the CA evaluates them separately.

## DNS Security Measures {#dns-security-measures}

To enhance the security and integrity of the validation process, CAs and clients should consider implementing advanced DNS security measures.

### DNSSEC

DNS Security Extensions (DNSSEC) {{?RFC4033}} provide cryptographic authentication of DNS data, ensuring that the validation records retrieved by a CA are authentic and have not been tampered with. CAs SHOULD use a DNSSEC-validating resolver when querying `dns-persist-01` TXT records. Without one, a CA will silently accept forged responses in DNSSEC-signed zones. If a CA performs DNSSEC validation, it MUST treat validation failure (e.g., expired signatures, broken chain of trust) as a challenge failure and MUST NOT use the record for domain validation. This requirement is stricter than the general DNSSEC guidance in {{!RFC8555}} because `dns-persist-01` records are long-lived and their compromise would persist for the record's lifetime.

### Multi-Perspective Validation

Multi-Perspective Issuance Corroboration (MPIC) is a technique to validate domain control from multiple network vantage points. This is a critical defense against localized network attacks, such as BGP hijacking and DNS spoofing, which could otherwise lead to certificate mis-issuance.

For CAs subject to requirements like the CA/Browser Forum Baseline Requirements, MPIC is essential for robust domain validation. However, for private PKI systems where the network topology is well-known and such localized attacks are not part of the threat model, operators might reasonably judge MPIC unnecessary.

## Validation Data Reuse {#validation-data-reuse}

This validation method is explicitly designed for persistence and reuse. The period for which a CA may rely on validation data is its `Validation Data Reuse Period` (as defined in {{conventions-and-definitions}}), bounded by the CA's own policy, applicable root program requirements, and any `persistUntil` constraint on the record ({{validation-record-format}}). The DNS TTL of the `_validation-persist` record governs caching at the DNS layer only; it is not a validation data reuse limit, and a CA MUST NOT derive the effective validation data reuse period from the record's observed TTL.

CAs MAY reuse validation data obtained through this method for the duration of their Validation Data Reuse Period. CAs MUST also respect any `persistUntil` constraint as specified in {{validation-record-format}}. For a validation attempt that queries DNS, removing or changing the TXT record takes effect after resolver caches expire, as described in {{revocation-and-invalidation}}. Record removal does not invalidate previously obtained validation data before its allowed reuse period expires.

## persistUntil Parameter Considerations {#persist-until-parameter-considerations}

The `persistUntil` parameter provides domain owners with direct control over the validity period of their validation records. CAs and clients should be aware of the following considerations:

- Domain owners should set expiration dates for validation records that balance security and operational needs. To avoid unexpected validation failures during certificate renewal, domain owners are advised to:
   - Align `persistUntil` values with certificate lifetimes or planned maintenance intervals
   - Monitor or set reminders for `persistUntil` expirations
   - Document `persistUntil` practices in certificate management procedures
   - Automate updates to validation records with new `persistUntil` values during certificate renewal workflows
- CAs MUST parse and apply the `persistUntil` timestamp as specified in {{validation-record-format}}.

## Revocation and Invalidation of Persistent Authorizations {#revocation-and-invalidation}

The persistent nature of `dns-persist-01` authorizations means that a valid DNS TXT record can grant control for an extended period, potentially even if the domain owner's intent changes or if the associated account key is compromised. Therefore, explicit mechanisms for revoking or invalidating these persistent authorizations are critical.

There are two primary revocation and invalidation actions:

1. **Remove DNS record:** Removing the corresponding DNS TXT record from the Validation Domain Name is the standard method for an Applicant to invalidate a `dns-persist-01` authorization. After the record is removed and resolver caches have expired, new validation attempts for the domain will fail. This action is available to the domain owner and does not require CA involvement.
2. **Deactivate ACME account:** For situations requiring immediate revocation of issuance capability (e.g., suspected account key compromise), the account bound by the record can be deactivated as specified in {{!RFC8555}}, Section 7.3.6. Note that this does not prevent issuance if a ACME account that previously have validated a hashed URI, becomes compromised and the attacker keyChanges into a key the attacker controls. For this situation, either 1, and/or 3, must be carried out to cease acceptance of the comrpomised key.
3. **Revoke a certificate using its keypair:** For situations where the account holder has lost control of a compromised ACME account, the account owner can request revocation of a certificate issued using DNS-PERSIST-01, using the account, using the certificate keypair as specified in as specified in {{!RFC8555}}, Section 7.6. This clears all previously accepted hashed URIs on the compromised account. After that, the account must perform a ACME account deactivation using the compromised keypair, to prevent the attacker from keyChanging back to the compromised account key to aquire a fresh validation using the old record.

The following table summarizes the applicability and timing of these actions:

| Action | Effect |
|---|---|
| Remove DNS record | New validations fail after resolver cache expiry; domain-owner action, no CA involvement |
| Deactivate ACME account | New validations fail immediately |
| Revoke a certificate using keypair | Deletes all previously accepted hashed URIs for the account that issued the certificate - prevents a compromised ACME account for which the owner lost access to, from continuing issuing using legacy records. This action is immediate. |

Independently of these actions, a CA MAY bound how long a record computed with a previosuly accepted account URI remains acceptable by advertising a `keyRotationPeriod` (see {{key-rotation-period}}). This bounds acceptance over time but is a routine lifecycle limit, not an immediate revocation mechanism.

ACME Clients SHOULD provide clear mechanisms for users to:

- Remove the `_validation-persist` DNS TXT record.
- Monitor the presence and content of their `_validation-persist` records to ensure they accurately reflect desired authorization.

Certificate Authorities (CAs) implementing this method MUST:

- During a validation attempt, fail the validation if the corresponding DNS TXT record is no longer present or if its content does not meet the requirements of this specification (e.g., incorrect `issuer-domain-name`, missing `accounturi`, altered `policy`).

- Respect the `persistUntil` constraint as specified in {{validation-record-format}}, rejecting new validation attempts after the specified timestamp even if the record remains present.

- Ensure their internal systems are capable of efficiently handling the validation failure when DNS records are removed or become invalid.

While this method provides a persistent signal of control, the fundamental ACME authorization object (as defined in {{!RFC8555}}) remains subject to its own lifecycle, including expiration. A persistent DNS record allows for repeated authorizations, but each authorization object issued by the CA will have a defined validity period, after which it expires unless renewed.

# IANA Considerations {#iana-considerations}

## ACME Validation Methods Registry {#acme-validation-methods-registry}

IANA is requested to register the following entry in the "ACME Validation Methods" registry:

- **Label**: dns-persist-01
- **Identifier Type**: dns
- **ACME**: Y
- **Reference**: This document

## Underscored and Globally Scoped DNS Node Names Registry {#dns-node-names-registry}

IANA is requested to register the following entry in the "Underscored and Globally Scoped DNS Node Names" registry defined in {{!RFC8552}}:

- **RR Type**: TXT
- **_NODE NAME**: _validation-persist
- **Reference**: This document

## ACME Directory Metadata Fields {#acme-directory-metadata-field}

IANA is requested to register the following entries in the "Fields in the 'meta' Object within a Directory Object" registry defined in {{!RFC8555}}, Section 9.7.6:

- **Field Name**: accountHashPrefix
- **Field Type**: string
- **Reference**: This document

- **Field Name**: keyRotationPeriod
- **Field Type**: integer
- **Reference**: This document

- **Field Name**: issuerDomainNames
- **Field Type**: array of string
- **Reference**: This document

# Implementation Considerations {#implementation-considerations}

When designing future extensions to this specification, new parameters SHOULD be designed to degrade gracefully when ignored by CAs that do not recognize them. Parameters that fundamentally change the security properties of the validation SHOULD NOT be introduced without a version negotiation mechanism.

## DNS Record Size Considerations

The RDATA of the TXT record, which contains the `issue-value`, may become large, particularly if the `accounturi` is long. While the total size of a TXT record's RDATA can be up to 65,535 octets, it must be formatted as a sequence of one or more character-strings, where each string is limited to 255 octets in length.

**CA Implementation Guidelines:**

- CAs SHOULD endeavor to keep the `accounturi` values they generate reasonably concise to minimize the final record size.

**Client Implementation Guidelines:**

- Clients MUST properly handle the creation of TXT records where the RDATA exceeds 255 octets. As specified in {{!RFC1035}}, Section 3.3.14, clients MUST split the RDATA into multiple, concatenated, quote-enclosed strings, each no more than 255 octets. For example:

~~~ dns
_validation-persist.example.com. IN TXT ("first-255-bytes..."
" ...remaining-bytes")
~~~
{: #ex-long-txt-record title="Multi-String TXT Record Format"}

Failure to correctly format long RDATA values may result in validation failures.

## Domain Name Normalization Algorithm {#normalization-algorithm}

This section provides an algorithm for domain name normalization to promote interoperability. Both clients and servers SHOULD follow a consistent normalization process to ensure that domain names are handled uniformly.

The recommended normalization process consists of the following four steps, applied in order:

1.  **Case-folding**: Apply Unicode-aware, locale-independent case-folding to the entire domain name string to convert it to lowercase.
2.  **Unicode Normalization**: Normalize the string to Unicode Normalization Form C (NFC).
3.  **Punycode Conversion**: Convert each label of the domain name to its A-label (Punycode) representation as specified in {{!RFC5890}}.
4.  **Trailing Dot Removal**: Remove any trailing dot from the final string.

For example, a domain name like `EXAMPLE.com.` is normalized as follows:

1. After case-folding: `example.com.`
2. After NFC normalization: `example.com.`
3. After Punycode conversion: `example.com.`
4. After removing trailing dot: `example.com`

An internationalized domain name like `üÑICODE-example.com.` is normalized as follows:

1. After case-folding: `ünicode-example.com.`
2. After NFC normalization: `ünicode-example.com.`
3. After Punycode conversion: `xn--nicode-example-9jb.com.`
4. After removing trailing dot: `xn--nicode-example-9jb.com`

## CA Implementation Guidelines {#ca-implementation-guidelines}

Certificate Authorities implementing this validation method should consider:

- Establishing clear policies for Issuer Domain Name disclosure in Certificate Policies and Certification Practice Statements
- Creating account security monitoring and incident response procedures
- Providing clear documentation for clients on proper record construction

### Error Handling

When implementing the "dns-persist-01" validation method, Certificate Authorities SHOULD return appropriate ACME error codes to provide clear feedback on validation failures. Specifically:

- CAs SHOULD return a `malformed` error (as defined in {{!RFC8555}}) when the TXT record has invalid syntax, such as duplicate parameters, invalid timestamp format in the `persistUntil` parameter, missing mandatory `accounturi` parameter, an `accounturi` that is not a well-formed hashed URI (for example, one not placed under the CA's `accountHashPrefix`, one using an algorithm identifier the CA does not accept, or one whose hash value is not valid base64url of the expected length), or other syntactic violations of the record format specified in this document.

- CAs SHOULD return an `unauthorized` error (as defined in {{!RFC8555}}) when validation fails due to authorization issues, including:
   - The `accounturi` hashed URI in the DNS TXT record does not, when recomputed by the CA over the requesting account's retained keys (see {{verification-procedure}}), match any value the CA accepts for that account
   - The `persistUntil` timestamp has expired, indicating that the validation record is no longer considered valid for new validation attempts
   - The `issuer-domain-name` in the DNS TXT record does not match any of the values provided in the `issuerDomainNames` array of the challenge object

Note that these error codes apply to validation attempts on specific challenges. In the case of Just-in-Time Validation (see {{just-in-time-validation}}), when a CA finds a pre-existing DNS TXT record that does not meet validation requirements, the CA proceeds with the standard authorization flow by issuing a new `pending` challenge rather than returning an error.

These error codes help ACME clients distinguish between different types of validation failures and take appropriate corrective actions.

## Client Implementation Guidelines {#client-implementation-guidelines}

ACME clients implementing this validation method should consider:

- Implementing secure DNS record management practices
- Providing clear user interfaces for managing persistent validation records
- Implementing validation record monitoring and alerting
- Designing appropriate error handling for validation failures
- Considering the security implications of persistent records in their threat models

Clients that manage or provision `_validation-persist` records SHOULD inspect their own local provisioning state (rather than relying on a DNS lookup) before initiating a new order, and SHOULD warn the operator if a record's `persistUntil` value has already expired or is likely to expire before validation completes. A client that skips this check risks initiating a validation attempt that the CA will reject with an `unauthorized` error (see {{ca-implementation-guidelines}}) instead of reporting an actionable, locally diagnosed cause.

## DNS Provider Considerations {#dns-provider-considerations}

DNS providers supporting this validation method should consider:

- Implementing appropriate access controls for validation record management
- Providing audit logging for validation record changes
- Considering dedicated interfaces or APIs for ACME validation record management

# Examples {#examples}

## Basic Validation Example (FQDN Only) {#basic-validation-example}

For validation of "example.com" by a CA using "authority.example" as its Issuer Domain Name, where the validation should only apply to "example.com":

1.  CA provides challenge object with a list of valid Issuer Domain Names:

    ~~~json
    {
      "type": "dns-persist-01",
      "url": "https://ca.example/acme/authz/1234/0",
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

With a CA `accountHashPrefix` of `https://ca.example/account-hash/` and the `sha-256` algorithm identifier, the hashed URI is `https://ca.example/account-hash/sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw`. Using issuer domain name "authority.example", the client provisions:

~~~ dns
_validation-persist.example.com. IN TXT ("authority.example;"
" accounturi=https://ca.example/account-hash/"
"sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw")
~~~
{: #ex-hashed-uri title="Hashed-URI Validation Record"}

To instead opt out of the domain-correlation mitigation ({{validation-record-format}}), the client sets `domain_name` to the single octet `*` (so `length_of_domain` is `0x01`), keeping the same `key` and `account_URL`. Computing `SHA-256(0x01 || "*" || key || account_URL)` yields:

~~~
NpDnSwUthQK8zCgFdefYxAdAVPnygMLbs9US6oO-5ug
~~~

The resulting hashed URI, `https://ca.example/account-hash/sha-256/NpDnSwUthQK8zCgFdefYxAdAVPnygMLbs9US6oO-5ug`, does not depend on the domain and MAY be reused across domains.


## Wildcard Validation Example {#wildcard-validation-example}

For validation of "*.example.com" (which also validates "example.com" and specific subdomains like "www.example.com") by a CA using "authority.example" as its Issuer Domain Name:

1.  The CA provides a challenge object similar to the basic example, containing an `issuerDomainNames` array.

2.  Client chooses one of the provided Issuer Domain Names (e.g., "authority.example") and provisions a DNS TXT record at the base domain's Validation Domain Name, including `policy=wildcard`:

    ~~~ dns
    _validation-persist.example.com. IN TXT ("authority.example;"
    " accounturi=https://ca.example/account-hash/"
    "sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw;"
    " policy=wildcard")
    ~~~
    {: #ex-wildcard-validation title="Wildcard Policy Validation Record"}

3.  CA validates the record through DNS queries. This validation authorizes certificates for "example.com", "*.example.com", and specific subdomains like "www.example.com".

## Validation Example with persistUntil

For validation of "example.com" with an explicit expiration date:

1.  The CA provides a challenge object similar to the basic example, containing an `issuerDomainNames` array.

2.  Client chooses one of the provided Issuer Domain Names (e.g., "authority.example") and provisions a DNS TXT record including `persistUntil`:

    ~~~ dns
    _validation-persist.example.com. IN TXT ("authority.example;"
    " accounturi=https://ca.example/account-hash/"
    "sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw;"
    " persistUntil=1721952000")
    ~~~
    {: #ex-persist-until title="Validation Record with Expiration Time"}

3.  CA validates the record. This validation is sufficient only for "example.com" and will not be considered valid after the specified timestamp (2024-07-26T00:00:00Z).

## Wildcard Validation Example with persistUntil

For validation of "*.example.com" with an explicit expiration date:

1.  The CA provides a challenge object similar to the basic example, containing an `issuerDomainNames` array.

2.  Client chooses one of the provided Issuer Domain Names (e.g., "authority.example") and provisions a DNS TXT record including `policy=wildcard` and `persistUntil`:

    ~~~ dns
    _validation-persist.example.com. IN TXT ("authority.example;"
    " accounturi=https://ca.example/account-hash/"
    "sha-256/5SQm7n6tPh2-PlLbCKGnViTXX5z19SCN4cPGQHSk-kw;"
    " policy=wildcard;"
    " persistUntil=1721952000")
    ~~~
    {: #ex-wildcard-persist-until title="Wildcard Validation Record with Expiration Time"}

3.  CA validates the record. This validation authorizes certificates for "example.com", "*.example.com", and specific subdomains, but will not be considered valid after the specified timestamp (2024-07-26T00:00:00Z).

--- back

# Acknowledgments
{:unnumbered}

The authors acknowledge prior community work that directly informed this specification:

- The CA/Browser Forum ballot proposals to enable persistent / static DNS Domain Control Validation signals in the Baseline Requirements {{cabf-br}}, in particular Ballot SC-082 ("Clarify CA Assisted DNS Validation under 3.2.2.4.7", authored by Michael Slaughter) and the active proposal SC-088 ("DNS TXT Record with Persistent Value DCV Method", also authored by Michael Slaughter). These efforts provided the policy framing and initial industry discussion motivating standardization of a reusable ACME DNS validation record.
- The formal and empirical security analysis of static / persistent DCV methods performed by Henry Birge-Lee ("Proof of static DCV security" presentation, the "Security of SC-082 Redux" paper {{birgelee-sc082-security}}, and related research), which helped clarify the threat model and informed the security considerations in this document.
- The Delegated DNS Domain Validation (DDDV) Threat Modeling Tiger Team discussions and document ("Validation SC - Delegated DNS Domain Validation (DDDV) Threat Model"), whose participants contributed to broad threat enumeration; notable contributors include Michael Slaughter (Amazon Trust Services), Corey Bonnell (DigiCert), Clint Wilson (Apple), and Martijn Katerbarg (Sectigo).

The authors also thank members of the ACME Working Group and CA/Browser Forum who provided early review, critique, and operational perspectives on persistent validation records.

Any errors or omissions are the responsibility of the authors.

# Change Log {#changelog}
{:unnumbered}

RFC Editor: please remove this section before publication.

## Since draft-ietf-acme-dns-persist-01
{:unnumbered}

- The `accounturi` is now a hashed URI that cryptographically binds the ACME account to the domain being validated. In -01 the `accounturi` was the ACME account URL, or another URI that uniquely and permanently identified a single account ({{!RFC8657}}, Section 5.4), with no cryptographic binding; that form is no longer accepted.
- Defined the hashed URI as `<accountHashPrefix><hash-alg>/<base64url hash value>`, advertised via the new `accountHashPrefix` directory metadata field and carrying an in-URI hash-algorithm identifier for algorithm agility (the registered `sha-256` Hash Name String is mandatory).
- Specified the digest as `H(length_of_domain || domain_name || key || account_URL)`, with a leading length octet delimiting the domain name.
- Added a mandatory domain-omitted opt-out (`domain_name` set to `*`) for correlation opt-out and cross-domain record reuse.
- Specified CA key retention as a data-retention obligation with ranked precedence (deactivation overrides certificate-backed retention overrides general retention); account deactivation is an immediate override.
- Added the `keyRotationPeriod` directory metadata field.
- Renamed the challenge object's plural field from `issuer-domain-names` to `issuerDomainNames` to align with ACME's camelCase convention ({{!RFC8555}}); the singular DNS `issuer-domain-name` parameter and `persistUntil` are unchanged (#56).
- Clarified that `key` in the digest is the 43-octet unpadded base64url encoding of the SHA-256 JWK Thumbprint {{!RFC7638}}, not the 32 raw digest octets, matching the encoding already used in ACME key authorizations ({{!RFC8555}}, Section 8.1).
- Required `sha-256` support in both clients and CAs for interoperability, moved `<hash-alg>` to the Hash Name String registry defined by {{!RFC6920}}, excluded truncated digests shorter than 256 bits, and removed the document-local slash-to-hyphen substitution rule.
- Clarified the relationship between the CAA `accounturi` and the `dns-persist-01` `accounturi` in the Introduction and {{caa-interaction}}, citing {{!RFC8657}}, Section 3.1.
- Added a privacy caveat noting that a cleartext `http-01` key authorization exposes the same account-key thumbprint used in the hashed URI, leaving account-URL entropy as the remaining protection.
- Added client-side guidance ({{client-implementation-guidelines}}) to check `persistUntil` against local provisioning state before initiating an order (#38).
- Removed DNS TTL as a validation data reuse limit; reuse remains subject to `persistUntil` and the CA's Validation Data Reuse Period. Renamed {{validation-data-reuse}} accordingly and removed the related TTL-specific implementation guidance (#42).
- Corrected the stale Account Key Rotation text: the account URL is stable, but the hashed `accounturi` changes with the account key, and continued acceptance of a prior key is governed by {{verification-procedure}}.
- Added the required directory `issuerDomainNames` metadata field and the D ⊆ C consistency rule with the challenge object's `issuerDomainNames`; the C ⊆ I rule against `caaIdentities` applies when the CA advertises `caaIdentities` (#60).
