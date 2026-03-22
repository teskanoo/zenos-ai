# Zen DojoTools Library — 4.5.0 'Meridian'

*Friday's unified system utility runner*

---

## Overview

`zen_dojotools_library` is the dispatch front-end for ZenOS-AI's built-in utility functions. It accepts a tool name and a text input, routes to the appropriate handler, and returns a structured result.

The Library is **MCP-exposed**. Friday uses it directly to hash strings, slugify values, and run command interpreter queries. It is also the underlying engine that Kung Fu components invoke via their `command` field.

---

## Tools

| Tool | What It Does |
|---|---|
| `library` | Routes query through `command_interpreter.jinja` (v1). Returns `{query, output}`. |
| `library_2` | Routes query through `command_interpreter_2.jinja` (v2). Returns raw interpreter output. |
| `hash_md5` | Computes MD5 hash of the input string. Returns `{tool, query, output}`. |
| `slugify` | Applies HA's `slugify()` filter to the input string. Returns `{tool, query, output}`. |

**Default tool:** `library`

---

## Input Fields

| Field | Required | Description |
|---|---|---|
| `tool` | Yes | Tool to invoke — one of `library`, `library_2`, `hash_md5`, `slugify` |
| `query` | No | Input string or Library command syntax (`~COMMANDS~`) |
| `caller_token` | No | Opaque pass-through token for correlation. Not interpreted. |

---

## Library Command Syntax

The `library` and `library_2` tools accept ZenOS Library Command Syntax — tilde-delimited domain dispatchers such as `~SECURITY~`, `~MEDIA~`, `~ELECTRICAL~`. These are defined in `command_interpreter.jinja` and `command_interpreter_2.jinja`.

Kung Fu components register their library command via the `command` field in their Dojo drawer. The Ninja Summarizer calls the Library automatically before building the monk prompt — the output lands in `library_console` in the review data.

---

## Response Format

All tools return a mapping with at least `output` and `caller_token`:

```json
{
  "tool": "hash_md5",
  "query": "test",
  "output": "098f6bcd4621d373cade4e832627b4f6",
  "caller_token": ""
}
```

`library` tool includes both `query` and `output`:

```json
{
  "query": "~SECURITY~",
  "output": { ... },
  "caller_token": ""
}
```

---

## Usage Examples

### Hash a string

```yaml
tool: hash_md5
query: "my-string-to-hash"
```

### Slugify a name

```yaml
tool: slugify
query: "Security Manager"
# output: "security_manager"
```

### Run a Library command

```yaml
tool: library
query: "~MEDIA~"
```

### Run an on-demand Ninja library command

The Ninja Summarizer calls this automatically using the component's `command` field from the Dojo drawer. You can also call it manually for testing:

```yaml
tool: library
query: "~SECURITY~"
```

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `command_interpreter.jinja` | v1 Library command dispatch engine |
| `command_interpreter_2.jinja` | v2 Library command dispatch engine |
| HA `md5` filter | MD5 hash computation |
| HA `slugify()` filter | String slugification |
