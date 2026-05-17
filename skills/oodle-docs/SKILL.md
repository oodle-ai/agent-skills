---
name: oodle-docs
description: Read Oodle documentation correctly — start from docs.oodle.ai/llms.txt, choose the relevant docs pages, and read the full Markdown source before answering or implementing docs-guided changes.
metadata:
  version: "1.0.0"
  author: oodle-ai
  repository: https://github.com/oodle-ai/agent-skills
  tags: oodle,docs,llms,documentation,research
  globs: ""
  alwaysApply: "false"
---

# Oodle Docs — Documentation Reading Workflow

This skill teaches the agent to use the Oodle documentation site as the source of truth. Always read `https://docs.oodle.ai/llms.txt` first, then read the full Markdown source for every relevant documentation page before answering, writing code, or changing configuration.

## Prerequisites

You need a tool that can fetch web URLs, such as `curl`, `wget`, an agent web-fetch tool, or browser tooling.

Verify docs access before relying on the docs:

```bash
curl -fsSL https://docs.oodle.ai/llms.txt | head
```

If direct internet access is unavailable, explain that the docs could not be fetched and ask the user to provide the relevant docs content or enable network access. Do not guess from memory.

## Command Execution Order

Follow this sequence for every Oodle docs research task:

1. Fetch `https://docs.oodle.ai/llms.txt`.
2. Search the fetched index for pages related to the user's task.
3. Select the most relevant docs page URLs from the index.
4. Fetch the full Markdown source for each selected docs page from the URL listed in `llms.txt`. This per-page file is the primary source for that page.
5. If an indexed page URL is unavailable, fetch `https://docs.oodle.ai/llms-full.txt` and read the full relevant section from that complete docs file instead.
6. Read each full file or full `llms-full.txt` section, not just titles, snippets, summaries, or search results.
7. Base the answer, implementation, or troubleshooting steps on the fetched docs.
8. Cite the docs pages you used by URL in the final answer when the user asks for an explanation or when docs details materially affect the work.

Do not skip `llms.txt`. Do not answer from stale memory when the docs are available.

## Quick Reference

| Task | Command |
|------|---------|
| Fetch docs index | `curl -fsSL https://docs.oodle.ai/llms.txt` |
| Save docs index | `mkdir -p /code/.generated_artifacts && curl -fsSL https://docs.oodle.ai/llms.txt -o /code/.generated_artifacts/oodle-llms.txt` |
| Search the index | `grep -i "<topic>" /code/.generated_artifacts/oodle-llms.txt` |
| Fetch a docs page | `curl -fsSL "<docs-page-url>"` |
| Fetch all docs as one Markdown file | `curl -fsSL https://docs.oodle.ai/llms-full.txt` |
| Read a fetched page fully | `sed -n '1,240p' <page>.md` then continue until EOF |

## Common Operations

### Starting from `llms.txt`

```bash
# ✅ CORRECT — start with the machine-readable docs index
mkdir -p /code/.generated_artifacts
curl -fsSL https://docs.oodle.ai/llms.txt -o /code/.generated_artifacts/oodle-llms.txt

# ❌ WRONG — guessing URLs or relying on memory without checking the index
# "I know Oodle uses ..."
```

### Choosing relevant pages

```bash
# ✅ CORRECT — search the index for the user's topic
grep -i "kubernetes\|helm\|integration" /code/.generated_artifacts/oodle-llms.txt

# ❌ WRONG — reading only the homepage or the first search result
curl -fsSL https://docs.oodle.ai
```

### Reading full docs page files

```bash
# ✅ CORRECT — fetch and read the complete Markdown source for the relevant page
curl -fsSL "https://docs.oodle.ai/path/from/llms-index.md" -o /code/.generated_artifacts/oodle-doc.md
sed -n '1,240p' /code/.generated_artifacts/oodle-doc.md
sed -n '241,480p' /code/.generated_artifacts/oodle-doc.md

# ❌ WRONG — using only search-result snippets, headings, or the first screenful
curl -fsSL "https://docs.oodle.ai/path/from/llms-index.md" | head
```

### Falling back to `llms-full.txt`

```bash
# ✅ CORRECT — if a per-page Markdown URL fails, use the complete docs file
mkdir -p /code/.generated_artifacts
curl -fsSL https://docs.oodle.ai/llms-full.txt -o /code/.generated_artifacts/oodle-llms-full.txt
grep -n "^## .*Kubernetes" /code/.generated_artifacts/oodle-llms-full.txt
# Read from the matching heading through the next heading at the same level before acting.

# ❌ WRONG — treating a failed page fetch as permission to guess
# "The docs page failed, so I assume ..."
```

### Using docs during implementation

```bash
# ✅ CORRECT — verify command names, flags, schemas, and examples against the docs page
mkdir -p /code/.generated_artifacts
curl -fsSL https://docs.oodle.ai/llms.txt -o /code/.generated_artifacts/oodle-llms.txt
grep -i "api key" /code/.generated_artifacts/oodle-llms.txt
curl -fsSL "<api-key-doc-url-from-index>" -o /code/.generated_artifacts/api-key-doc.md
cat /code/.generated_artifacts/api-key-doc.md

# ❌ WRONG — copying an old command from memory without checking current docs
oodle old-command --maybe-valid
```

## Best Practices

### Treat `llms.txt` as the docs table of contents

Use `https://docs.oodle.ai/llms.txt` to discover canonical docs pages. It is the first source to fetch because it is designed for agents and should contain the current page list.

### Read full files before acting

For every selected docs page, read the whole fetched file before making decisions. Important prerequisites, warnings, and examples may appear after the introduction.

When using `llms-full.txt` as a fallback, read the whole matching section from its heading through the next heading at the same level. Include all nested subsections, prose, tables, and code blocks in that range before acting.

### Prefer docs over memory

If the current docs contradict prior knowledge, follow the docs. Mention the discrepancy if it affects existing code, configuration, or user instructions.

### Keep fetched docs as session artifacts

When using shell tools, save fetched docs under `/code/.generated_artifacts/` so they can be inspected later in the session. Do not save temporary docs under `/tmp`.

### Verify examples before presenting them as working commands

If you include commands from the docs in final instructions or code comments, verify that the command names and flags match the full docs page. When credentials or live resources are required, state what could not be executed rather than claiming success.

## Failure Handling

| Error | Cause | Fix |
|-------|-------|-----|
| `404 Not Found` for a docs page | URL is stale, unavailable, or was guessed | Re-fetch `https://docs.oodle.ai/llms.txt`; if the indexed URL still fails, fetch `https://docs.oodle.ai/llms-full.txt` and read the full relevant section |
| `curl: (6) Could not resolve host` | Network or DNS unavailable | Retry once, then ask the user to provide docs content or enable network access |
| No relevant page in `llms.txt` | The topic may be undocumented or named differently | Search related terms, then report that no matching docs page was found |
| Docs conflict with CLI/API behavior | Docs or implementation may be out of date | Report the exact conflict with URLs and observed command output |
| Page is long | Partial reading can miss critical details | Page through the whole file in chunks until EOF before acting |

## References

- Oodle docs index for agents: https://docs.oodle.ai/llms.txt
- Oodle documentation: https://docs.oodle.ai
