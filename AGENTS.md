# Agent Instructions

This repository hosts the IETF Internet-Draft `draft-ietf-acme-dns-persist`,
authored in kramdown-rfc markdown and processed by the `i-d-template` toolchain
into xml2rfc XML, HTML, and RFC-format text.

## Scope

- The only normative artifact is `draft-ietf-acme-dns-persist.md`.
- `drafts/` holds published revisions; do not edit.
- `presentations/` holds meeting slide decks built by the pre-commit hook.
- Generated `.txt`, `.html`, and `.xml` outputs are build artifacts and must
  not be committed.

## Reviewing changes

`.github/copilot-instructions.md` is the authoritative guide for reviewing
protocol content. Read it before commenting on any PR that touches normative
text. The rules apply regardless of which agent is performing the review.

## Authoring changes

- Use kramdown-rfc syntax: `{{!RFCNNNN}}` for normative references,
  `{{?RFCNNNN}}` for informative, `{#anchor-name}` for section anchors, `~~~`
  fences for figures, kramdown definition lists (`:`) for term definitions.
  Full rules in `.github/copilot-instructions.md` §1.
- BCP 14 keywords (MUST, SHOULD, MAY) are normative only when capitalized.
  Every SHOULD requires a stated consequence for deviation.
- The draft terminology ("Validation Domain Name", "Issuer Domain Name",
  "Subscriber Account") is deliberately distinct from CA/Browser Forum
  Baseline Requirements terminology. Do not conflate.

## Build and verify

```sh
make          # build draft-ietf-acme-dns-persist.{txt,html}
make check    # run idnits validation on the generated xml
make lint     # run branch-name, docname, and whitespace checks
```

The first `make` clones `martinthomson/i-d-template` into `lib/` and creates
the `.xml` intermediate before producing the `.txt` and `.html` outputs. CI
runs the same targets via `.github/workflows/ghpages.yml`.

## Commits and pull requests

- Use Conventional Commits prefixes (`feat:`, `fix:`, `docs:`, `chore:`), as
  established in the existing history.
- Reference the relevant issue number in the PR body (`Fixes #NN`,
  `Closes #NN`).
- Include a short rationale. Protocol changes are reviewed by humans on the
  ACME working group mailing list; a clear PR description accelerates that.

## External actions

This is an IETF working group repository. Do not push branches, open PRs,
comment on issues, or send mail to `acme@ietf.org` without explicit approval
from the user. Reading the issue tracker, mailing list archive, and
Datatracker is always permitted.

## Open design questions

Several issues are typically unresolved at any given time and may change
normative text. Before proposing edits, retrieve the current list:

```sh
gh issue list --repo ietf-wg-acme/draft-ietf-acme-dns-persist \
  --state open --limit 50
gh pr list --repo ietf-wg-acme/draft-ietf-acme-dns-persist \
  --state open --limit 20
```

Read the titles and bodies to identify which issues touch normative text
(record format, verification procedure, account binding, revocation, security
considerations) as opposed to editorial or tooling concerns. For any PR that
modifies a section referenced by an open issue, either address the open issue
or note in the PR description why the change is orthogonal. Do not assume the
existing text is the final form in areas with active discussion.
