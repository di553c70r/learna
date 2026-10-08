# MCP, MCP Client, MCPO, OpenAPI & Swagger UI — Notes

## 1. Big Picture

The overall architecture we built is:

```text
                         ┌──────────────┐
                         │     User     │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │     LLM      │
                         └──────┬───────┘
                                │
                         AI Application
                                │
                                │ HTTP
                                ▼
                         ┌──────────────┐
                         │     MCPO     │
                         │   :1339      │
                         └──────┬───────┘
                                │
                         MCP / JSON-RPC
                                │
                                ▼
                      ┌───────────────────┐
                      │   MCP Server      │
                      │  simple_mcp.py    │
                      │                   │
                      │  add()            │
                      │  greet()          │
                      │  lookup_cwe()     │
                      └────────┬──────────┘
                               │
                              HTTPS
                               │
                               ▼
                      ┌───────────────────┐
                      │   MITRE CWE API   │
                      └───────────────────┘
```

The important components are:

* **LLM** — reasons about the user's request.
* **MCP Client** — communicates with an MCP server.
* **MCP Server** — exposes tools.
* **MCPO** — acts as a bridge/wrapper, exposing MCP tools through HTTP/OpenAPI.
* **OpenAPI** — describes the HTTP API.
* **Swagger UI** — provides a visual interface for the OpenAPI API.

---

# 2. What is MCP?

**MCP = Model Context Protocol**

MCP is a protocol that allows AI applications to communicate with external tools, data sources, and services in a standardized way.

Instead of hard-coding every tool directly into an AI application, an AI application can connect to an MCP server and discover what tools are available.

For example:

```text
MCP Server
├── add()
├── greet()
└── lookup_cwe()
```

The AI can discover these tools and determine:

```text
add
├── a: integer
└── b: integer

greet
└── name: string

lookup_cwe
└── cwe_id: integer
```

---

# 3. Basic MCP Server

Our first MCP server looked like this:

```python
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel

mcp = FastMCP("my-first-mcp")


class AddResponse(BaseModel):
    """Structured response for the add tool."""
    result: int


@mcp.tool()
def add(a: int, b: int) -> AddResponse:
    """Add two numbers and return the result."""
    return AddResponse(result=a + b)


@mcp.tool()
def greet(name: str) -> str:
    """Greet a user by name."""
    return f"Hello, {name}!"


if __name__ == "__main__":
    mcp.run()
```

---

# 4. `FastMCP`

```python
from mcp.server.fastmcp import FastMCP
```

`FastMCP` makes it easy to create an MCP server.

We create the server with:

```python
mcp = FastMCP("my-first-mcp")
```

This creates an MCP server named:

```text
my-first-mcp
```

---

# 5. `@mcp.tool()`

The decorator:

```python
@mcp.tool()
```

turns a normal Python function into an MCP tool.

For example:

```python
@mcp.tool()
def add(a: int, b: int) -> AddResponse:
    ...
```

means:

> Expose `add()` as a tool that an MCP client can discover and call.

Similarly:

```python
@mcp.tool()
def greet(name: str) -> str:
    ...
```

creates a second tool.

---

# 6. Type Hints Become Tool Schema

This:

```python
def add(a: int, b: int)
```

tells the MCP framework that:

```text
a → integer
b → integer
```

This:

```python
def greet(name: str)
```

tells it:

```text
name → string
```

This information becomes part of the tool's schema.

The AI/client can therefore discover how the tool should be called.

---

# 7. Pydantic Structured Response

We defined:

```python
class AddResponse(BaseModel):
    result: int
```

This describes a structured response.

Instead of simply returning:

```python
30
```

we return:

```python
AddResponse(result=30)
```

Conceptually:

```json
{
  "result": 30
}
```

Pydantic helps define and validate structured data.

---

# 8. MCP Server and MCP Client

An important distinction:

> The MCP server provides tools. The MCP client connects to the server and communicates with it.

Architecture:

```text
MCP Client
     │
     │ MCP
     ▼
MCP Server
     │
     ├── add()
     ├── greet()
     └── lookup_cwe()
```

