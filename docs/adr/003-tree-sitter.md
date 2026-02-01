# ADR-003: tree-sitter for AST Parsing

## Status

**Accepted**

## Context

Docrunch needs to parse source code in multiple languages to extract functions, classes, imports, and relationships.

## Options Considered

| Option                    | Pros                                              | Cons                             |
| ------------------------- | ------------------------------------------------- | -------------------------------- |
| **tree-sitter**           | Multi-language, fast, incremental, error-tolerant | C bindings required              |
| Python ast module         | Native Python, no deps                            | Python only                      |
| esprima/acorn             | Popular for JS                                    | JavaScript only                  |
| Language-specific parsers | Precise                                           | N parsers for N languages        |
| Regex-based               | Simple                                            | Unreliable, breaks on edge cases |

## Decision

Use **tree-sitter** because:

1. Single library for all languages
2. Incremental parsing (fast for large files)
3. Error-tolerant (parses partial/broken code)
4. Concrete syntax tree preserves all tokens
5. Battle-tested (used in GitHub, Neovim)

## Supported Languages (MVP)

1. Python (`tree-sitter-python`)
2. TypeScript (`tree-sitter-typescript`)
3. JavaScript (`tree-sitter-javascript`)

## Post-MVP Languages

- SQL, CSS, Go, Rust, Java

## Fallback Plan

For unsupported languages:

- Basic file metadata only (no AST)
- Use LLM for summarization

## Consequences

- Need to install tree-sitter language packages
- Some Python packaging complexity (C extension)
- Must pre-build grammars or use py-tree-sitter-languages
