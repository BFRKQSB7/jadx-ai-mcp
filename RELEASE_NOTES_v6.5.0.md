## Release Notes: JADX AI MCP v6.5.0

A feature release: Kotlin dumps get much smaller, and the suite can now be driven from any
OpenAI-compatible endpoint.

Huge thanks to **[@pll177](https://github.com/pll177)**, **[@pateldarsh20](https://github.com/pateldarsh20)**,
**[@anotherAwesomeKitten](https://github.com/anotherAwesomeKitten)** and **[@b4shful](https://github.com/b4shful)**
for the reports and proposals behind this release.

> **One behaviour change, up front.** Tools that return decompiled source now strip Kotlin
> compiler metadata by default, so a Kotlin class comes back 25-50% smaller. Every one of
> those tools takes `slim=False` and returns byte-for-byte what it returned before, so an
> existing workflow can be restored with a single argument.

---
### Kotlin metadata slimming (#70, #122)

Kotlin stamps every compiled class with `@Metadata(d1 = {...}, d2 = {...})`, an encoded table
of every name, signature and type in the class, and coroutines classes with
`@DebugMetadata(...)` on top. JADX prints both into the decompiled source. They are compiler
IR rather than code - unreadable to a human and useless to a model - yet they are a large
share of a dump: measured across the 1270 Kotlin classes of a real APK they were **30% of all
bytes**, and 46% for `kotlin.Pair` alone, whose single metadata line was 1613 of the file's
3485 characters. Every one of those bytes lands in an agent's context window.

Tools that return source now replace each annotation in place with `/*[slimmed: Kotlin metadata]*/`:

| Tool | Parameter |
| --- | --- |
| `get_class_source` | `slim=True` |
| `get_method_by_name` | `slim=True` |
| `fetch_current_class` | `slim=True` |
| `get_main_activity_class` | `slim=True` |
| `get_main_application_classes_code` | `slim=True` |

```python
source = await get_class_source("com.example.Feature")            # slim, the default
numbered = await get_class_source("com.example.Feature", with_line_numbers=True)
raw = await get_class_source("com.example.Feature", slim=False)   # the previous behaviour
```

Design notes worth knowing:

- **The line count never changes**, which is what makes this safe to turn on by default.
  `add_comment(line=N)` is resolved by the plugin against the same decompiled string split on
  newlines, so a dropped line would silently retarget a comment. Verified live: a number read
  from a slimmed numbered dump landed on exactly that line inside JADX.
- The scan is string- and paren-aware, not a regex - the payload is full of quotes, braces and
  descriptors like `(I)V`. Checked over 3078 real decompiled sources: 0 line-count changes and
  0 surviving annotation calls.
- Real annotations (`@Override`, `@Nullable`, ...) and `/* JADX INFO: ... */` notes are kept on
  purpose: the `JADX INFO` lines carry provenance (the original `.kt` file, the source dex).
- Java dumps are unaffected, and the work happens in the server, so **this needs no plugin
  update to take effect** - installing the new server is enough.

### An example client for OpenAI-compatible endpoints (#71)

The README only covered three MCP-aware desktop apps. Everything else speaks
`POST /v1/chat/completions` - OpenRouter, LiteLLM, vLLM, Ollama's OpenAI layer, OmniRoute - and
had no documented way in. `examples/openai_compatible_client.py` closes that gap: it connects
to a running `--http` server, converts the MCP tool list into OpenAI function-calling tools
(34 of them), and runs the call -> execute -> feed-back loop until the model answers.

```bash
uv run jadx_mcp_server.py --http --host 127.0.0.1 --port 9000     # terminal 1
pip install -r examples/requirements.txt

OPENAI_BASE_URL=http://127.0.0.1:11434/v1 OPENAI_API_KEY=ollama OPENAI_MODEL=llama3.1 \
  python examples/openai_compatible_client.py --prompt "What does com.example.MainActivity do?"
```

- `--base-url` / `--api-key` / `--model` (or the `OPENAI_*` environment variables) point it at
  any provider; `--list-tools` prints the offered schema without calling a model at all.
- `--max-steps` and `--max-tool-chars` bound a session, so one large class dump cannot blow a
  small local model's context.
- **The server gains no dependency.** The OpenAI SDK lives in `examples/requirements.txt`;
  the example still imports cleanly without it, and the conversion contract is covered by
  tests so CI does not need the SDK.

### Packaging: the installed package actually works now (server #67)

`pyproject.toml` listed only `py-modules = ["jadx_mcp_server"]`, so the wheel shipped the entry
point without `src/` and every installed copy failed with `ModuleNotFoundError: No module named
'src'`. The package now ships every module under `src/`, declares its build system explicitly,
and a CI job builds the wheel, verifies that each `src/**/*.py` is inside it, installs it into a
clean environment and runs the tests.

> **If you use the release zip:** it is rebuilt for this version. The v6.4.1 zip shipped a
> `class_tools.py` from May and had no `comment_tools.py` at all, so the comment tools were
> missing for zip users. That is fixed here.

### Dependencies

- **jackson-databind 2.18.12**, up from 2.17.2, covering the open advisory set. Nothing in the
  plugin touches jackson directly - the exposure is Javalin's JSON mapping - so this was
  validated with a runtime JSON round-trip on the shaded jar, not just a compile.

### Documentation

- Installation paths corrected for the split server repository, and the tree listing in the
  README now matches what is actually shipped (#119).
- API reference documents `slim` on all five tools, plus a dedicated slimming section.
- The Python reference gained `src/server/slim.py` and `examples/`; its tool count said 32 and
  now says 34.
- New troubleshooting section for running the server inside WSL with JADX-GUI on Windows
  (`--jadx-host`, and why `127.0.0.1` cannot work there).

### Issues Closed

| Issue | Reporter | Summary |
| --- | --- | --- |
| [server #70](https://github.com/zinja-coder/jadx-mcp-server/issues/70) | @pll177 | Strip Kotlin `@Metadata` from class/method source dumps |
| [plugin #122](https://github.com/zinja-coder/jadx-ai-mcp/issues/122) | @pll177 | Same request, tracked on the plugin repo |
| [server #71](https://github.com/zinja-coder/jadx-mcp-server/issues/71) | @pateldarsh20 | Example client for OpenAI-compatible endpoints |
| [server #67](https://github.com/zinja-coder/jadx-mcp-server/issues/67) | @anotherAwesomeKitten | `No module named 'src'` on packaged installs |
| [plugin #119](https://github.com/zinja-coder/jadx-ai-mcp/issues/119) | @b4shful | README/install path inconsistencies |

Still open and tracked: plugin #67 (tool refactor and batch acquisition - the search-scope half
of it already ships), #116 (Windows port errors, environment rather than code), #66 and #65.

### Assets

- `jadx-ai-mcp-6.5.0.jar` - install with `jadx plugins --install-jar jadx-ai-mcp-6.5.0.jar`
- `jadx-mcp-server-6.5.0.zip` - the Python MCP server (unzip and run, see the README)