The LLM and MCP client are **not necessarily the same thing**.

Usually:

```text
LLM
 │
 │ decides which tool to use
 ▼
AI Application / MCP Client
 │
 │ communicates using MCP
 ▼
MCP Server
```

The MCP client is typically part of the host AI application.

---

# 9. Our Notebook as an MCP Client

We manually created an MCP client using:

```python
import subprocess
import json
```

The notebook started the server:

```python
proc = subprocess.Popen(
    ["python", f"{DEMO_DIR}/simple_mcp.py"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
    text=True,
)
```

This is approximately equivalent to running:

```bash
python simple_mcp.py
```

but the Python notebook keeps control of the process.

---

# 10. stdin and stdout

The MCP server communicates through standard input/output.

```text
Client ─────────────► Server
          stdin

Client ◄───────────── Server
          stdout
```

Therefore:

```python
stdin=subprocess.PIPE
```

allows the client to send data to the server.

And:

```python
stdout=subprocess.PIPE
```

allows the client to receive data from the server.

---

# 11. JSON-RPC

MCP communication uses JSON-RPC-style messages.

Example:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "initialize",
  "params": {}
}
```

Important fields:

```text
jsonrpc
    Protocol/message version

id
    Identifies the request

method
    Operation being requested

params
    Arguments for the operation
```

---

# 12. MCP Initialization

The client first sends:

```python
init_request = {
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
        "protocolVersion": "2024-11-05",
        "capabilities": {},
        "clientInfo": {
            "name": "notebook-demo",
            "version": "0.1"
        },
    },
}
```

The client sends it:

```python
proc.stdin.write(json.dumps(init_request) + "\n")
proc.stdin.flush()
```

Then reads the response:

```python
init_response = json.loads(proc.stdout.readline())
```

Conceptually:

```text
CLIENT                         SERVER

initialize ───────────────────►
            ◄────────────────── initialization response
```

---

# 13. `notifications/initialized`

After initialization, the client sends:

```json
{
  "jsonrpc": "2.0",
  "method": "notifications/initialized"
}
```

This is a notification.

Unlike a normal request, it does not have an `id` and does not expect a response.

Conceptually:

```text
Request:

{
    "id": 1,
    "method": "initialize"
}

        ↓

Response:

{
    "id": 1,
    "result": ...
}
```

Whereas:

```text
Notification:

{
    "method": "notifications/initialized"
}

        ↓

No response required
```

---

# 14. Discovering Tools

The client then sends:

```python
tools_request = {
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/list",
    "params": {}
}
```

This means:

> Give me the list of tools available on this MCP server.

The response contains information about:

```text
Tool name
Description
Input schema
Parameters
```

For example:

```text
add
├── a: integer
└── b: integer

greet
└── name: string
```

The important point:

> `tools/list` discovers the tools. It does not execute them.

---

# 15. `tools/list` vs `tools/call`

### `tools/list`

Asks:

> What tools are available?

```text
tools/list
```

### `tools/call`

Asks:

> Execute this particular tool.

Conceptually:

```text
tools/call
    │
    └── add(a=25, b=17)
```

This distinction is important.

Our first notebook example only demonstrated tool discovery.

---

# 16. Why JSON-RPC?

JSON-RPC provides a standardized way to communicate between the client and server.

For example:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

The server returns a response associated with:

```text
id = 2
```

This allows the client to match responses to requests.

---

# 17. Upgraded MCP Server with Logging

We then upgraded the server:

```python
import logging
import sys
import requests

from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel
```

The new server has:

```text
add()
greet()
lookup_cwe()
```

and logging.

---

# 18. Logging Configuration

We used:

```python
logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    stream=sys.stderr,
)
```

This means:

* Log level = `DEBUG`
* Include timestamp
* Include log level
* Include logger name
* Send logs to `stderr`

---

# 19. Why Logs Must Go to stderr

This is extremely important for an MCP server using stdio.

The MCP protocol uses:

```text
stdout
```

for protocol communication.

Therefore, you should avoid doing:

```python
print("debug message")
```

to stdout.

Otherwise:

```text
stdout:

