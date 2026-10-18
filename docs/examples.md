# Examples

These examples use current tool signatures and response behavior.

## 1) Entry-point mapping

```python
pkg_tree = await get_package_tree()
main = await get_main_activity_class()
app_classes = await get_main_application_classes_names()
```

Then ask the LLM to rank likely auth/network/storage classes.

## 2) Keyword search with scopes

```python
crypto = await search_classes_by_keyword(
    search_term="cipher",
    search_in="method,code",
    count=50,
)
```

```python
comments = await search_classes_by_keyword(
    search_term="todo",
    search_in="comment",
    count=20,
)
```

## 3) Method lookup in known class

```python
method = await get_method_by_name(
    class_name="com.example.auth.AuthManager",
    method_name="login",
    method_signature="(Ljava/lang/String;Ljava/lang/String;)Z",
)
```

## 4) Xref tracing

```python
refs = await get_xrefs_to_method(
    class_name="com.example.auth.AuthManager",
    method_name="login",
    count=100,
)
```

`refs` is paginated with `items` containing `class` and `method`.

## 5) Resource analysis

```python
manifest = await get_android_manifest()
exported_activities = await get_manifest_component("activity", only_exported=True)
strings = await get_strings(offset=0, count=200)
```

## 6) Refactor flow

```python
await rename_class("a.b.c", "CryptoHelper")
await rename_field("com.example.Config", "a", "apiBaseUrl")
await rename_variable("com.example.Auth", "login", "v0", "username")
```

Method rename (current signature):

```python
await rename_method("a", "initialize", class_name="com.example.MainActivity")
# Better disambiguation if overloaded:
await rename_method(
    "a",
    "initialize",
    class_name="com.example.MainActivity",
    method_signature="(Landroid/content/Context;)V",
)
```

## 7) Debug snapshot

```python
stack = await debug_get_stack_frames()
threads = await debug_get_threads()
variables = await debug_get_variables()
```

Use these together for runtime flow analysis while paused.

## 8) Cache maintenance

```python
stats_before = await get_cache_stats()
cleared = await clear_cache()
stats_after = await get_cache_stats()
```

Useful after switching APKs or after bulk decompilation requests.

## 9) Kotlin metadata slimming

Tools that return source strip the Kotlin compiler annotations by default:

```python
# default: @Metadata / @DebugMetadata become /*[slimmed: Kotlin metadata]*/,
# which is 25-50% fewer bytes on a Kotlin class
source = await get_class_source("com.example.Feature")

# numbering is unchanged, so comment_line[0] is the number add_comment wants
numbered = await get_class_source("com.example.Feature", with_line_numbers=True)
comment_line = [line for line in numbered["response"].split("\n") if "onCreate" in line]

# the untouched dump, when you want the metadata itself
raw = await get_class_source("com.example.Feature", slim=False)
```

The line count is never altered, so a number read from the numbered dump addresses the same
line in `add_comment(..., line=N)`. Java dumps come back unchanged, and `@Override`-style
annotations plus `/* JADX INFO: ... */` notes are kept.

## 10) Driving the server from an OpenAI-compatible model

`examples/openai_compatible_client.py` connects to a running `--http` server, exposes its MCP
tools to the model as OpenAI function tools, and runs the call → execute → feed-back loop, so
OpenRouter / LiteLLM / vLLM / Ollama / OmniRoute clients can use the suite as well:

```bash
# terminal 1
uv run jadx_mcp_server.py --http --host 127.0.0.1 --port 9000

# the example needs the openai SDK; the server itself does not
pip install -r examples/requirements.txt

# any provider: --base-url / --api-key / --model, or the OPENAI_* env vars
OPENAI_BASE_URL=http://127.0.0.1:11434/v1 OPENAI_API_KEY=ollama OPENAI_MODEL=llama3.1 \
  python examples/openai_compatible_client.py --prompt "What does com.example.MainActivity do?"

# print the tool schema the model would be offered, without calling a model
python examples/openai_compatible_client.py --list-tools
```

`--max-steps` caps how many tool rounds one question may take and `--max-tool-chars`
truncates a single result, so one large class dump cannot blow a small local model's context.
