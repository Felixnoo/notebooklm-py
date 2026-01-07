# Python API Reference

Complete reference for the `notebooklm` Python library.

## Migration Guide (v0.x → v1.x)

### Breaking Changes

The service layer has been removed. Methods are now accessed directly on the client through namespaced APIs.

**Before (v0.x):**
```python
from notebooklm import NotebookLMClient
from notebooklm.services import NotebookService, SourceService, ArtifactService

async with await NotebookLMClient.from_storage() as client:
    nb_svc = NotebookService(client)
    src_svc = SourceService(client)
    art_svc = ArtifactService(client)

    notebooks = await nb_svc.list()
    source = await src_svc.add_url(nb_id, url)
    status = await art_svc.generate_audio(nb_id)
```

**After (v1.x):**
```python
from notebooklm import NotebookLMClient

async with await NotebookLMClient.from_storage() as client:
    notebooks = await client.notebooks.list()
    source = await client.sources.add_url(nb_id, url)
    status = await client.artifacts.generate_audio(nb_id)
```

### Method Mapping

| Old (v0.x) | New (v1.x) |
|------------|------------|
| `NotebookService(client).list()` | `client.notebooks.list()` |
| `NotebookService(client).create(title)` | `client.notebooks.create(title)` |
| `SourceService(client).add_url(...)` | `client.sources.add_url(...)` |
| `SourceService(client).add_file(...)` | `client.sources.add_file(...)` |
| `ArtifactService(client).generate_audio(...)` | `client.artifacts.generate_audio(...)` |
| `ArtifactService(client).wait_for_completion(...)` | `client.artifacts.wait_for_completion(...)` |

---

## Quick Start

```python
import asyncio
from notebooklm import NotebookLMClient

async def main():
    # Create client from saved authentication
    async with await NotebookLMClient.from_storage() as client:
        # List notebooks
        notebooks = await client.notebooks.list()
        print(f"Found {len(notebooks)} notebooks")

        # Create a new notebook
        nb = await client.notebooks.create("My Research")
        print(f"Created: {nb.id}")

        # Add sources
        await client.sources.add_url(nb.id, "https://example.com/article")

        # Ask a question
        result = await client.chat.ask(nb.id, "Summarize the main points")
        print(result.answer)

        # Generate a podcast
        status = await client.artifacts.generate_audio(nb.id)
        final = await client.artifacts.wait_for_completion(nb.id, status.task_id)
        print(f"Audio ready: {final.url}")

asyncio.run(main())
```

---

## Core Concepts

### Async Context Manager

The client must be used as an async context manager to properly manage HTTP connections:

```python
# Correct - uses context manager
async with await NotebookLMClient.from_storage() as client:
    ...

# Also correct - manual management
client = await NotebookLMClient.from_storage()
await client.__aenter__()
try:
    ...
finally:
    await client.__aexit__(None, None, None)
```

### Authentication

The client requires valid Google session cookies obtained via browser login:

```python
# From storage file (recommended)
client = await NotebookLMClient.from_storage()
client = await NotebookLMClient.from_storage("/path/to/storage_state.json")

# From AuthTokens directly
from notebooklm import AuthTokens
auth = AuthTokens(
    cookies={"SID": "...", "HSID": "...", ...},
    csrf_token="...",
    session_id="..."
)
client = NotebookLMClient(auth)
```

### Error Handling

The library raises `RPCError` for API failures:

```python
from notebooklm import RPCError

try:
    result = await client.notebooks.create("Test")
except RPCError as e:
    print(f"RPC failed: {e}")
    # Common causes:
    # - Session expired (re-run `notebooklm login`)
    # - Rate limited (wait and retry)
    # - Invalid parameters
```

### Refreshing Authentication

Sessions can expire. Refresh without re-logging in:

```python
async with await NotebookLMClient.from_storage() as client:
    # Refresh CSRF token and session ID
    await client.refresh_auth()
```

---

## API Reference

### NotebookLMClient

Main client class providing access to all APIs.

```python
class NotebookLMClient:
    notebooks: NotebooksAPI    # Notebook operations
    sources: SourcesAPI        # Source management
    artifacts: ArtifactsAPI    # AI-generated content
    chat: ChatAPI              # Conversations
    research: ResearchAPI      # Web/Drive research
    notes: NotesAPI            # User notes

    @classmethod
    async def from_storage(cls, path: str = None) -> "NotebookLMClient"

    async def refresh_auth(self) -> AuthTokens
```