debug message
{"jsonrpc": "2.0", ...}
```

can corrupt the protocol stream.

Instead:

```python
logger.info("debug message")
```

goes to:

```text
stderr
```

So:

```text
MCP protocol
    │
    ▼
 stdout

Logging/debugging
    │
    ▼
 stderr
```

This separation is critical.

---

# 20. Logging Example

For:

```python
logger.info(f"add() called with a={a}, b={b}")
```

you might see:

```text
[INFO] simple_mcp: add() called with a=10, b=20
```

For:

```python
logger.debug(f"add() returning {result}")
```

you might see:

```text
[DEBUG] simple_mcp: add() returning result=30
```

---

# 21. `lookup_cwe()` Tool

We added:

```python
@mcp.tool()
def lookup_cwe(cwe_id: int) -> dict:
```

This exposes an external API through MCP.

The tool constructs:

```python
url = f"https://cwe-api.mitre.org/api/v1/cwe/weakness/{cwe_id}"
```

For:

```text
cwe_id = 79
```

the server requests the corresponding CWE information from the MITRE API.

---

# 22. External API Tool Architecture

The flow becomes:

```text
LLM
 │
 ▼
MCP Client
 │
 │ tools/call
 ▼
MCP Server
 │
 │ lookup_cwe(79)
 ▼
MITRE CWE API
 │
 │ JSON
 ▼
MCP Server
 │
 ▼
MCP Client
 │
 ▼
LLM
```

This demonstrates a very useful MCP pattern:

> An MCP server can wrap external APIs and expose them as AI-callable tools.

The AI doesn't need to understand the underlying REST API.

It only needs to understand:

```text
lookup_cwe(cwe_id)
```

---

# 23. Error Handling

The external API call is wrapped in:

```python
try:
    ...
except requests.exceptions.RequestException as e:
    ...
```

If the API fails, the server logs the exception:

```python
logger.exception(...)
```

and returns structured error information:

```python
{
    "error": "...",
    "details": "..."
}
```

This prevents a simple API failure from unnecessarily crashing the entire MCP server.

---

# 24. Introducing MCPO

MCPO is where the architecture changes.

Instead of manually writing an MCP client that communicates through JSON-RPC, we can put MCPO in front of the MCP server.

Architecture:

```text
HTTP Client / AI Application
            │
            │ HTTP
            ▼
         MCPO
            │
            │ MCP / stdio
            ▼
      MCP Server
```

MCPO acts as a bridge/adapter between an HTTP interface and the MCP server.

---

# 25. Starting MCPO

We used:

```python
proc = subprocess.Popen(
    [
        "mcpo",
        "--port",
        "1339",
        "--",
        "python",
        str(DEMO_DIR / "simple_mcp.py")
    ],
    stdout=subprocess.PIPE,
    stderr=log_handle,
)
```

This is approximately equivalent to:

```bash
mcpo --port 1339 -- python simple_mcp.py
```

---

# 26. What does `--port 1339` mean?

It tells MCPO:

> Listen for HTTP requests on port 1339.

So MCPO becomes available at:

```text
http://localhost:1339
```

---

# 27. What does `--` mean?

The:

```text
--
```

separates MCPO's arguments from the command used to start the MCP server.

Conceptually:

```text
mcpo
├── --port 1339
│
└── MCP server command
      └── python simple_mcp.py
```

So MCPO knows:

> Run this MCP server and expose it through the MCPO HTTP interface.

---

# 28. MCPO Architecture

After starting MCPO:

```text
                    HTTP
                     │
                     ▼
              ┌─────────────┐
              │    MCPO     │
              │   :1339     │
              └──────┬──────┘
                     │
                MCP / stdio
                     │
                     ▼
              ┌─────────────┐
              │ MCP Server  │
              │             │
              │ add()       │
              │ greet()     │
              │ lookup_cwe()│
              └─────────────┘
