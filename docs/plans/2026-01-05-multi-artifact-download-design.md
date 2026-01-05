# Multi-Artifact Download Design

**Date:** 2026-01-05
**Status:** Approved
**Primary Use Case:** LLM agents with occasional human users

## Overview

Enhance download commands to intelligently handle scenarios where multiple artifacts of the same type exist in a notebook. Support natural language selection (latest, earliest, by name) with smart defaults that work seamlessly for LLM-driven interactions.

## Command Interface

All download commands support natural language selection:

```bash
notebooklm download audio [OUTPUT_PATH] [OPTIONS]
notebooklm download video [OUTPUT_PATH] [OPTIONS]
notebooklm download slide-deck [OUTPUT_DIR] [OPTIONS]
notebooklm download infographic [OUTPUT_PATH] [OPTIONS]

Arguments:
  OUTPUT_PATH    Optional. File path or directory.
                 - If omitted: smart defaults (see below)
                 - If directory: saves there with artifact title
                 - If file path: uses exact name

Selection Options:
  -n, --notebook TEXT     Notebook ID (uses current if not set)
  --latest               Download most recent (default if multiple found)
  --earliest             Download oldest
  --all                  Download all (OUTPUT_PATH must be directory)
  --name TEXT            Match by artifact title (fuzzy search)
  --artifact-id ID       Exact artifact ID

Output Options:
  --json                 Output structured JSON instead of text
  --dry-run              Preview what would be downloaded without downloading
  --force                Overwrite existing files
  --no-clobber           Skip if file exists (vs. auto-rename)
```

## Smart Defaults

### Single Artifact Downloads
**Default OUTPUT_PATH:** `./[artifact-title].[ext]`

Downloads to current working directory with auto-named file.

Example: `./Deep Dive Overview.mp3`

### Multiple Artifacts (--all)
**Default OUTPUT_PATH:** `./<type>/`

Creates subdirectory by artifact type:
- Audio: `./audio/`
- Video: `./video/`
- Slide-deck: `./slide-deck/`
- Infographic: `./infographic/`

## Selection Behavior

**CRITICAL: Filter → Count → Select Logic**

The selection process follows this order:
1. **Filter** artifacts by `--name` or `--artifact-id` if provided
2. **Count** how many artifacts match the filter
3. **Select** based on count and remaining flags

### Zero Matches
- Error: `No audio artifacts found. Generate one with: notebooklm generate audio`
- If filter was used: Show available artifacts that didn't match

### Single Match
- Downloads the matched artifact
- Respects the filter (critical: if `--name "music"` matches nothing, error even if other artifacts exist)
- Output: `Downloaded: "Deep Dive Overview.mp3"`

### Multiple Matches (No --all)
- **Default:** Selects latest by creation timestamp
- **With --earliest:** Selects oldest by creation timestamp
- Output: `Downloaded latest: "Deep Dive Overview.mp3" (1 of 3 total)`
- Hint: `Tip: Use --earliest, --name, or --all for other artifacts`

### Multiple Matches + --all
- OUTPUT_PATH must be directory
- Downloads all matched artifacts, auto-naming by title
- Handles title conflicts with " (2)", " (3)" suffixes
- Output: `Downloaded 3 audio overviews to ./audio/`

### Filter Examples

**Correct behavior:**
```bash
# Only 1 artifact exists: "Meeting.mp3"
$ notebooklm download audio --name "music"
Error: No audio artifacts matching "music"
Available:
  - Meeting (Jan 5, 15:30)
# Does NOT download "Meeting.mp3" - filter is respected
```

**Multiple artifacts with filter:**
```bash
# 5 artifacts exist, 2 match "debate"
$ notebooklm download audio --name "debate"
Downloaded latest: "Debate Format.mp3" (1 of 2 matching)
# Downloads latest of the 2 matches, not latest of all 5
```

## Usage Examples

### LLM-Friendly Commands
```bash
# LLM: "download the latest audio"
$ notebooklm download audio
→ ./Deep Dive Overview.mp3

# LLM: "download all the videos"
$ notebooklm download video --all
→ ./video/Explainer.mp4
→ ./video/Brief Summary.mp4

# LLM: "download the debate audio"
$ notebooklm download audio --name debate
→ ./Debate Format.mp3

# LLM: "download the earliest infographic to ~/images/"
$ notebooklm download infographic ~/images/ --earliest
→ ~/images/Timeline Overview.png
```

