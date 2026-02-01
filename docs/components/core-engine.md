# Core Engine

The Core Engine is the heart of Docrunch, responsible for scanning, parsing, and analyzing codebases.

## Components

```
docrunch/core/
- scanner.py      # File tree scanner
- parser.py       # AST parser (tree-sitter)
- analyzer.py     # Relationship analyzer
- watcher.py      # Real-time file watcher
- task_manager.py # Task orchestration
- llm/            # LLM providers
  - base.py
  - openai.py
  - anthropic.py
  - cli_bridge.py
  - ollama.py

docrunch/storage/
- markdown.py     # Markdown documentation output
```

---

## Scanner (`scanner.py`)

Scans repository file structure and generates a complete file tree.

### Responsibilities

- Walk directory tree recursively
- Detect file types and languages
- Respect `.gitignore` and `.docrunch/ignore.yaml`
- Track file metadata (size, modified date, hash)

### Key Functions

```python
class Scanner:
    def scan(self, path: Path) -> FileTree:
        """Scan directory and return file tree."""

    def get_files_by_language(self, language: str) -> list[Path]:
        """Filter files by programming language."""

    def get_changed_files(self, since: datetime) -> list[Path]:
        """Get files changed since timestamp."""
```

### Output

```python
@dataclass
class FileNode:
    path: Path
    name: str
    type: Literal["file", "directory"]
    language: str | None  # python, typescript, etc.
    size_bytes: int
    modified_at: datetime
    content_hash: str  # content hash for change detection
    children: list[FileNode]  # if directory
```

---

## Parser (`parser.py`)

Parses source code into Abstract Syntax Trees using tree-sitter.

### Supported Languages

MVP:

- Python (`.py`)
- TypeScript (`.ts`, `.tsx`)

Planned (Phase 2+):

- JavaScript (`.js`, `.jsx`)
- SQL (`.sql`)
- CSS/SCSS (`.css`, `.scss`)

### Extracted Elements

```python
@dataclass
class ParsedFile:
    file_path: Path
    language: str
    success: bool
    error: str | None

    imports: list[Import]
    exports: list[Export]
    functions: list[Function]
    classes: list[Class]
    components: list[Component]  # React/Vue
    schemas: list[Schema]  # SQL/ORM
```

### Key Functions

```python
class Parser:
    def parse_file(self, path: Path) -> ParsedFile:
        """Parse single file into AST."""

    def extract_functions(self, ast: Tree) -> list[Function]:
        """Extract function definitions."""

    def extract_imports(self, ast: Tree) -> list[Import]:
        """Extract import statements."""
```

---

## Analyzer (`analyzer.py`)

Analyzes relationships between code elements across files.

### Relationship Types

| Type         | Description                      |
| ------------ | -------------------------------- |
| `imports`    | File A imports from File B       |
| `exports`    | File A exports to File B         |
| `calls`      | Function A calls Function B      |
| `inherits`   | Class A extends Class B          |
| `implements` | Class implements Interface       |
| `uses`       | Component uses another Component |

### Key Functions

```python
class Analyzer:
    def analyze(self, parsed_files: list[ParsedFile]) -> RelationshipGraph:
        """Build relationship graph from parsed files."""

    def find_dependencies(self, file: Path) -> list[Dependency]:
        """Find all dependencies of a file."""

    def find_dependents(self, file: Path) -> list[Dependent]:
        """Find all files that depend on this file."""

    def detect_patterns(self) -> list[Pattern]:
        """Detect common code patterns."""

    def find_circular_dependencies(self) -> list[Cycle]:
        """Detect circular dependency chains."""
```

### Output

```python
@dataclass
class RelationshipGraph:
    nodes: dict[str, CodeNode]  # path -> node
    edges: list[Edge]  # relationships
    patterns: list[Pattern]

@dataclass
class Edge:
    source: str
    target: str
    type: str  # imports, calls, inherits, etc.
    metadata: dict
```

---

## LLM Providers (`llm/`)

Configurable LLM integration for intelligent code analysis.

### Base Interface

```python
class BaseLLMProvider(ABC):
    @abstractmethod
    async def analyze_code(self, code: str, context: str) -> Analysis:
        """Analyze code and generate insights."""

    @abstractmethod
    async def summarize(self, content: str) -> str:
        """Generate human-readable summary."""

    @abstractmethod
    async def explain(self, code: str) -> str:
        """Explain what code does."""
```

### Provider Implementations

- **OpenAI** (`openai.py`) - GPT-4, GPT-4o-mini
- **Anthropic** (`anthropic.py`) - Claude 3.5 Sonnet
- **Ollama** (`ollama.py`) - Local models (CodeLlama, etc.)
- **CLI Bridge** (`cli_bridge.py`) - Local CLI tools via subprocess or proxy

### Configuration

```yaml
# .docrunch/config.yaml
llm:
  settings_source: database
  default_profile: manager
  allow_ui_updates: true
  bootstrap_from_env: false
```

Providers and specialist profiles are stored in Postgres and managed
via the dashboard UI. CLI Bridge models live in `.docrunch/cli_models.json`.
Local dev can optionally bootstrap settings from env.

---

## Documentation Generation

> **Note:** The `MarkdownGenerator` class lives in `docrunch/storage/markdown.py` but is invoked by the Core Engine after analysis completes. See [Storage Layer](./storage.md) for output format details.

The Core Engine coordinates when documentation is generated, but the actual file writing and template rendering is handled by the Storage Layer.

### Output Structure

```
docrunch-docs/
- index.md           # Main index
- architecture.md    # System overview
- modules/           # Per-module docs
  - auth.md
  - api.md
- functions/         # Function reference
- schemas/           # Database schemas
- patterns/          # Code patterns
- decisions/         # Architecture decisions
```

### Templates

- Module overview
- Function documentation
- Schema definitions
- Pattern descriptions
- Mermaid diagrams for relationships

---

## Watcher (`watcher.py`)

Real-time file system monitoring for incremental updates.

Status: Planned post-MVP (Phase 2+).

### Technology

- Uses `watchdog` library
- Debounced updates (configurable)
- Filters by configured ignore patterns

### Key Functions

```python
class Watcher:
    def start(self, path: Path, callback: Callable):
        """Start watching directory."""

    def stop(self):
        """Stop watching."""

    async def on_change(self, event: FileEvent):
        """Handle file change event."""
```

### Events

- File created
- File modified
- File deleted
- File renamed

---

## Task Manager (`task_manager.py`)

Orchestrates AI agent tasks through the Kanban pipeline.

See [Task System](./task-system.md) for detailed documentation.
