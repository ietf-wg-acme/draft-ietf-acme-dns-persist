# IETF 125 Presentation

**Meeting:** IETF 125 ACME Working Group
**Location:** Shenzhen, China
**Dates:** March 15-21, 2026
**Draft:** draft-ietf-acme-dns-persist-01

## Files

- `slides.md` - Marp markdown source for presentation slides
- `speaker-notes.md` - Detailed speaker notes and timing outline
- `slides.pdf` - Generated PDF for IETF submission (committed to repo)

## Building

### Generate PDF

```bash
marp slides.md -o slides.pdf --pdf
```

### Preview in browser

```bash
marp -p slides.md
```

This opens an interactive preview with presenter mode support.

## Requirements

Install Marp CLI:

```bash
npm install -g @marp-team/marp-cli
```

Or via Homebrew:

```bash
brew install marp-cli
```

## Submission

Submit `slides.pdf` through the IETF Datatracker before the meeting deadline.
