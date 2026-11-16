---
sidebar_position: 20
title: "Serving Agents over HTTP - agent.serve()"
description: "Expose a SynapseKit agent as a REST endpoint with agent.serve(): JSON responses, Server-Sent Events streaming, and error behaviour."
---

# Serving Agents over HTTP

`agent.serve()` turns an in-process agent into a small web server, so other programs (a web UI, a mobile app, a script) can send it questions over HTTP. It exposes a single `POST /chat` endpoint. If the agent can stream, the response is streamed as Server-Sent Events (SSE); otherwise it is plain JSON.

## Install

FastAPI and uvicorn are optional and only loaded when you serve an agent:

```bash
pip install 'synapsekit[serve]'
```

## Quick start

`FunctionCallingAgent`, `ReActAgent` and `ReasoningAgent` have a built-in `.serve()` method:

```python
from synapsekit import FunctionCallingAgent, CalculatorTool

agent = FunctionCallingAgent(llm=llm, tools=[CalculatorTool()])
agent.serve(port=8000)  # blocks until you stop the server
```

You can also serve any object that has an async `run()` or `stream()` method with `serve_agent`:

```python
from synapsekit.serve import serve_agent


class EchoAgent:
    async def run(self, query: str) -> str:
        return f"You said: {query}"


serve_agent(EchoAgent(), port=8000)
```

Once it is running, open `http://127.0.0.1:8000/docs` for the auto-generated interactive API page.

## Request

Send a `POST` request to `/chat` with a JSON body containing `query`:

```bash
curl -X POST http://127.0.0.1:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"query": "hi"}'
```

## Responses

`serve()` checks whether the agent has a `stream()` method.

### Agent with only `run()` - JSON

```json
{"response": "You said: hi"}
```

### Agent with `stream()` - Server-Sent Events

The response has content type `text/event-stream`. Each chunk is sent as a `data:` line with a `token`, and the stream ends with `[DONE]`:

```text
data: {"token": "You "}

data: {"token": "said: "}

data: {"token": "hi "}

data: [DONE]
```

Use `curl -N` to see chunks as they arrive. Non-string chunks are converted to strings instead of aborting the stream.

## Errors

| Situation | Result |
|---|---|
| Missing `query` field | HTTP 422 with a FastAPI validation error, for example `{"detail":[{"type":"missing","loc":["body","query"],"msg":"Field required","input":{}}]}` |
| `run()` raises an exception | HTTP 500 with `{"detail": "internal server error"}` |
| `stream()` raises mid-stream | An SSE event `data: {"error": "internal server error"}` |

Internal exception messages are never sent to the client. They are logged on the server.

## Options

| Parameter | Default | Description |
|---|---|---|
| `host` | `"127.0.0.1"` | Interface to bind to |
| `port` | `8000` | Port to listen on |
| `path` | `"/chat"` | URL path of the endpoint |

:::warning
The endpoint has **no authentication**. By default it only listens on `127.0.0.1` (your own machine). Pass `host="0.0.0.0"` only if you deliberately want to expose it on all network interfaces, and put authentication in front of it.
:::

## Testing and embedding

`build_agent_app(agent, path="/chat")` builds the FastAPI app without running it. Use it in tests with FastAPI's `TestClient`, or mount it inside a larger application:

```python
from fastapi.testclient import TestClient
from synapsekit.serve import build_agent_app

client = TestClient(build_agent_app(EchoAgent()))
response = client.post("/chat", json={"query": "hi"})
```