```

---

# 29. Do We Still Need JSON-RPC?

This is an important distinction.

Without MCPO:

```text
Your code
    │
    │ MCP / JSON-RPC
    ▼
MCP Server
```

Your code manually creates messages such as:

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/list",
  "params": {}
}
```

With MCPO:

```text
Your application
       │
       │ HTTP
       ▼
     MCPO
       │
       │ MCP / JSON-RPC
       ▼
  MCP Server
```

Your application doesn't need to manually construct the MCP JSON-RPC messages.

MCPO handles that side.

Therefore:

> **You don't need to manually deal with JSON-RPC when using MCPO's HTTP interface, but JSON-RPC/MCP communication still exists internally between MCPO and the MCP server.**

MCPO hides that complexity.

---

# 30. HTTP API After MCPO

Once MCPO is running:

```text
http://localhost:1339
```

the MCP tools become accessible through HTTP endpoints.

For example:

```text
POST /add
POST /greet
POST /lookup_cwe
```

Conceptually:

```text
HTTP:

POST /add

{
    "a": 40,
    "b": 20
}
```

MCPO then handles the MCP side and invokes:

```python
add(a=40, b=20)
```

---

# 31. Calling `/add`

We can use Python:

```python
import requests

BASE_URL = "http://localhost:1339"

response = requests.post(
    f"{BASE_URL}/add",
    json={"a": 40, "b": 20}
)

print(response.json())
```

This is now ordinary HTTP.

You are no longer manually constructing:

```json
{
    "jsonrpc": "2.0",
    "method": "tools/call"
}
```

Instead:

```text
POST /add
```

MCPO handles the translation.

---

# 32. Calling `/greet`

Similarly:

```python
response = requests.post(
    f"{BASE_URL}/greet",
    json={"name": "Reverse Engineer"}
)
```

This becomes conceptually:

```text
HTTP
POST /greet
{
    "name": "Reverse Engineer"
}
```

MCPO then invokes:

```python
greet(name="Reverse Engineer")
```

---

# 33. OpenAPI

MCPO also exposes an OpenAPI schema:

```text
http://localhost:1339/openapi.json
```

We can retrieve it using:

```python
schema = requests.get(
    f"{BASE_URL}/openapi.json"
).json()
```

The OpenAPI document describes the HTTP API.

It can tell a client:

```text
Available paths
HTTP methods
Request schemas
Parameters
Responses
Descriptions
```

Conceptually:

```json
{
  "paths": {
    "/add": {
      "post": {}
    },
    "/greet": {
      "post": {}
    },
    "/lookup_cwe": {
      "post": {}
    }
  }
}
```

---

# 34. OpenAPI Is a Description

OpenAPI does not itself execute the tool.

It describes the API.

Think:

```text
OpenAPI
    │
    └── "Here is what my HTTP API looks like."
```

For example:

```text
/add

POST

Request:
{
    "a": integer,
    "b": integer
}
```

---

# 35. Swagger UI

Swagger UI is a visual interface built from the OpenAPI description.

MCPO provides:

```text
http://localhost:1339/docs
```

Open that URL in a browser and you get an interactive API page.

Conceptually:

```text
┌─────────────────────────────────────────┐
│              Swagger UI                 │
│                                         │
│ POST /add                               │
│ Add two numbers                         │
│                                         │
│ POST /greet                             │
│ Greet a user                            │
│                                         │
│ POST /lookup_cwe                        │
│ Retrieve CWE information                │
│                                         │
└─────────────────────────────────────────┘
```

You can inspect the endpoints and interact with them.

---

# 36. Swagger UI vs OpenAPI vs MCPO

Memorize this:

```text
MCPO
 │
 ├── /openapi.json
 │       │
 │       └── Machine-readable API description
 │
 └── /docs
         │
         └── Swagger UI
             Human-friendly interactive interface
```

So:

### MCPO

Creates/exposes the HTTP interface.

### OpenAPI

Describes the HTTP interface.

### Swagger UI

Displays the OpenAPI interface interactively.

---

