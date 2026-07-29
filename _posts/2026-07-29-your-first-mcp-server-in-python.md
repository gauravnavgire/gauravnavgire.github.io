---
title: "Your First MCP Server in Python"
date: 2026-07-29 10:30:00 +0530
categories: [AI, Tutorials]
tags: [mcp, python, llm, anthropic]
---

A year ago, the [Model Context Protocol](https://blog.modelcontextprotocol.io/posts/2025-11-25-first-mcp-anniversary/)
(MCP) started as "a little open-source experiment" from Anthropic. Twelve
months later it has become the de facto way to connect large language models to
real data and tools — over 2,000 servers exist, with adoption spanning GitHub,
Stripe, Notion, Microsoft, Google, AWS, and more.

To celebrate the anniversary, let's build the smallest useful thing you can with
MCP: **a server that exposes a single tool**. The tool takes a *type* of data
and returns one example of it. Ask it for `animal` and it hands back `tiger`;
ask for `fruit` and you get `mango`.

## What is an MCP server?

An MCP server is a small program that advertises **capabilities** — tools,
resources, and prompts — to an MCP *client* (Claude Desktop, an IDE extension,
or any custom app). The client asks the model what it wants to do; when the
model decides to call a tool, the client forwards that call to your server,
your server runs the code, and the result flows back to the model. You write
plain Python functions; MCP handles the wiring.

## Setup

The official Python SDK ships the `mcp` package. The `[cli]` extra adds a handy
development runner:

```bash
pip install "mcp[cli]"
```
{: .nolineno }

> If you use [`uv`](https://docs.astral.sh/uv/), `uv add "mcp[cli]"` works too.
{: .prompt-tip }

## The server

Save this as `sample_server.py`. The `FastMCP` helper is the quickest way to
stand up a server — decorate a function with `@mcp.tool()` and it becomes a
tool the model can call, with the type hints and docstring turned into the
tool's schema automatically.

```python
import random

from mcp.server.fastmcp import FastMCP

# The name shows up in the client so users know which server they're talking to.
mcp = FastMCP("sample-data")

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
    # Runs over stdio by default — the transport MCP clients expect.
    mcp.run()
```

That's the whole server. A few things worth noting:

- **The docstring is the contract.** The model reads the description and the
  `Args` to decide when and how to call `get_sample`, so write it for a reader
  who has never seen your code.
- **Type hints become the schema.** `data_type: str` tells MCP the input is a
  string — no separate schema file needed.
- **Raising is fine.** When the input is unknown, raising `ValueError` returns a
  tool error the model can read and recover from, rather than crashing the
  server.

Ask for `"animal"` and you'll get one of the four animals back — `tiger` is one
possible answer. Swap `random.choice` for `SAMPLES[key][0]` if you'd rather it
always return the same representative value.

## Try it out

The CLI's development inspector launches a local UI where you can call the tool
by hand and watch the request/response:

```bash
mcp dev sample_server.py
```
{: .nolineno }

Click into `get_sample`, pass `animal`, and confirm you get an animal back
before wiring it into a real client.

## Connect it to a client

To let Claude Desktop use the server, add it to the client's MCP config
(`claude_desktop_config.json`):

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

Restart the client and it'll discover `get_sample`. Now, when you ask something
like *"give me an example animal,"* the model can call your tool and answer with
a real value from your server instead of guessing.

## Where to go next

This is deliberately the *first version* — one tool, in-memory data, stdio
transport. From here you can:

- Add more tools (`list_types`, `add_sample`, ...) — each is just another
  decorated function.
- Expose **resources** (read-only data the model can pull in as context) and
  **prompts** (reusable prompt templates).
- Swap the in-memory dict for a database or an external API.
- Serve over HTTP instead of stdio for remote clients.

The protocol that started as an experiment is now a foundational piece of AI
infrastructure — and the barrier to contributing to it is about 30 lines of
Python.
