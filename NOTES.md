# MCP Notes — Introduction to Model Context Protocol

Notes from [Anthropic's "Introduction to Model Context Protocol" course](https://anthropic.skilljar.com/introduction-to-model-context-protocol),
cross-referenced against the working example in this repo (`main.py`, `mcp_client.py`, `mcp_server.py`, `core/`).

---

## 1. Introduction

### Introducing MCP
**MCP (Model Context Protocol)** — a communication layer that gives Claude context and tools without developers having to write tedious integration code themselves.

**Core architecture:** an MCP *client* connects to an MCP *server*. The server contains tools, resources, and prompts as its internal components.

**Problem it solves:** the traditional approach requires a developer to manually author tool schemas and function implementations for every service they want to integrate (e.g. hand-writing tool wrappers for the whole GitHub API). This is a heavy maintenance burden for any complex service with lots of functionality.

**MCP's solution:** shift tool definition and execution off the developer's own server and onto a dedicated MCP server. The MCP server acts as the interface to the outside service, pre-packaging its functionality into ready-made tools.

**Key benefit:** developers no longer need to write or maintain tool schemas/implementations themselves — someone else (often the service provider) authors the tools once, packaged inside an MCP server, and anyone can plug into it.

**Common questions:**
- *Who authors MCP servers?* Anyone can, but service providers often ship official implementations.
- *How is this different from calling the API directly?* It saves developer time — you get pre-built tool schemas and functions instead of writing them yourself.
- *How does this relate to "tool use"?* MCP and tool use are complementary, not the same thing. MCP is about **who does the work of creating the tools**, not a replacement for the tool-use mechanism itself.

### MCP Clients
The **MCP client** is the communication interface between your application's server and the MCP server — it's what gives your app access to the MCP server's tools.

**Transport agnostic:** client and server can communicate over multiple transports (stdio, HTTP, WebSockets, etc). The common local setup: both processes on the same machine, talking over stdin/stdout.

**Message types (per the MCP spec):**
- **list tools** request/result — client asks the server what tools it has; server replies with the list.
- **call tool** request/result — client asks the server to run a specific tool with arguments; server replies with the execution result.

**Typical end-to-end flow:**
1. User sends a query to the app.
2. App's server asks its MCP client for available tools.
3. MCP client sends a *list tools* request to the MCP server.
4. MCP server returns its tool list.
5. App's server sends the query + tool list to Claude.
6. Claude decides to use a tool and requests its execution.
7. App's server asks the MCP client to run that tool.
8. MCP client sends a *call tool* request to the MCP server.
9. MCP server actually executes the tool (e.g. a GitHub API call).
10. The result flows back up the chain to Claude.
11. Claude formulates the final response for the user.

**Key point:** the MCP client is only an intermediary — it never executes tools itself. It just facilitates communication between your app and the MCP server, which is the one that actually runs the tools.

---

## 2. Hands-on with MCP Servers

### Project Setup
This course's project is a CLI chatbot implementing **both** an MCP client and an MCP server in the same codebase, for learning purposes — [main.py](main.py) wires up the client side, [mcp_server.py](mcp_server.py) is the server. (In the real world, most projects implement only a client *or* a server, not both — this repo does both intentionally, for the exercise.)

- Document system: fake documents held in memory only, no persistence — see the `docs` dict in [mcp_server.py:8-15](mcp_server.py).
- Server tools: two are implemented — reading a document's contents, and editing a document's contents.
- Setup: `.env` needs the API key configured (`ANTHROPIC_API_KEY`, `CLAUDE_MODEL`), dependencies installed via `uv sync` or `pip install`.
- Run with `uv run main.py` (with uv) or `python main.py` (without).
- Verify it's working: the chat prompt appears, and it can answer a basic query like "what's one plus one".

### Defining Tools with MCP
The MCP Python SDK simplifies tool creation versus writing raw JSON schemas by hand.

**Tool definition syntax:** `@mcp.tool` decorator + a function with typed parameters + `pydantic.Field(description=...)` on each parameter.

- **`read_doc_contents`** — takes a `doc_id` string, returns the document's content from the `docs` dict, raises `ValueError` if the doc isn't found.
- **`edit_document`** — takes `doc_id`, `old_string`, `new_string`; performs a find/replace on the document's content; also validates the doc exists first.

```python
@mcp.tool(
    name="read_doc_contents",
    description="Read the contents of a document and return it as a string."
)
def read_document(doc_id: str = Field(description="Id of the document to read.")):
    if doc_id not in docs:
        raise ValueError(f"Doc with id {doc_id} not found")
    return docs[doc_id]
```
([mcp_server.py:17-27](mcp_server.py), `edit_document` at [mcp_server.py:29-41](mcp_server.py))

**Why the SDK helps:** it auto-generates JSON schemas from the decorated function's type hints, so the server can be stood up in essentially one line, with no manual schema-writing.

**Implementation pattern:** decorator → function definition → typed parameters → existence validation → core logic.

### The Server Inspector
**MCP Inspector** — an in-browser debugger for testing an MCP server standalone, without needing to connect it to a real client application.

- **Access:** run `mcp dev [server_file.py]` in a terminal with the Python environment active → it opens the server on a port → visit the localhost address it gives you. In this repo: `uv run mcp dev mcp_server.py`.
- **Interface:** a left sidebar with a *Connect* button; a top nav with Resources / Prompts / Tools sections; the Tools section lists available tools, and clicking one opens a right-hand panel for manual testing.
- **Testing flow:** select a tool → fill in the required parameters (e.g. a document ID) → click *Run Tool* → check the output/success message.
- The Inspector is under active development, so the UI may shift, but the core workflow (connect → list → invoke → inspect) stays the same.
- This is the fastest feedback loop while building a server — verify each tool/resource/prompt in isolation before ever wiring an LLM to it.

---

## 3. Connecting with MCP Clients

### Implementing a Client
**MCP Client** — a wrapper class around a `ClientSession`, responsible for connecting to the MCP server and cleaning up the connection afterward.

- **Client session:** the actual connection to the MCP server, provided by the MCP Python SDK; it needs explicit cleanup when shutting down.
- **Resource cleanup:** handled through `connect` / `cleanup` / `__aenter__` / `__aexit__` methods.
- **Purpose of the client class:** exposes the MCP server's functionality to the rest of the codebase — it's the interface between your application code and the server.

Key functions:
```python
async def list_tools(self):
    result = await self.session().list_tools()
    return result.tools

async def call_tool(self, tool_name, tool_input):
    return await self.session().call_tool(tool_name, tool_input)
```
(matches this repo's [mcp_client.py:46-53](mcp_client.py))

**Implementation flow:**
1. App requests the tool list, to hand to Claude.
2. Client calls `list_tools()` to fetch what the server offers.
3. Claude picks a tool and supplies parameters.
4. Client calls `call_tool()` to actually run it on the server.
5. Results flow back to Claude.

**Testing:** run `mcp_client.py` directly with a small test harness to confirm the connection and tool listing work — see the `main()` test function at the bottom of [mcp_client.py:85-97](mcp_client.py).

**Once wired up:** you can run the CLI and have Claude use the tools naturally, e.g. asking "what is the contents of the report.pdf document".

**Common practice:** wrap the client session inside a larger class (like `MCPClient` here) rather than using the raw session directly — better resource management.

### Defining Resources
**Resources** — an MCP server feature that exposes data to clients for **read-only** access.

**Two resource types:**
- **Direct/static** — a fixed URI, e.g. `docs://documents`.
- **Templated** — a parameterized URI with a wildcard segment, e.g. `docs://documents/{doc_id}`.

**Resource flow:**
1. Client sends a *read resource* request with a URI.
2. Server matches the URI to the corresponding resource function.
3. Server executes the function and returns the result.
4. Client receives the data via a *read resource result* message.

**Implementation:**
- Use the `@mcp.resource` decorator.
- Define the URI (a route-like address).
- Set a MIME type (`application/json`, `text/plain`, etc.) as a hint to the client about how to deserialize the returned data.
- For templated resources, URI parameters become keyword arguments on the function.
- The SDK auto-serializes the return value to a string.

```python
@mcp.resource("docs://documents", mime_type="application/json")
def list_docs() -> list[str]:
    return list(docs.keys())

@mcp.resource("docs://documents/{doc_id}", mime_type="text/plain")
def fetch_doc(doc_id: str) -> str:
    if doc_id not in docs:
        raise ValueError(f"Doc with id {doc_id} not found")
    return docs[doc_id]
```
([mcp_server.py:43-57](mcp_server.py))

**Common pattern:** one resource per distinct read operation — e.g. a "list all documents" resource is separate from a "fetch one document's contents" resource.

### Accessing Resources
The client's `read_resource` method takes a URI, requests that resource from the server, and parses the response:

```python
async def read_resource(self, uri: str):
    result = await self.session().read_resource(AnyUrl(uri))
    resource = result.contents[0]
    if resource.mimeType == "application/json":
        return json.loads(resource.text)
    return resource.text
```
(matches [mcp_client.py:63-70](mcp_client.py); key dependencies: the `json` module and `pydantic.AnyUrl`)

- Accesses `result.contents[0]` — the first item in the returned contents list.
- Checks the resource's `mimeType` to decide how to deserialize it — JSON gets `json.loads`'d, otherwise treated as plain text.

**Resource integration in the app:** resource-reading client functions get called by other application components. In this repo, resources back the `@mention` feature — e.g. typing `@report.pdf` triggers `_extract_resources` to fetch the doc list resource, match the mention, pull that doc's content resource, and splice it straight into the prompt as context ([cli_chat.py:35-49](core/cli_chat.py)) — **eliminating the need for a tool call** just to read document contents during chat. (Some course UIs implement this as document selection via arrow keys + space instead of `@mentions` — same underlying resource mechanism either way.)

### Defining Prompts
**Prompts** — pre-written, tested instructions that an MCP server exposes to clients for specialized tasks.

- Servers define high-quality prompts tailored to their own domain.
- Clients access them via something like a slash command (e.g. `/format`).
- This is an alternative to leaving prompt-writing entirely up to end users.

**Implementation pattern:**
- Use the `@mcp.prompt` decorator with `name` and `description`.
- The function receives arguments (e.g. a document ID).
- It returns a list of messages (user/assistant format).
- Those messages get sent directly to Claude.

```python
@mcp.prompt(name="format", description="Rewrites document in markdown")
def format_document(doc_id: str) -> list[base.Message]:
    return [base.UserMessage(prompt_text)]
```
(matches [mcp_server.py:59-82](mcp_server.py), `format_document`)

**Key benefit:** the server author encodes their own prompt-engineering expertise once, tested and optimized, rather than leaving prompt quality up to whoever happens to be using the client.

**Workflow:** user types `/format` → selects a document → server returns the specialized prompt → client sends it to Claude → Claude uses tools to read, reformat, and save the document.

### Prompts in the Client
Client-side functions for working with prompts:

```python
async def list_prompts(self):
    result = await self.session().list_prompts()
    return result.prompts

async def get_prompt(self, prompt_name, arguments):
    result = await self.session().get_prompt(prompt_name, arguments)
    return result.messages
```
(matches [mcp_client.py:55-61](mcp_client.py) — note: the actual SDK field is `result.prompts`/`result.messages`, plural)

**Prompt workflow:** client requests a prompt by name → passes arguments as keyword parameters → the server interpolates those arguments into its prompt template → returns formatted messages ready to feed to the model.

**Arguments flow:** client-supplied arguments become the prompt function's keyword arguments, which get interpolated into the prompt text (e.g. a `doc_id` argument gets inserted into the template).

**CLI usage:** `/format` command → pick a document → the prompt (with that doc's ID baked in) gets sent to Claude → Claude uses its tools to fetch the document and return a formatted result. In this repo: `/format deposition.md` → [cli_chat.py:51-63](core/cli_chat.py) calls `get_prompt`, converts the returned messages via `convert_prompt_messages_to_message_params`, and appends them to the conversation.

**Key concept:** prompts are server-defined templates that clients invoke with parameters — reusable AI instructions with dynamic content insertion, controlled by the *user*, not the model.

---

## 4. Assessment & Wrap Up

### MCP Review — the three primitives at a glance

| Primitive | Who controls it | Purpose | Real-world example |
|---|---|---|---|
| **Tools** | **Model** — Claude decides when to execute them | Add capabilities to Claude | JavaScript execution for calculations |
| **Resources** | **App** — application code decides when to fetch data | Get data into the app, for UI display or prompt augmentation | Autocomplete options, document listings from Google Drive |
| **Prompts** | **User** — triggered by explicit user action (button click, slash command) | Predefined workflows | Chat starter buttons in the Claude interface |

**How to decide which primitive to use:**
- Need Claude to be able to *do* something → implement a **tool**.
- Need to get data *into* the app (for context or display) → use a **resource**.
- Need a predefined, user-triggered *workflow* → create a **prompt**.

---

## Debugging notes from building this project

Bugs actually hit and fixed while running this repo's code — worth remembering since the SDK fails on some of these silently or with a confusing traceback:

1. **Missing `await`** — every `ClientSession` method (`list_tools`, `call_tool`, `get_prompt`, `read_resource`, `initialize`) is a coroutine. Forgetting `await` gives a coroutine object instead of a result (`AttributeError: 'coroutine' object has no attribute ...`).
2. **Wrong result field name** — `GetPromptResult` has `.messages` (plural), not `.message`. Check the actual Pydantic model fields rather than guessing.
3. **Silent duplicate registration** — stacking two `@mcp.prompt(name="format", ...)` (or `@mcp.tool`) decorators under the same name doesn't raise an error; the second registration just silently overwrites the first in FastMCP's internal dict.
4. **PATH issues with `uv`/`mcp` CLI** — installing via `pip3 install uv` or `pip3 install mcp[cli]` can put the executable somewhere not on `$PATH` (e.g. `~/Library/Python/3.9/bin`). Prefer `brew install uv`, and always run project tools via `uv run <cmd>` so they resolve inside the project's own `.venv` regardless of global PATH state.
