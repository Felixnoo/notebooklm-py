# Package Rename: notebooklm-client → notebooklm-py

**Date:** 2026-01-07
**Status:** Approved

## Decision

Rename the PyPI package from `notebooklm-client` to `notebooklm-py`.

## Goals

1. **Discoverability** - Easy to find on PyPI via "notebooklm" searches
2. **Differentiation** - Signal this is a community/unofficial client
3. **Branding** - Memorable, follows established convention
4. **Brevity** - Shorter than current name

## Naming Strategy

| Name Type | Before | After |
|-----------|--------|-------|
| PyPI package | `notebooklm-client` | `notebooklm-py` |
| Import name | `notebooklm` | `notebooklm` (unchanged) |
| CLI command | `notebooklm` | `notebooklm` (unchanged) |

## Rationale

- `-py` suffix is a common PyPI convention (`stripe-py`, `twilio-py`, `openai-py`)
- Shorter than `-client` (13 chars vs 17 chars)
- Signals Python-specific implementation without claiming "official" status
- Import and CLI remain intuitive (`import notebooklm`, `notebooklm login`)

## Trademark Considerations

- "NotebookLM" is a Google trademark
- Mitigations in place:
  - "Unofficial" prominently stated in description and README
  - No Google branding/logos used
  - Clear attribution that this is not affiliated with Google
- Risk level: Low (follows precedent of community clients like `python-telegram-bot`)

## Files to Update

1. `pyproject.toml` - Package name
2. `README.md` - Installation instructions, badges
3. `docs/getting-started.md` - Installation instructions
4. GitHub repository name (optional, separate decision)

## Alternatives Considered

| Option | Verdict |
|--------|---------|
| `notebooklm` | Too bold, could be mistaken for official |
| `py-notebooklm` | Less conventional than suffix pattern |
| `nlm-py` | Less discoverable |
| `notebooklm-sdk` | Longer, no clear advantage |