---

### NotebooksAPI (`client.notebooks`)

| Method | Parameters | Returns | Description |
|--------|------------|---------|-------------|
| `list()` | - | `list[Notebook]` | List all notebooks |
| `create(title)` | `title: str` | `Notebook` | Create a notebook |
| `get(notebook_id)` | `notebook_id: str` | `Notebook` | Get notebook details |
| `delete(notebook_id)` | `notebook_id: str` | `bool` | Delete a notebook |
| `rename(notebook_id, new_title)` | `notebook_id: str, new_title: str` | `Notebook` | Rename a notebook |
| `get_description(notebook_id)` | `notebook_id: str` | `NotebookDescription` | Get AI summary and topics |

**Example:**
```python
# List all notebooks
notebooks = await client.notebooks.list()
for nb in notebooks:
    print(f"{nb.id}: {nb.title} ({nb.sources_count} sources)")

# Create and rename
nb = await client.notebooks.create("Draft")
nb = await client.notebooks.rename(nb.id, "Final Version")

# Get AI-generated description
desc = await client.notebooks.get_description(nb.id)
print(desc.summary)
for topic in desc.suggested_topics:
    print(f"  - {topic.title}")
```

---

### SourcesAPI (`client.sources`)

| Method | Parameters | Returns | Description |
|--------|------------|---------|-------------|
| `list(notebook_id)` | `notebook_id: str` | `list[Source]` | List sources |
| `get(notebook_id, source_id)` | `str, str` | `Source` | Get source details |
| `add_url(notebook_id, url)` | `str, str` | `Source` | Add URL source |
| `add_youtube(notebook_id, url)` | `str, str` | `Source` | Add YouTube video |
| `add_text(notebook_id, title, content)` | `str, str, str` | `Source` | Add text content |
| `add_file(notebook_id, path, mime_type=None)` | `str, Path, str` | `Source` | Upload file |
| `add_drive(notebook_id, file_id, title, mime_type)` | `str, str, str, str` | `Source` | Add Google Drive doc |
| `rename(notebook_id, source_id, new_title)` | `str, str, str` | `Source` | Rename source |
| `refresh(notebook_id, source_id)` | `str, str` | `bool` | Refresh URL/Drive source |
| `delete(notebook_id, source_id)` | `str, str` | `bool` | Delete source |

**Example:**
```python
# Add various source types
await client.sources.add_url(nb_id, "https://example.com/article")
await client.sources.add_youtube(nb_id, "https://youtube.com/watch?v=...")
await client.sources.add_text(nb_id, "My Notes", "Content here...")
await client.sources.add_file(nb_id, Path("./document.pdf"))

# List and manage
sources = await client.sources.list(nb_id)
for src in sources:
    print(f"{src.id}: {src.title} ({src.source_type})")

await client.sources.rename(nb_id, src.id, "Better Title")
await client.sources.refresh(nb_id, src.id)  # Re-fetch URL content
```

---

### ArtifactsAPI (`client.artifacts`)

| Method | Parameters | Returns | Description |
|--------|------------|---------|-------------|
| `list(notebook_id, type=None)` | `str, StudioContentType` | `list[Artifact]` | List artifacts |
| `get(notebook_id, artifact_id)` | `str, str` | `Artifact` | Get artifact details |
| `delete(notebook_id, artifact_id)` | `str, str` | `bool` | Delete artifact |
| `rename(artifact_id, new_title)` | `str, str` | `bool` | Rename artifact |
| `poll_status(notebook_id, task_id)` | `str, str` | `ArtifactStatus` | Check generation status |
| `wait_for_completion(notebook_id, task_id, ...)` | `str, str, ...` | `ArtifactStatus` | Wait for generation |
| `generate_audio(...)` | See below | `GenerationStatus` | Generate podcast |
| `generate_video(...)` | See below | `GenerationStatus` | Generate video |
| `generate_report(...)` | See below | `GenerationStatus` | Generate report |
| `generate_quiz(...)` | See below | `GenerationStatus` | Generate quiz |
| `generate_flashcards(...)` | See below | `GenerationStatus` | Generate flashcards |
| `generate_slides(...)` | See below | `GenerationStatus` | Generate slide deck |
| `generate_infographic(...)` | See below | `GenerationStatus` | Generate infographic |
| `generate_data_table(...)` | See below | `GenerationStatus` | Generate data table |
| `generate_mind_map(...)` | See below | `GenerationStatus` | Generate mind map |