## Structured Output & Preview

### --json Flag
For LLM-friendly parsing, output structured JSON instead of text:

```bash
$ notebooklm download audio --json
{
  "downloaded": [
    {
      "id": "abc123",
      "title": "Deep Dive Overview",
      "path": "/Users/me/Deep Dive Overview.mp3",
      "size_bytes": 15728640
    }
  ],
  "skipped": [],
  "failed": [],
  "total_found": 3,
  "total_downloaded": 1
}
```

**With --all:**
```bash
$ notebooklm download audio --all --json
{
  "downloaded": [
    {"id": "abc", "title": "Deep Dive", "path": "./audio/Deep Dive.mp3"},
    {"id": "def", "title": "Brief", "path": "./audio/Brief.mp3"}
  ],
  "skipped": [],
  "failed": [],
  "total_found": 2,
  "total_downloaded": 2
}
```

**Errors also JSON:**
```bash
$ notebooklm download audio --name "xyz" --json
{
  "error": "No audio artifacts matching 'xyz'",
  "available": [
    {"id": "abc", "title": "Deep Dive", "created": "2026-01-05T15:30:00Z"},
    {"id": "def", "title": "Brief", "created": "2026-01-04T10:15:00Z"}
  ],
  "total_found": 2
}
```

### --dry-run Flag
Preview what would be downloaded without actually downloading:

```bash
$ notebooklm download audio --dry-run
Would download: "Deep Dive Overview.mp3" (latest, 15.0 MB)
Total: 1 of 3 artifacts
```

**With --all:**
```bash
$ notebooklm download audio --all --dry-run
Would download to ./audio/:
  1. Deep Dive Overview.mp3 (15.0 MB)
  2. Brief Summary.mp3 (8.5 MB)
  3. Debate Format.mp3 (12.3 MB)
Total: 3 artifacts, 35.8 MB
```

**Combines with --json:**
```bash
$ notebooklm download audio --dry-run --json
{
  "would_download": [
    {
      "id": "abc123",
      "title": "Deep Dive Overview",
      "path": "/Users/me/Deep Dive Overview.mp3",
      "size_bytes": 15728640,
      "exists": false
    }
  ],
  "total_found": 3,
  "total_size_bytes": 15728640
}
```

## Implementation

### Helper Functions

**Artifact Selection:**
```python
def select_artifact(
    artifacts: list,
    latest: bool = True,
    earliest: bool = False,
    name: Optional[str] = None,
    artifact_id: Optional[str] = None
) -> tuple[Any, str]:
    """
    Select an artifact from a list based on criteria.

    CRITICAL: Implements Filter → Count → Select logic:
    1. Filter artifacts by name/artifact_id if provided
    2. Count matches (0/1/many)
    3. Apply latest/earliest to remaining matches

    Returns: (selected_artifact, selection_reason)
    Raises: ValueError if no match or invalid criteria
    """
```

Responsibilities:
1. Validate mutually exclusive flags (--latest + --earliest, etc.)
2. **Filter first:** Apply --name or --artifact-id filter if provided
3. **Count matches:** Determine if 0, 1, or many artifacts remain
4. **Select:** Apply --latest/--earliest only if multiple matches
5. Return selected artifact + human-readable reason (e.g., "latest", "matched by name")
6. Raise clear errors with available options when no match

**Filename Sanitization:**
```python
def artifact_title_to_filename(
    title: str,
    extension: str,
    existing_files: set
) -> str:
    """
    Convert artifact title to safe filename.

    - Sanitizes special characters
    - Handles conflicts with " (2)", " (3)", etc.
    - Adds extension
    """
```

### Integration Points

1. Modify existing download commands (`download_audio`, `download_video`, etc.)
2. Make OUTPUT_PATH optional with Click
3. Add selection flags (--latest, --earliest, --all, --name)
4. Call `select_artifact()` before `client.download_*()`
5. Handle smart defaults for OUTPUT_PATH

## Error Handling

### Invalid Flag Combinations
```bash
$ notebooklm download audio --latest --earliest
Error: Cannot use --latest and --earliest together
```

