# GEMINI.md

## Project Overview

`notebooklm-client` is an unofficial Python client and CLI for Google NotebookLM. It allows programmatic management of notebooks, sources, and AI-generated artifacts (podcasts, videos, quizzes) by reverse-engineering Google's internal `batchexecute` RPC protocol.

**Core Functionality:**
- **Notebook Management:** CRUD operations for notebooks.
- **Source Integration:** Add URLs (including YouTube), raw text, and file uploads.
- **AI Querying:** Chat interface with streaming responses.
- **Studio Artifacts:** Generation of audio/video overviews, study guides, and more.

**Critical Note:** This project relies on obfuscated RPC method IDs (e.g., `CCqFvf`) which Google may change at any time. These are defined in `src/notebooklm/rpc/types.py`.

## Architecture

The project follows a three-layer architecture:

1.  **RPC Layer (`src/notebooklm/rpc/`)**
    -   Handles the low-level `batchexecute` protocol.
    -   `types.py`: **CRITICAL**. Defines all RPC method IDs and parameter enums.
    -   `encoder.py` / `decoder.py`: Serializes and parses the nested list structures used by the API.

2.  **Client Layer (`src/notebooklm/api_client.py`)**
    -   `NotebookLMClient`: The main async API client.
    -   Manages authentication (cookies/headers), request IDs, and session state.
    -   Executes RPC calls via `_rpc_call`.

3.  **Service Layer (`src/notebooklm/services/`)**
    -   Provides high-level, object-oriented abstractions (`NotebookService`, `SourceService`, etc.).
    -   Wraps raw RPC responses into typed Python objects.

## Key Files

-   `src/notebooklm/api_client.py`: Core client implementation.
-   `src/notebooklm/rpc/types.py`: The "source of truth" for API method IDs. If the API breaks, check this file first against network traffic.
-   `src/notebooklm/auth.py`: specific authentication logic using Playwright to handle Google's login flow and persist cookies.
-   `src/notebooklm/notebooklm_cli.py`: Entry point for the CLI application.

## Building and Running

### Installation
This project uses `hatchling` as the build backend.

```bash
# Install in editable mode with all dependencies (including dev and browser support)
pip install -e ".[all]"

# Install playwright browsers (required for login)
playwright install chromium
```

### CLI Usage
The CLI entry point is `notebooklm`.

```bash
notebooklm --help
notebooklm login       # Authenticate via browser
notebooklm list        # List notebooks
```

### Testing
Tests are managed with `pytest`.

```bash
# Run unit and integration tests (fast)
pytest

# Run end-to-end tests (requires valid auth state)
pytest tests/e2e -m e2e

# Run slow tests (e.g., audio/video generation)
pytest -m slow
```

## Development Conventions

-   **Authentication:** The project uses a persistent browser profile (`~/.notebooklm/browser_profile/`) to avoid bot detection during login.
-   **RPC Method Discovery:** New features often require capturing network traffic to identify new method IDs, which are then added to `types.py`.
-   **Code Style:** Follows standard Python conventions.
-   **Testing:** 
    -   Unit tests mock the RPC layer.
    -   E2E tests hit the real Google API and are prone to rate limiting or flake.
