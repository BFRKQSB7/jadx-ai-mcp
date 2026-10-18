# Python Module Reference

This page documents the current Python MCP server modules and behavior.

## `jadx_mcp_server.py`

Main MCP entry point:

- Initializes `FastMCP`
- Registers all tools from `src/server/tools/*`
- Parses CLI args:
  - `--http`
  - `--host`
  - `--port`
  - `--jadx-host`
  - `--jadx-port`
- Runs connectivity health check
- Starts stdio or HTTP transport

### Current registered tool count

`34` tools.

## `src/server/config.py`

Responsibilities:

- Tracks `JADX_HOST`, `JADX_PORT`, `JADX_HTTP_BASE`
- Performs HTTP requests to plugin routes via `httpx`
- Uses `trust_env=False`
- Handles:
  - HTTP status errors
  - timeouts
  - connect errors
  - text fallback

Important behavior:

- If plugin returns non-JSON text, helper returns:
  - `{"response": "<text>"}`

## `src/PaginationUtils.py`

Provides generic pagination wrapper:

- Parameter validation (`offset`, `count`)
- Endpoint request with additional params
- Standardized result:

```json
{
  "type": "paginated-list",
  "items": [],
  "pagination": {
    "total": 0,
    "offset": 0,
    "limit": 0,
    "count": 0,
    "has_more": false
  }
}
```

## `src/server/slim.py`

Strips Kotlin compiler metadata from decompiled source before it reaches the client:

- `slim_source(code)` removes `@Metadata(...)` / `@DebugMetadata(...)` calls with a
  string- and paren-aware scan (the payload is full of quotes, braces and descriptors like
  `(I)V`, so a regex cannot match it), replacing each call in place with
  `/*[slimmed: Kotlin metadata]*/`
- `slim_payload(value, keys=("response", "content", "code"))` applies that to the plugin
  responses the source-returning tools read, including the paginated class lists
- If a pass would change the line count it returns the original text: `add_comment(line=N)`
  is resolved by the plugin against the same decompiled string split on newlines, so a
  shifted line would silently comment the wrong code

Used by `get_class_source`, `fetch_current_class`, `get_method_by_name`,
`get_main_activity_class` and `get_main_application_classes_code`, each of which exposes
`slim: bool = True`. Java dumps are unaffected, and `@Override`-style annotations plus
`/* JADX INFO: ... */` notes are kept.

## `examples/`

`examples/openai_compatible_client.py` is a reference client for any OpenAI-compatible
`/v1/chat/completions` endpoint (OpenRouter, LiteLLM, vLLM, Ollama, OmniRoute, …): it
converts the MCP tool list into OpenAI function tools and runs the tool-calling loop. It is
never imported by the server, and its only extra dependency (the OpenAI SDK) lives in
`examples/requirements.txt`.

## Tool Modules

### `src/server/tools/class_tools.py`

Class and project structure tools:

- `fetch_current_class`
- `get_selected_text`
- `get_class_source`
- `get_all_classes`
- `get_methods_of_class`
- `get_fields_of_class`
- `get_smali_of_class`
- `get_main_application_classes_names`
- `get_main_application_classes_code`
- `get_main_activity_class`
- `get_package_tree`
- `get_cache_stats`
- `clear_cache`

### `src/server/tools/search_tools.py`

Search tools:

- `get_method_by_name(class_name, method_name, method_signature=None, slim=True)`
- `search_method_by_name(method_name)`
- `search_classes_by_keyword(search_term, package="", search_in="code", offset=0, count=20)`

Includes search progress polling (`/search-progress`) and optional MCP progress reporting.

### `src/server/tools/resource_tools.py`

Resource/manifest tools:

- `get_android_manifest`
- `get_manifest_component(component_type, only_exported=False)`
- `get_strings`
- `get_all_resource_file_names`
- `get_resource_file`

### `src/server/tools/refactor_tools.py`

Refactor tools:

- `rename_class(class_name, new_name)`
- `rename_method(method_name, new_name, class_name=None, method_signature=None)`
- `rename_field(class_name, field_name, new_name)`
- `rename_package(old_package_name, new_package_name)`
- `rename_variable(class_name, method_name, variable_name, new_name, reg=None, ssa=None)`

### `src/server/tools/comment_tools.py`

Comment tools:

- `add_comment(class_name, comment, method_name=None, method_signature=None, field_name=None, line=None, style="LINE")`
- `list_comments(class_name="")`

### `src/server/tools/xrefs_tools.py`

Xref tools:

- `get_xrefs_to_class`
- `get_xrefs_to_method`
- `get_xrefs_to_field`

### `src/server/tools/debug_tools.py`

Debug tools:

- `debug_get_stack_frames`
- `debug_get_threads`
- `debug_get_variables`

## Response Caveats

Some Java routes still emit plain text (`ctx.result`) rather than JSON.  
For those routes, MCP consumers should handle:

```json
{ "response": "..." }
```

This is expected for several current endpoints (for example method/class listing style routes).

## Versioning Note

Python requirement is aligned at `>=3.10` for both script execution and package metadata.