### --all with File Path
```bash
$ notebooklm download audio ./podcast.mp3 --all
Error: --all requires OUTPUT_PATH to be a directory, not a file
```

### No Matches for --name
```bash
$ notebooklm download audio --name "xyz"
Error: No audio artifacts matching "xyz"
Available:
  - Deep Dive Overview (Jan 5, 15:30)
  - Brief Summary (Jan 4, 10:15)
  - Debate Format (Jan 3, 09:00)
```

### Filename Conflicts Within Batch (--all)
- Append " (2)", " (3)" for duplicate artifact titles
- Example: "Overview.mp3", "Overview (2).mp3"

### Existing File Conflicts
**Default behavior:** Auto-rename to avoid overwriting

```bash
# File already exists
$ ls
Deep Dive Overview.mp3

$ notebooklm download audio
Downloaded: "Deep Dive Overview (1).mp3"
```

**Override with flags:**
- `--force`: Overwrite existing files silently
- `--no-clobber`: Skip download if file exists, print warning

```bash
$ notebooklm download audio --force
Downloaded: "Deep Dive Overview.mp3" (overwritten)

$ notebooklm download audio --no-clobber
Skipped: "Deep Dive Overview.mp3" (already exists)
```

### Download Failures (--all)
- Continue downloading others if one fails
- Print summary: `Downloaded 2 of 3 (1 failed)`

## Design Principles

1. **LLM-First:** Commands work without explicit paths or indices
2. **Smart Defaults:** Latest is implicit default for multiple artifacts
3. **Natural Language:** Use semantic selectors (--name, --latest) not numeric indices
4. **Non-Blocking:** Never prompt for interactive input
5. **Helpful Hints:** Print tips when multiple options exist
6. **YAGNI:** No complex filtering or sorting beyond latest/earliest/name

## Testing Considerations

### Critical Filter Logic Tests
- **Filter respected with single artifact:** If only "Meeting.mp3" exists, `--name "music"` must error (not download Meeting)
- **Filter narrows before latest/earliest:** 5 artifacts, 2 match "debate", `--name debate` downloads latest of 2 (not latest of 5)
- **Zero matches shows available:** `--name "xyz"` with no matches lists all available artifacts

### Selection Logic Tests
- Test with 0, 1, and multiple artifacts
- Verify fuzzy name matching (case-insensitive, substring)
- Test all flag combinations for mutual exclusivity (--latest + --earliest)
- Verify --all flag validation (directory required)

### File Handling Tests
- **Filename sanitization:** Special characters, long titles
- **Conflict resolution within batch:** Duplicate titles get " (2)", " (3)"
- **Existing file conflicts:** Default auto-rename, --force overwrites, --no-clobber skips
- Verify smart default paths for each artifact type
- Test --all with directory creation

### Output Format Tests
- **--json:** Valid JSON for success and error cases
- **--dry-run:** No actual downloads, accurate previews
- **--json + --dry-run:** Combined correctly

### Error Handling Tests
- Invalid flag combinations produce clear errors
- Network failures handled gracefully
- --all continues on partial failures
- Missing permissions produce helpful errors

## Design Review Feedback

**Reviewed by:** Gemini CLI (2026-01-05)

### Critical Issues Identified
1. **Filter logic was backwards:** Original design would ignore filters when only 1 artifact existed
   - **Fixed:** Implemented Filter → Count → Select order
   - **Test case:** `--name "music"` with only "Meeting.mp3" must error, not download

### Important Improvements
2. **File overwrite behavior was undefined**
   - **Added:** Default auto-rename `(1)`, `(2)` suffixes
   - **Added:** `--force` and `--no-clobber` flags for override

### Valuable Additions
3. **Structured output for LLM parsing**
   - **Added:** `--json` flag for all commands
   - **Benefit:** Eliminates text parsing errors in agents

4. **Preview capability**
   - **Added:** `--dry-run` flag
   - **Benefit:** Agents can check before downloading large files

### Assessment
Original design: ~70% complete
Updated design: Production-ready for agentic workflows

**Key quote from review:**
> "The design is ~90% there. Fixing the 'Single Artifact' filter logic and adding --json output would make it production-ready for agentic workflows."