**Generation Methods:**

```python
# Audio (podcast)
status = await client.artifacts.generate_audio(
    notebook_id,
    source_ids=None,        # List of source IDs (None = all)
    instructions="...",     # Custom instructions
    format=AudioFormat.DEEP_DIVE,  # DEEP_DIVE, BRIEF, CRITIQUE, DEBATE
    length=AudioLength.DEFAULT,    # SHORT, DEFAULT, LONG
    language="en"
)

# Video
status = await client.artifacts.generate_video(
    notebook_id,
    source_ids=None,
    instructions="...",
    format=VideoFormat.EXPLAINER,  # EXPLAINER, BRIEF
    style=VideoStyle.AUTO,         # AUTO, CLASSIC, WHITEBOARD, KAWAII, ANIME, etc.
    language="en"
)

# Report
status = await client.artifacts.generate_report(
    notebook_id,
    source_ids=None,
    title="...",
    description="...",
    format=ReportFormat.STUDY_GUIDE,  # BRIEFING_DOC, STUDY_GUIDE, BLOG_POST, CUSTOM
    language="en"
)

# Quiz
status = await client.artifacts.generate_quiz(
    notebook_id,
    source_ids=None,
    instructions="...",
    quantity=QuizQuantity.STANDARD,    # FEWER, STANDARD
    difficulty=QuizDifficulty.MEDIUM,  # EASY, MEDIUM, HARD
    language="en"
)
```

**Waiting for Completion:**

```python
# Start generation
status = await client.artifacts.generate_audio(nb_id)

# Wait with polling
final = await client.artifacts.wait_for_completion(
    nb_id,
    status.task_id,
    timeout=300,      # Max wait time in seconds
    poll_interval=5   # Seconds between polls
)

if final.is_complete:
    print(f"Download URL: {final.url}")
else:
    print(f"Failed or timed out: {final.status}")
```

---

### ChatAPI (`client.chat`)

| Method | Parameters | Returns | Description |
|--------|------------|---------|-------------|
| `ask(notebook_id, question, ...)` | `str, str, ...` | `AskResult` | Ask a question |
| `configure(notebook_id, ...)` | `str, ...` | `bool` | Set chat persona |
| `get_history(notebook_id)` | `str` | `list[ConversationTurn]` | Get conversation |

**Example:**
```python
# Ask questions
result = await client.chat.ask(nb_id, "What are the main themes?")
print(result.answer)

# Continue conversation
result = await client.chat.ask(
    nb_id,
    "Can you elaborate on the first point?",
    conversation_id=result.conversation_id
)

# Configure persona
await client.chat.configure(
    nb_id,
    goal=ChatGoal.LEARNING_GUIDE,
    response_length=ChatResponseLength.LONGER,
    custom_prompt="Focus on practical applications"
)
```

---

### ResearchAPI (`client.research`)

| Method | Parameters | Returns | Description |
|--------|------------|---------|-------------|
| `start_fast(notebook_id, query, source)` | `str, str, str` | `dict` | Start fast research |
| `start_deep(notebook_id, query, source)` | `str, str, str` | `dict` | Start deep research |
| `poll_status(notebook_id)` | `str` | `dict` | Check research status |
| `import_sources(notebook_id, task_id, sources)` | `str, str, list` | `bool` | Import findings |

**Example:**
```python
# Start deep web research
result = await client.research.start_deep(nb_id, "AI safety regulations", "web")
task_id = result["task_id"]

# Poll until complete
import asyncio
while True:
    status = await client.research.poll_status(nb_id)
    if status["status"] == "completed":
        break
    await asyncio.sleep(10)

# Import discovered sources
await client.research.import_sources(nb_id, task_id, status["sources"])
```

---

### NotesAPI (`client.notes`)

