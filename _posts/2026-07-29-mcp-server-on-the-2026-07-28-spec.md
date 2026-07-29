---
title: "Updating Your MCP Server for the 2026-07-28 Spec"
date: 2026-07-29 16:00:00 +0530
categories: [AI, Tutorials]
tags: [mcp, python, llm, anthropic, mcp-2026-07-28]
---

Barely a day after MCP's [first anniversary]({% post_url 2026-07-29-your-first-mcp-server-in-python %}),
the protocol shipped its most significant revision yet: the
[**2026-07-28 specification**](https://blog.modelcontextprotocol.io/posts/2026-07-28/).
It's a bigger jump than a routine version bump, so before we touch code, here's
what actually changed.

## What's new in 2026-07-28

- **Stateless protocol core.** MCP moves from a stateful, bidirectional session
  to a request/response model. Every request now carries its own protocol
  version, client identity, and capabilities in `_meta` — the old
  `initialize` / `initialized` handshake and the `Mcp-Session-Id` header are
  retired. Servers can now sit behind a plain round-robin load balancer with
  no shared session state.
- **Multi Round-Trip Requests (MRTR).** Instead of a server holding a stream
  open to ask the client something mid-call, a tool can return
  `resultType: "input_required"`. The client answers and retries with
  `inputResponses` attached — interactive tools without a persistent
  connection.
- **Header-based routing.** Requests carry `Mcp-Method` and `Mcp-Name` HTTP
  headers, so gateways and rate limiters can route and meter traffic without
  parsing the JSON body.
- **Cacheable list responses.** `tools/list`, `prompts/list`, and friends now
  carry `ttlMs` and `cacheScope`, so clients can cache them intelligently
  instead of re-fetching on every call.
- **Authorization hardening.** RFC 9207 issuer validation is required, and
  Dynamic Client Registration gives way to Client ID Metadata Documents
  (CIMD).
- **Tasks became an extension.** Long-running work moved out of the core spec
  into the official `io.modelcontextprotocol/tasks` extension, using
  poll-based operations instead of open streams.

That's a lot of protocol-level change — but the good news for anyone building
a basic server is that most of it is **handled by the SDK**, not by your code.

## What changes in your code

The Python SDK's [v2 release](https://github.com/modelcontextprotocol/python-sdk)
supports "the 2026-07-28 specification (and every earlier revision)" — it
negotiates the protocol version per request and transparently serves both
legacy and modern clients. For the single-tool server we built in the
[first post]({% post_url 2026-07-29-your-first-mcp-server-in-python %}), the
visible change is small:

| v1 | v2 |
|---|---|
| `from mcp.server.fastmcp import FastMCP` | `from mcp.server import MCPServer` |
| `mcp = FastMCP("sample-data")` | `mcp = MCPServer("sample-data")` |
| `@mcp.tool()` | `@mcp.tool()` — unchanged |
| `mcp.run()` | `mcp.run()` — unchanged |

Same decorator-driven ergonomics, same "your type hints *are* the schema"
philosophy — just a renamed entry point.

## Setup

```bash
pip install "mcp[cli]"
```
{: .nolineno }

> If your environment already has the SDK installed, make sure it's v2 —
> `pip show mcp` should report a version built against the 2026-07-28 spec.
{: .prompt-tip }

## The server

Same example as before: one tool, `get_sample`, that takes a *type* of data
and returns one example value of that type.

```python
import random

from mcp.server import MCPServer

# The name shows up in the client so users know which server they're talking to.
mcp = MCPServer("sample-data")

# A tiny "database": each data type maps to a few example values.
SAMPLES = {
    "animal": ["tiger", "elephant", "penguin", "koala"],
    "fruit": ["mango", "banana", "cherry", "kiwi"],
    "color": ["crimson", "teal", "amber", "indigo"],
    "country": ["India", "Japan", "Brazil", "Norway"],
}


@mcp.tool()
def get_sample(data_type: str) -> str:
    """Return one example value of the requested data type.

    Args:
        data_type: The category to sample from, e.g. "animal", "fruit",
            "color", or "country".
    """
    key = data_type.strip().lower()
    if key not in SAMPLES:
        available = ", ".join(sorted(SAMPLES))
        raise ValueError(
            f"Unknown data type '{data_type}'. Try one of: {available}."
        )
    return random.choice(SAMPLES[key])


if __name__ == "__main__":
    # Runs over stdio by default. The SDK negotiates 2026-07-28 vs. legacy
    # protocol per request — nothing here needs to know which era it's serving.
    mcp.run()
```

Ask for `"animal"` and you'll get one of `tiger`, `elephant`, `penguin`, or
`koala` back — exactly as before. The tool's contract hasn't changed at all;
only the server class that hosts it has.

## Try it out

The `mcp` CLI still ships the same development inspector:

```bash
mcp dev sample_server.py
```
{: .nolineno }

Call `get_sample` with `animal` from the inspector UI and confirm you get a
real value back before wiring it into a client.

## Where the new spec actually shows up

Nothing in `get_sample` needed MRTR — it's a pure, single-shot function with
no follow-up questions. But the shape is there if you need it. A tool that
needs to ask the caller something mid-execution can now do this instead of
holding a stream open:

```python
from mcp.server import Context, InputRequiredResult


@mcp.tool()
def risky_action(ctx: Context, target: str) -> str:
    """Perform an action that needs confirmation first."""
    if not ctx.input_responses:
        return InputRequiredResult(
            prompt=f"Confirm you want to act on '{target}'? (yes/no)"
        )
    if ctx.input_responses[0].strip().lower() != "yes":
        return "Cancelled."
    return f"Done: acted on {target}."
```

On the first call, the tool returns `InputRequiredResult` instead of a final
answer; the client collects the response and retries the same call with
`ctx.input_responses` populated — no server-held stream, no session state.

## Connect it to a client

The Claude Desktop config is unchanged — same command, same entry point:

```json
{
  "mcpServers": {
    "sample-data": {
      "command": "python",
      "args": ["/absolute/path/to/sample_server.py"]
    }
  }
}
```

## Migrating an existing v1 server

If you have a `FastMCP`-based server already running, the SDK's migration
guide is the place to start, but for anything shaped like the example above,
the checklist is short:

1. Swap the import: `mcp.server.fastmcp.FastMCP` → `mcp.server.MCPServer`.
2. Drop any code that reads `Mcp-Session-Id` or manages session state by
   hand — the stateless core replaces it.
3. If you were streaming server-initiated follow-up questions, look at
   whether `InputRequiredResult` + MRTR now covers the same case more simply.
4. Re-test with `mcp dev` — decorators, transports, and tool schemas are
   otherwise untouched.

A protocol shift this size sounds intimidating from the outside, but for a
server this small, it really is close to a one-line change.
