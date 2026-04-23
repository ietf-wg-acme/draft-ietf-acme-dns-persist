# ACME Persistent DNS Challenge

[![Build Status](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/actions/workflows/ghpages.yml/badge.svg)](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/actions/workflows/ghpages.yml)

This is the working area for the IETF [ACME Working Group](https://datatracker.ietf.org/wg/acme/documents/) Internet-Draft, "ACME Challenge for Persistent DNS TXT Record Validation".

The draft defines `dns-persist-01`, a new ACME challenge type that validates control of a domain via a persistent DNS TXT record at `_validation-persist.<fqdn>`. The record binds the domain to a specific Certification Authority and account, and may be reused for repeated issuance without renewed real-time interaction. This supports multi-tenant hosting, pre-validation, strict change-management environments, wildcard certificates, and IoT deployments.

**Status:** Working group document. Substantive protocol discussion occurs in the [issue tracker](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues) and on the ACME working group mailing list.

* [Editor's Copy (HTML)](https://ietf-wg-acme.github.io/draft-ietf-acme-dns-persist/draft-ietf-acme-dns-persist.html)
* [Editor's Copy (txt)](https://ietf-wg-acme.github.io/draft-ietf-acme-dns-persist/draft-ietf-acme-dns-persist.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-ietf-acme-dns-persist/)
* [Compare Editor's Copy to Latest](https://ietf-wg-acme.github.io/draft-ietf-acme-dns-persist/#go.draft-ietf-acme-dns-persist.diff)

## Related Specifications

* [RFC 8657](https://www.rfc-editor.org/rfc/rfc8657) — CAA `accounturi` and `validationmethods`, the basis for account binding in this draft.
* [RFC 8659](https://www.rfc-editor.org/rfc/rfc8659) — CAA record format; the `issue-value` syntax is reused for the validation record.
* [RFC 9444](https://www.rfc-editor.org/rfc/rfc9444) — ACME for Subdomains. Related to but distinct from the subdomain validation defined here.
* [CA/Browser Forum](https://cabforum.org/) — Baseline Requirements, including the [ongoing ballot](https://github.com/cabforum/servercert/pull/626) to enshrine DNS Persist for IP address validation via reverse zones (see [issue #32](https://github.com/ietf-wg-acme/draft-ietf-acme-dns-persist/issues/32)).

## Building the Draft

Formatted text and HTML versions of the draft can be built using `make`.

```sh
make
```

This requires that you have the necessary software installed. See [the instructions](https://github.com/martinthomson/i-d-template/blob/master/doc/SETUP.md).

## Contributing

See the [guidelines for contributions](CONTRIBUTING.md). Discussion of this work occurs on the [ACME WG mailing list](mailto:acme@ietf.org) ([archive](https://mailarchive.ietf.org/arch/browse/acme/)).

Agents and reviewers: [`AGENTS.md`](AGENTS.md) describes authoring conventions, the build loop, and commit style. [`.github/copilot-instructions.md`](.github/copilot-instructions.md) captures the project's protocol review rules and applies to all reviewers, human or automated.

## Presentations

Meeting materials are in [presentations/](presentations/).
