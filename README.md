# ACME Persistent DNS Challenge

This draft specifies `dns-persist-01`, a new ACME validation method proving domain control via persistent DNS TXT records. It suits IoT deployments, multi-tenant platforms, and batch certificate operations where real-time interaction is impractical.

## Links

* [Editor's Copy (HTML)](https://ietf-wg-acme.github.io/draft-ietf-acme-dns-persist/draft-ietf-acme-dns-persist.html)
* [Editor's Copy (txt)](https://ietf-wg-acme.github.io/draft-ietf-acme-dns-persist/draft-ietf-acme-dns-persist.txt)
* [Datatracker Page](https://datatracker.ietf.org/doc/draft-ietf-acme-dns-persist/)
* [Compare Editor's Copy to Latest](https://ietf-wg-acme.github.io/draft-ietf-acme-dns-persist/#go.draft-ietf-acme-dns-persist.diff)

## Abstract

The method verifies domain control through a persistent DNS TXT record containing CA and account identification. The record binds strongly to accounts through `accounturi` and expires optionally through `persistUntil`. Operators reuse the validation record across multiple certificate issuances. Its security properties align with CA/Browser Forum Baseline Requirements.

## Building the Draft

```sh
make              # Build .txt and .html outputs
make lint         # Check formatting
make fix-lint     # Fix formatting issues automatically
make idnits       # Run IETF submission checker
make diff         # Show changes since last submission
```

## GitHub Actions

**ghpages.yml** builds and publishes the editor's copy on every push.
**publish.yml** submits to the IETF datatracker when you push tags matching `draft-*`.
**archive.yml** archives GitHub issues and PRs on a schedule.

## Presentations

See [presentations/](presentations/) for meeting materials.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.
