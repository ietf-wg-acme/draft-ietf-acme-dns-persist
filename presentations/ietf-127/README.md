<!-- markdownlint-disable MD013 -->

# IETF 127 Presentation

Meeting: IETF 127 ACME Working Group

Location: San Francisco, USA (in-person and remote)

Slot: Wednesday, November 18, 2026, 09:00–11:00 PST (17:00–19:00 UTC; 10-minute presentation slot)

Draft: `draft-ietf-acme-dns-persist-03` (WGLC candidate; includes `#74`, `#75`, `#77`–`#87`, `#89`, and pre-WGLC review and protocol-clarification updates)

Status: Nine-slide Marp deck and speaker notes covering `-03` clarifications and normative guardrails (ASCII-only hash normalization, exact issuer matching, CNAME/pre-alias `R` binding, `accountHashPrefix` character verification, unsigned `persistUntil` bounds, account-URL non-reuse, directory `issuerDomainNames` stability), the four feature additions (`#84` `ancestorDomain` + RFC 9444 coexistence, `#85` `_ip-validation-persist` reverse-DNS IP validation, `#86` `persistExpired` error type, `#87` certificate-key attestation conjunction), the October 13 implementation feedback, earlier-draft non-interoperability, and the WGLC request.

## Files

- `slides.md`: Marp source
- `speaker-notes.md`: speaker notes, timing outline, `[IF SHORT]` scripts, and likely questions
- `slides.pdf`: generated PDF for IETF meeting materials

## Build

```sh
npx @marp-team/marp-cli slides.md -o slides.pdf --pdf
```

Preview in a browser with:

```sh
npx @marp-team/marp-cli -p slides.md
```

## Submission

Upload `slides.pdf` through the IETF 127 Meeting Materials tool. Rebuild the PDF after each slide edit.
