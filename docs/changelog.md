# Changelog

Starting from v6.0.0 all notable changes to JADX-AI-MCP will be documented here.

## v6.5.0 (2026-10)

MCP server (`jadx-mcp-server`):

- Kotlin metadata slimming: tools that return source now strip `@Metadata` /
  `@DebugMetadata` before answering (`slim=False` returns the raw dump). Measured 30% fewer
  bytes across the Kotlin classes of a real APK, up to 46% on metadata-heavy ones, with the
  line count untouched so `with_line_numbers` and `add_comment(line=N)` keep working.
- Added `examples/openai_compatible_client.py`, a reference client for any OpenAI-compatible
  endpoint (OpenRouter, LiteLLM, vLLM, Ollama, OmniRoute, ...). The server itself gains no
  dependency; the OpenAI SDK lives in `examples/requirements.txt`.
- Fixed packaging: the wheel now ships every module under `src/` (the `No module named 'src'`
  failure), with a CI job that builds the wheel, checks the module list, installs it in a
  clean environment and runs the tests.
- Added tests for the tool schemas, the slimming rules, the line-number contract and the
  example client.

Plugin (`jadx-ai-mcp`):

- jackson-databind pinned to 2.18.12, covering the open advisory set.
- Documentation corrected for the split server repository: clone path, run directory, tree
  listing and `--directory` in client configs.

## 2026-07 Documentation Sync

- Updated ReadTheDocs pages to match current MCP tool signatures and route behavior.
- Corrected refactor/search signature docs (`rename_method`, `search_classes_by_keyword`).
- Reworked API, installation, architecture, examples, Python and Java references.
- Clarified Python requirement differences between script mode and packaged install mode.