| Method | Parameters | Returns | Description |
|--------|------------|---------|-------------|
| `list(notebook_id)` | `str` | `list[Note]` | List notes |
| `create(notebook_id, content, title=None)` | `str, str, str` | `Note` | Create note |
| `get(notebook_id, note_id)` | `str, str` | `Note` | Get note |
| `update(notebook_id, note_id, content, title=None)` | `str, str, str, str` | `Note` | Update note |
| `delete(notebook_id, note_id)` | `str, str` | `bool` | Delete note |

---

## Data Types

### Notebook

```python
@dataclass
class Notebook:
    id: str
    title: str
    created_at: Optional[datetime]
    sources_count: int
    is_owner: bool
```

### Source

```python
@dataclass
class Source:
    id: str
    title: Optional[str]
    url: Optional[str]
    source_type: str  # "url", "youtube", "text", "pdf", "upload", etc.
    created_at: Optional[datetime]
```

### Artifact

```python
@dataclass
class Artifact:
    id: str
    title: Optional[str]
    artifact_type: StudioContentType
    status: str  # "in_progress", "completed", "failed"
    url: Optional[str]
    created_at: Optional[datetime]
```

### AskResult

```python
@dataclass
class AskResult:
    answer: str
    conversation_id: str
    sources_used: list[str]
```

---

## Enums

### Audio Generation

```python
class AudioFormat(Enum):
    DEEP_DIVE = 1   # In-depth discussion
    BRIEF = 2       # Quick summary
    CRITIQUE = 3    # Critical analysis
    DEBATE = 4      # Two-sided debate

class AudioLength(Enum):
    SHORT = 1
    DEFAULT = 2
    LONG = 3
```

### Video Generation

```python
class VideoFormat(Enum):
    EXPLAINER = 1
    BRIEF = 2

class VideoStyle(Enum):
    AUTO = 1
    CUSTOM = 2
    CLASSIC = 3
    WHITEBOARD = 4
    KAWAII = 5
    ANIME = 6
    WATERCOLOR = 7
    RETRO_PRINT = 8
    HERITAGE = 9
    PAPER_CRAFT = 10
```

### Quiz/Flashcards

```python
class QuizQuantity(Enum):
    FEWER = 1
    STANDARD = 2

class QuizDifficulty(Enum):
    EASY = 1
    MEDIUM = 2
    HARD = 3
```

### Reports

```python
class ReportFormat(Enum):
    BRIEFING_DOC = 1
    STUDY_GUIDE = 2
    BLOG_POST = 3
    CUSTOM = 4
```

### Infographics

```python
class InfographicOrientation(Enum):
    LANDSCAPE = 1
    PORTRAIT = 2
    SQUARE = 3

class InfographicDetail(Enum):
    CONCISE = 1
    STANDARD = 2
    DETAILED = 3
```

### Slide Decks

```python
class SlideDeckFormat(Enum):
    DETAILED = 1
    PRESENTER = 2

class SlideDeckLength(Enum):
    DEFAULT = 1
    SHORT = 2
```

### Chat Configuration

```python
class ChatGoal(Enum):
    DEFAULT = 1        # General purpose
    CUSTOM = 2         # Uses custom_prompt
    LEARNING_GUIDE = 3 # Educational focus

class ChatResponseLength(Enum):
    DEFAULT = 1
    LONGER = 4
    SHORTER = 5
```

---

## Advanced Usage

### Custom RPC Calls

For undocumented features, you can make raw RPC calls:

```python
from notebooklm.rpc import RPCMethod

async with await NotebookLMClient.from_storage() as client:
    # Access the core client for raw RPC
    result = await client._core.rpc_call(
        RPCMethod.SOME_METHOD,
        params=[...],
        source_path="/notebook/123"
    )
```

### Handling Rate Limits

Google rate limits aggressive API usage:

```python
import asyncio
from notebooklm import RPCError

async def safe_create_notebooks(client, titles):
    for title in titles:
        try:
            await client.notebooks.create(title)
        except RPCError:
            # Wait and retry on rate limit
            await asyncio.sleep(10)
            await client.notebooks.create(title)
        # Add delay between operations
        await asyncio.sleep(2)
```

### Streaming Chat Responses

The chat endpoint supports streaming (internal implementation):

```python
# Standard (non-streaming) - recommended
result = await client.chat.ask(nb_id, "Question")
print(result.answer)

# Streaming is handled internally by the library
# The ask() method returns the complete response
```
