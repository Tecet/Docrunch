# Git Integration

Seamless integration with Git for automatic commits, branch-aware scanning, and changelog generation.

## Overview

Git Integration connects Docrunch with your repository's version control for automated documentation management.

- Current branch: feature/auth
- Last commit: abc1234 "Add login endpoint"
- Auto-commits: on
  - Docs updated: 3 files
  - Pending commit: "docs: Update auth module docs"
  - Last sync: 2 minutes ago
- PR ready:
  - #42 "Feature: User Authentication"
  - Summary generated from: task-015, task-016

## Features

### 1. Auto-Commit Documentation

Automatically commit doc changes:

```python
class GitIntegration:
    def commit_docs(self, message: str = None):
        """Commit documentation changes."""
        changed_files = self.get_changed_docs()

        if not changed_files:
            return

        message = message or self.generate_commit_message(changed_files)

        self.repo.index.add(changed_files)
        self.repo.index.commit(f"docs: {message}")
```

### 2. Branch-Aware Scanning

Different documentation per branch:

```python
def scan_branch(branch: str):
    """Scan specific branch without checkout."""
    with temp_worktree(branch) as worktree:
        return scanner.scan(worktree)

def get_docs_for_branch(branch: str) -> Documentation:
    """Get documentation specific to branch."""
    return storage.get_docs(branch=branch)
```

### 3. PR Summary Generation

Generate PR descriptions from completed tasks:

```python
def generate_pr_summary(task_ids: list[str]) -> str:
    """Generate PR description from task reports."""
    tasks = [get_task(id) for id in task_ids]

    return llm.generate(f"""
        Generate a PR description for these completed tasks:
        {format_tasks(tasks)}

        Include:
        - Summary of changes
        - Key decisions made
        - Testing done
        - Breaking changes (if any)
    """)
```

### 4. Changelog Generation

Auto-generate changelogs from tasks:

```python
def generate_changelog(since: str = "last_release") -> str:
    """Generate changelog from completed tasks."""
    tasks = get_tasks_since(since)

    grouped = group_by_type(tasks)

    return f"""
    ## [{version}] - {date}

    ### Features
    {format_tasks(grouped['feature'])}

    ### Bug Fixes
    {format_tasks(grouped['bug_fix'])}

    ### Other
    {format_tasks(grouped['other'])}
    """
```

### 5. Commit Analysis

Analyze commits for documentation updates:

```python
def analyze_commit(commit_hash: str) -> CommitAnalysis:
    """Analyze commit for documentation needs."""
    commit = repo.commit(commit_hash)

    return CommitAnalysis(
        files_changed=commit.stats.files,
        needs_doc_update=detect_doc_needs(commit),
        suggested_updates=suggest_doc_updates(commit)
    )
```

## Configuration

```yaml
# .docrunch/config.yaml
git:
  enabled: true

  auto_commit:
    enabled: true
    prefix: "docs:"
    branch: null # null = current branch

  branch_aware:
    enabled: true
    track_branches:
      - main
      - develop
      - "feature/*"

  pr_integration:
    enabled: true
    template: |
      ## Summary
      {summary}

      ## Changes
      {changes}

      ## Tasks Completed
      {tasks}

  changelog:
    enabled: true
    file: CHANGELOG.md
    format: keepachangelog
```

## CLI Commands

```bash
# Commit documentation changes
docrunch git commit

# Generate PR summary
docrunch git pr-summary --tasks task-015,task-016

# Generate changelog
docrunch git changelog --since v1.0.0

# Scan specific branch
docrunch scan --branch feature/auth
```

## MCP Tools

| Tool                     | Description                        |
| ------------------------ | ---------------------------------- |
| `get_recent_commits`     | Get recent commits with analysis   |
| `generate_pr_summary`    | Generate PR description            |
| `get_branch_diff`        | Get changes between branches       |
| `suggest_commit_message` | Suggest commit message for changes |

## Dashboard

- Branch selector for docs
- Commit history with doc changes
- PR preview
- Changelog preview
