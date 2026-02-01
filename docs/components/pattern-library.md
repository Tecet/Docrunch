# Pattern Library

Curated collection of code patterns, anti-patterns, and examples for consistent AI agent implementations.

## Overview

The Pattern Library maintains a repository of approved patterns that AI agents should follow, with code examples and anti-patterns to avoid.

Approved patterns:
- auth-middleware (12 usages)
- api-error-handling (45 usages)
- database-transaction (8 usages)
- react-form-hook (23 usages)

Anti-patterns (avoid):
- callback-hell
- god-object
- magic-numbers

## Data Model

```python
@dataclass
class Pattern:
    id: str
    name: str
    description: str
    category: str  # auth, api, database, ui, testing
    language: str  # python, typescript, etc.

    # Code
    example_code: str
    file_references: list[str]  # Files using this pattern

    # Metadata
    created_by: str
    created_at: datetime
    usage_count: int

    # Guidelines
    when_to_use: str
    when_not_to_use: str
    related_patterns: list[str]

@dataclass
class AntiPattern:
    id: str
    name: str
    description: str
    why_bad: str
    correct_alternative: str  # Link to correct pattern
    example_bad_code: str
    example_good_code: str
```

## Pattern Definition

```yaml
# .docrunch/patterns/auth-middleware.yaml
pattern:
  name: auth-middleware
  description: "JWT authentication middleware for API routes"
  category: auth
  language: python

  example: |
    from functools import wraps
    from flask import request, jsonify
    import jwt

    def require_auth(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            token = request.headers.get('Authorization', '').replace('Bearer ', '')
            if not token:
                return jsonify({'error': 'Missing token'}), 401
            try:
                payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
                request.user = payload
            except jwt.InvalidTokenError:
                return jsonify({'error': 'Invalid token'}), 401
            return f(*args, **kwargs)
        return decorated

  when_to_use:
    - "Protecting API endpoints that require authentication"
    - "Extracting user info from JWT for route handlers"

  when_not_to_use:
    - "Public endpoints"
    - "Webhooks with different auth mechanisms"

  related_patterns:
    - api-error-handling
    - rate-limiting
```

## Features

### 1. Automatic Pattern Detection

Scan codebase to find pattern usages:

```python
def detect_patterns(file: Path) -> list[PatternMatch]:
    """Detect which patterns are used in a file."""
    ast = parse_file(file)
    matches = []

    for pattern in get_all_patterns():
        if pattern.matches(ast):
            matches.append(PatternMatch(
                pattern=pattern,
                file=file,
                line=find_line(ast, pattern)
            ))

    return matches
```

### 2. Pattern Enforcement

Guardrails that reference patterns:

```yaml
task:
  guardrails:
    - "Follow pattern: auth-middleware"
    - "Avoid anti-pattern: callback-hell"
```

### 3. Pattern Suggestions

AI suggests new patterns from repeated code:

```python
@server.call_tool()
async def suggest_pattern(name: str, description: str, example: str) -> str:
    """Suggest a new pattern for review."""
```

## MCP Tools

| Tool                    | Description                     |
| ----------------------- | ------------------------------- |
| `get_pattern`           | Get pattern details and example |
| `find_pattern_usages`   | Find where pattern is used      |
| `check_anti_patterns`   | Check code for anti-patterns    |
| `suggest_pattern`       | Suggest new pattern             |
| `get_patterns_for_task` | Get relevant patterns for task  |

## Storage

Patterns stored as YAML files and indexed in Postgres:

```
.docrunch/
- patterns/
  - auth-middleware.yaml
  - api-error-handling.yaml
  - ...
```

## Dashboard Integration

Visual pattern browser:

- Pattern catalog with search
- Usage statistics
- Anti-pattern violations
- Pattern suggestions queue