# 37. The Three Important URLs

After starting MCPO on port 1339:

### Tool endpoint

```text
http://localhost:1339/add
```

This is an actual API endpoint.

### OpenAPI schema

```text
http://localhost:1339/openapi.json
```

This is the machine-readable description of the API.

### Swagger UI

```text
http://localhost:1339/docs
```

This is the interactive visual interface.

---

# 38. Relationship Between Them

```text
                    MCPO
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
    /openapi.json              /docs
          │                     │
          │                     │
    OpenAPI schema         Swagger UI
          │                     │
          └──────────┬──────────┘
                     │
                     ▼
               HTTP endpoints
                     │
          ┌──────────┼──────────┐
          ▼          ▼          ▼
        /add       /greet    /lookup_cwe
```

Swagger UI reads the OpenAPI information and uses it to display the API.

---

# 39. Final Architecture

Putting everything together:

```text
                           USER
                             │
                             ▼
                           LLM
                             │
                             │ decides:
                             │ "Use lookup_cwe(79)"
                             ▼
                    AI Application
                    / MCP Client
                             │
                             │ HTTP
                             ▼
                     ┌───────────────┐
                     │     MCPO      │
                     │    :1339      │
                     └───────┬───────┘
                             │
                       MCP / JSON-RPC
                             │
                             ▼
                  ┌─────────────────────┐
                  │    MCP SERVER       │
                  │   simple_mcp.py     │
                  │                     │
                  │  add()              │
                  │  greet()            │
                  │  lookup_cwe()       │
                  └──────────┬──────────┘
                             │
                             │ HTTPS
                             ▼
                    ┌────────────────┐
                    │ MITRE CWE API  │
                    └────────────────┘


              MCPO also exposes:

              /openapi.json
                     │
                     ▼
               OpenAPI Schema
                     │
                     ▼
                  /docs
                     │
                     ▼
               Swagger UI
```

---

# 40. Most Important Mental Model

Remember these layers:

```text
┌────────────────────────────────────┐
│              LLM                   │
│ "Which tool should I use?"         │
└──────────────────┬─────────────────┘
                   │
                   ▼
┌────────────────────────────────────┐
│        AI Application / Client     │
│                                    │
│ Talks to MCPO using HTTP           │
└──────────────────┬─────────────────┘
                   │
                   │ HTTP
                   ▼
┌────────────────────────────────────┐
│               MCPO                 │
│                                    │
│ HTTP/OpenAPI ↔ MCP                 │
└──────────────────┬─────────────────┘
                   │
                   │ MCP / JSON-RPC
                   ▼
┌────────────────────────────────────┐
│            MCP Server              │
│                                    │
│ add()                              │
│ greet()                            │
│ lookup_cwe()                       │
└──────────────────┬─────────────────┘
                   │
                   │ HTTP
                   ▼
              External APIs
```

And separately:

```text
MCPO
 │
 ├── /docs
 │      └── Swagger UI
 │
 └── /openapi.json
        └── OpenAPI specification
```

---

# 41. Key Takeaways

### MCP Server

Provides tools:

```text
add()
greet()
lookup_cwe()
```

### MCP Client

Communicates with the MCP server.

### LLM

Decides when/which tool should be used.

### JSON-RPC

Message format used for MCP communication.

### MCPO

Acts as a bridge:

```text
HTTP/OpenAPI ↔ MCP
```

### OpenAPI

Machine-readable description of MCPO's HTTP API.

### Swagger UI

Interactive webpage generated from/exposing the OpenAPI API.

### Logging

Should go to:

```text
stderr
```

when using stdio MCP transport, so that logs don't corrupt protocol messages on:

```text
stdout
```

---

# 42. One-Sentence Summary

> **MCP lets an AI application discover and use tools; the MCP server provides those tools, the MCP client communicates with the server, JSON-RPC carries the MCP messages, MCPO hides that MCP complexity behind HTTP/OpenAPI, OpenAPI describes the HTTP API, and Swagger UI provides a visual interface for exploring and testing it.**
