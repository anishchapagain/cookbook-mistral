# Human-in-the-Loop Tool Confirmation Cookbook

Add approval flows to tool calls so users can review and confirm or reject actions before they execute.

> **API status:** This cookbook uses the Python SDK's `extra` module (`RunContext`, `DeferredToolCallsException`). These are **beta** features and may change.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Concepts](#concepts)
- [Recipes](#recipes)
  - [1. Raw API with curl](#1-raw-api-with-curl)
  - [2. Local Functions with Confirmation](#2-local-functions-with-confirmation)
  - [3. Connector (Gmail) with Confirmation](#3-connector-gmail-with-confirmation)
  - [4. Server-Side Confirmation (Web Search)](#4-server-side-confirmation-web-search)
  - [5. Mixed Tools — MCP + Local Functions + Built-in Tools](#5-mixed-tools--mcp--local-functions--built-in-tools)
  - [6. Streaming with Confirmation](#6-streaming-with-confirmation)
  - [7. Stateless / API-Friendly — Serialize and Resume](#7-stateless--api-friendly--serialize-and-resume)

---

## Prerequisites

### Install

`mistralai-private` is hosted on [Gemfury](https://gemfury.com/). Make sure the Gemfury index is configured before installing.

```bash
uv add 'mistralai-private>=1.14.2'
```

For [Recipe 5 (MCP mixed tools)](#5-mixed-tools--mcp--local-functions--built-in-tools), you also need the `agents` extra:

```bash
uv add 'mistralai-private[agents]>=1.14.2'
```

### Required environment variables

```bash
MISTRAL_API_KEY=your-mistral-api-key
```

Get your API key from the [Mistral AI dashboard](https://console.mistral.ai/).

---

## Concepts

There are two kinds of deferral flows with the `RunContext.run_async` loop:
- Client-side: local functions registered through the run context `register_func` or local MCPs through `register_mcp_client`
- Server-side: remote connectors (gmail, web-search, ...)

The `run_async` loop is responsible for interrupting itself when encountering a deferred tool call, deferred tool calls can be manifested both by local functions
or by server-side events (through FunctionCallEntry with `confirmation_status: "pending"`).

### Configuration

To configure the behavior we use the same flow as for connectors with the `requires_confirmation` list.

```
tools=[
      {
          "type": "connector",
          "connector_id": "gmail",
          "tool_configuration": {
              "include": ["gmail_search"],
              "exclude": ["gmail_send"],           # mutually exclusive with include
              "requires_confirmation": ["gmail_search"],
          },
      },
      {
          "type": "web_search_premium",
          "tool_configuration": {
              "requires_confirmation": ["web_search", "news_search"],
          },
      },
  ]
```


---

## Recipes

---

### 1. Raw API with curl

**Goal:** The Conversations API should return `FunctionCallEntry` with `confirmation_status: "pending"` for tools that require confirmation, and accept `tool_confirmations` with `"allow"` or `"deny"` to resume.

#### Connector example (Gmail)
> Get your GMail token by enabling a Gmail connector in a session on [Le Chat](https://chat.mistral.ai) and then clicking on the 🐞 icon and searching for string `oauth` to retrieve your token.

**Step 1 — Start a conversation with `requires_confirmation` and extract the IDs:**

```bash
RESPONSE=$(curl -s -X POST "https://api.mistral.ai/v1/conversations" \
  -H "Authorization: Bearer ${MISTRAL_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mistral-medium-latest",
    "stream": false,
    "inputs": "whats the title of the last email i received?",
    "tools": [
      {
        "type": "connector",
        "connector_id": "gmail",
        "authorization": {
          "type": "oauth2-token",
          "value": "'${GMAIL_OAUTH_TOKEN}'"
        },
        "tool_configuration": {
          "requires_confirmation": ["gmail_search"]
        }
      }
    ]
  }')

echo "$RESPONSE" | jq .

# Extract IDs for the next step
CONVERSATION_ID=$(echo "$RESPONSE" | jq -r '.conversation_id')
TOOL_CALL_ID=$(echo "$RESPONSE" | jq -r '.outputs[0].tool_call_id')
```

**Expected response — early return with a pending `function.call`:**

```json
{
    "object": "conversation.response",
    "conversation_id": "conv_abc123...",
    "outputs": [
        {
            "object": "entry",
            "type": "function.call",
            "created_at": "2026-02-19T13:08:53.099130Z",
            "completed_at": null,
            "id": "fc_019c760482eb706191b8e7396e39d540",
            "tool_call_id": "WJfo42Ow3",
            "name": "gmail_search",
            "arguments": "{\"limit\": 1}",
            "confirmation_status": "pending"
        }
    ],
    "usage": {
        "prompt_tokens": 561,
        "completion_tokens": 11,
        "total_tokens": 572
    }
}
```

**Step 2a — Approve the tool call:**
Use the values from Step 1.

```bash
curl -X POST "https://api.mistral.ai/v1/conversations/${CONVERSATION_ID}" \
  -H "Authorization: Bearer ${MISTRAL_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "tool_confirmations": [
      {
        "tool_call_id": "'${TOOL_CALL_ID}'",
        "confirmation": "allow"
      }
    ]
  }'
```

**Expected response — the server executes the tool and the model answers:**

```json
{
    "object": "conversation.response",
    "conversation_id": "conv_abc123...",
    "outputs": [
        {
            "object": "entry",
            "type": "message.output",
            "created_at": "2026-02-19T13:18:37.951771Z",
            "completed_at": "2026-02-19T13:18:38.668420Z",
            "id": "msg_019c760d6f7f77ff9bd7b1186226483e",
            "model": "mistral-medium-latest",
            "role": "assistant",
            "content": "Your last email is a marketing email from Cursor, feel free to ask!"
        }
    ],
    "usage": {
        "prompt_tokens": 610,
        "completion_tokens": 44,
        "total_tokens": 654
    }
}
```

**Step 2b — Or deny instead:**
Use the values from Step 1.

```bash
curl -X POST "https://api.mistral.ai/v1/conversations/${CONVERSATION_ID}" \
  -H "Authorization: Bearer ${MISTRAL_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "tool_confirmations": [
      {
        "tool_call_id": "'${TOOL_CALL_ID}'",
        "confirmation": "deny"
      }
    ]
  }'
```

**Expected response — the model handles the rejection gracefully:**

```json
{
    "object": "conversation.response",
    "conversation_id": "conv_abc123...",
    "outputs": [
        {
            "object": "entry",
            "type": "message.output",
            "created_at": "2026-02-19T13:21:19.675520Z",
            "completed_at": "2026-02-19T13:21:20.283129Z",
            "id": "msg_019c760fe73b7794913721d4866b3623",
            "model": "mistral-medium-latest",
            "role": "assistant",
            "content": "Sorry, but I couldn't retrieve the information as the tool call was denied. If you have any other questions or need assistance with something else, feel free to ask!"
        }
    ],
    "usage": {
        "prompt_tokens": 597,
        "completion_tokens": 35,
        "total_tokens": 632
    }
}
```

#### Built-in tools example (Web Search)

Multiple tools can require confirmation in a single response:

```bash
RESPONSE=$(curl -s -X POST "https://api.mistral.ai/v1/conversations" \
  -H "Authorization: Bearer ${MISTRAL_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "mistral-medium-latest",
    "stream": false,
    "inputs": "what are the most recent AI news? use both web_search & news_search tools",
    "tools": [
      {
        "type": "web_search_premium",
        "tool_configuration": {
          "requires_confirmation": ["web_search", "news_search"]
        }
      }
    ]
  }')

echo "$RESPONSE" | jq .

# Extract IDs for the next step
CONVERSATION_ID=$(echo "$RESPONSE" | jq -r '.conversation_id')
TOOL_CALL_ID_1=$(echo "$RESPONSE" | jq -r '.outputs[0].tool_call_id')
TOOL_CALL_ID_2=$(echo "$RESPONSE" | jq -r '.outputs[1].tool_call_id')
```

**Expected response — both tool calls returned as pending:**

```json
{
    "object": "conversation.response",
    "conversation_id": "conv_abc123...",
    "outputs": [
        {
            "object": "entry",
            "type": "function.call",
            "id": "fc_019c763128da74b5959a9a4f6a689edb",
            "tool_call_id": "khQCtUy16",
            "name": "web_search",
            "arguments": "{\"query\": \"recent AI news\"}",
            "confirmation_status": "pending"
        },
        {
            "object": "entry",
            "type": "function.call",
            "id": "fc_019c7631294670ecb5d124bbd5048ce7",
            "tool_call_id": "KKt6VP6ew",
            "name": "news_search",
            "arguments": "{\"query\": \"AI news\", \"start_date\": \"2026-02-12\", \"end_date\": \"2026-02-19\"}",
            "confirmation_status": "pending"
        }
    ],
    "usage": {
        "prompt_tokens": 385,
        "completion_tokens": 57,
        "total_tokens": 442
    }
}
```

**Resume — confirm both (or deny selectively):**

```bash
curl -X POST "https://api.mistral.ai/v1/conversations/${CONVERSATION_ID}" \
  -H "Authorization: Bearer ${MISTRAL_API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "tool_confirmations": [
      { "tool_call_id": "'${TOOL_CALL_ID_1}'", "confirmation": "allow" },
      { "tool_call_id": "'${TOOL_CALL_ID_2}'", "confirmation": "allow" }
    ]
  }'
```

---

### 2. Local Functions with Confirmation

**Goal:** `run_async` should auto-execute tools with `requires_confirmation=False`, raise `DeferredToolCallsException` for tools with `requires_confirmation=True`, and resume after calling `dc.confirm()` or `dc.reject()`.

```python
import asyncio
import os
import random

from mistralai_private import MistralPrivate
from mistralai_private.extra.run.context import RunContext
from mistralai_private.extra.exceptions import DeferredToolCallsException

MODEL = "mistral-large-latest"


def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    temp = random.randint(10, 30)
    conditions = random.choice(["sunny", "cloudy", "partly cloudy"])
    return f"The weather in {city} is {conditions}, {temp}C"


def book_flight(destination: str, date: str) -> str:
    """Book a flight to a destination."""
    return f"Flight booked to {destination} on {date}. Confirmation: FL-{random.randint(10000, 99999)}"


def request_approval(dc) -> bool:
    print(f"\n[APPROVAL REQUIRED] {dc.tool_name}")
    print(f"  Arguments: {dc.arguments}")
    return input("  Approve? (y/n): ").strip().lower() == "y"


async def main():
    client = MistralPrivate(api_key=os.environ["MISTRAL_API_KEY"])

    conversation_id = None
    pending_inputs = [
        {"role": "user", "content": "I need a vacation somewhere warm next Friday. Can you help?"}
    ]

    while True:
        async with RunContext(model=MODEL) as run_ctx:
            run_ctx.conversation_id = conversation_id
            run_ctx.register_func(get_weather, requires_confirmation=False)
            run_ctx.register_func(book_flight, requires_confirmation=True)

            try:
                result = await client.beta.conversations.run_async(
                    run_ctx=run_ctx,
                    inputs=pending_inputs,
                    instructions="You are a travel assistant. Available destinations are: Guingamp, Aurillac, Brive-la-Gaillarde, Rodez, and Millau. Check the weather and book a flight to the warmest one. Do not ask for confirmation, just book it.",
                )

                print(f"\nFinal response: {result.output_entries}")
                break

            except DeferredToolCallsException as deferred:
                conversation_id = deferred.conversation_id

                pending_inputs = [
                    dc.confirm() if request_approval(dc) else dc.reject("Denied by user")
                    for dc in deferred.deferred_calls
                ]


asyncio.run(main())
```

Expected flow:

```
> get_weather("Guingamp") → auto-executed
> get_weather("Aurillac") → auto-executed
> book_flight("Aurillac", "2025-03-21") → APPROVAL REQUIRED
  Approve? (y/n): y
> Final response: Flight booked to Aurillac on 2025-03-21. Confirmation: FL-42857
```

---

### 3. Connector (Gmail) with Confirmation

**Goal:** `run_async` should raise `DeferredToolCallsException` when the server returns a `FunctionCallEntry` with `confirmation_status: "pending"` for a connector tool listed in `requires_confirmation`. Resuming with `dc.confirm()` should send `tool_confirmations` back to the server.

Requires a valid Google OAuth2 token (`GMAIL_OAUTH_TOKEN` env var).

```python
import asyncio
import os

from mistralai_private import MistralPrivate
from mistralai_private.extra.exceptions import DeferredToolCallsException
from mistralai_private.extra.run.context import RunContext

MODEL = "mistral-large-latest"


def request_approval(dc) -> bool:
    print(f"\n[APPROVAL REQUIRED] {dc.tool_name}")
    print(f"  Arguments: {dc.arguments}")
    return input("  Approve? (y/n): ").strip().lower() == "y"


async def main():
    client = MistralPrivate(api_key=os.environ["MISTRAL_API_KEY"])

    conversation_id = None
    pending_inputs = [
        {"role": "user", "content": "Summarize my latest emails from Gmail."}
    ]

    while True:
        async with RunContext(model=MODEL) as run_ctx:
            run_ctx.conversation_id = conversation_id

            try:
                result = await client.beta.conversations.run_async(
                    run_ctx=run_ctx,
                    inputs=pending_inputs,
                    instructions="You are a helpful assistant. Use the Gmail connector to access the user's emails.",
                    tools=[
                        {
                            "type": "connector",
                            "connector_id": "gmail",
                            "authorization": {
                                "type": "oauth2-token",
                                "value": os.environ["GMAIL_OAUTH_TOKEN"],
                            },
                            "tool_configuration": {
                                "requires_confirmation": ["gmail_search"],
                            },
                        },
                    ],
                )

                for entry in result.output_entries:
                    if hasattr(entry, "content"):
                        print(f"\n[{entry.type}] {entry.content}")
                    else:
                        print(f"\n[{entry.type}] {entry.name}({entry.arguments})")
                break

            except DeferredToolCallsException as deferred:
                conversation_id = deferred.conversation_id

                pending_inputs = [
                    dc.confirm() if request_approval(dc) else dc.reject("Denied by user")
                    for dc in deferred.deferred_calls
                ]


asyncio.run(main())
```

---

### 4. Server-Side Confirmation (Web Search)

**Goal:** `run_async` should detect server-side deferred tool calls (`confirmation_status: "pending"`) and raise `DeferredToolCallsException` with `reason=DeferralReason.SERVER_SIDE_CONFIRMATION_REQUIRED`. Already-executed results should be available via `deferred.executed_results`.

```python
import asyncio
import os

from mistralai_private import MistralPrivate
from mistralai_private.extra.run.context import RunContext
from mistralai_private.extra.exceptions import DeferredToolCallsException, DeferralReason

MODEL = "mistral-medium-latest"


def request_approval(dc) -> bool:
    print(f"\n[APPROVAL REQUIRED] {dc.tool_name}")
    print(f"  Arguments: {dc.arguments}")
    print(f"  Reason: {dc.reason.value}")
    return input("  Approve? (y/n): ").strip().lower() == "y"


async def main():
    client = MistralPrivate(api_key=os.environ["MISTRAL_API_KEY"])

    conversation_id = None
    pending_inputs = "Search the web for the latest news about artificial intelligence today."

    while True:
        async with RunContext(model=MODEL) as run_ctx:
            if conversation_id:
                run_ctx.conversation_id = conversation_id

            try:
                run_result = await client.beta.conversations.run_async(
                    run_ctx=run_ctx,
                    inputs=pending_inputs,
                    instructions="You are a helpful assistant. Use web search to find information.",
                    tools=[{
                        "type": "web_search_premium",
                        "tool_configuration": {
                            "requires_confirmation": ["web_search", "news_search"],
                        },
                    }],
                )

                print("\nFinal Response:")
                print(run_result.output_as_text)
                break

            except DeferredToolCallsException as deferred:
                conversation_id = deferred.conversation_id
                print(f"\n{len(deferred.deferred_calls)} tool(s) need approval:")

                for dc in deferred.deferred_calls:
                    is_server_side = dc.reason == DeferralReason.SERVER_SIDE_CONFIRMATION_REQUIRED
                    print(f"  - {dc.tool_name} (server_side={is_server_side})")

                confirmations = [
                    dc.confirm() if request_approval(dc) else dc.reject("Denied by user")
                    for dc in deferred.deferred_calls
                ]
                pending_inputs = deferred.executed_results + confirmations


asyncio.run(main())
```

---

### 5. Mixed Tools — MCP + Local Functions + Built-in Tools

**Goal:** `run_async` should handle three tool sources (remote MCP, local functions, built-in tools) in a single conversation. Each source should respect its own confirmation settings independently. `deferred.executed_results` should contain results from tools that auto-executed before the deferral.

Requires `pip install mistralai-private[agents]` and a running MCP SSE server (this example uses CoinGecko).

```python
import asyncio
import os

from mistralai_private import MistralPrivate
from mistralai_private.extra.run.context import RunContext
from mistralai_private.extra.mcp.sse import MCPClientSSE, SSEServerParams
from mistralai_private.extra.exceptions import DeferredToolCallsException

MODEL = "mistral-medium-latest"

USER_PORTFOLIO = {
    "bitcoin": 0.5,
    "ethereum": 2.0,
    "solana": 10.0,
}


def get_portfolio() -> dict:
    """Get the user's cryptocurrency portfolio holdings.

    Returns:
        dict: A dictionary mapping coin names to amounts held
    """
    return USER_PORTFOLIO


def execute_trade(coin: str, action: str, amount: float) -> str:
    """Execute a buy or sell trade for a cryptocurrency.

    Args:
        coin: The cryptocurrency to trade (e.g., 'bitcoin', 'ethereum')
        action: Either 'buy' or 'sell'
        amount: The amount to trade

    Returns:
        str: Confirmation message of the trade
    """
    return f"Trade executed: {action} {amount} {coin}"


def request_approval(dc) -> bool:
    print(f"\n[APPROVAL REQUIRED] {dc.tool_name}")
    print(f"  Arguments: {dc.arguments}")
    return input("  Approve? (y/n): ").strip().lower() == "y"


async def main():
    client = MistralPrivate(api_key=os.environ["MISTRAL_API_KEY"])

    coingecko_url = "https://mcp.api.coingecko.com/sse"
    mcp_client = MCPClientSSE(sse_params=SSEServerParams(url=coingecko_url, timeout=60))

    conversation_id = None
    pending_inputs = (
        "I want a full analysis of my crypto portfolio. "
        "First, get my current holdings using get_portfolio. "
        "Then use get_coins_markets from CoinGecko to get market data for Bitcoin, Ethereum and Solana. "
        "Search the web for any recent news about Bitcoin. "
        "Finally, if Bitcoin's price is under $100k, buy 0.1 more using execute_trade."
    )

    while True:
        async with RunContext(model=MODEL) as run_ctx:
            if conversation_id:
                run_ctx.conversation_id = conversation_id

            await run_ctx.register_mcp_client(mcp_client=mcp_client)
            run_ctx.register_func(get_portfolio, requires_confirmation=False)
            run_ctx.register_func(execute_trade, requires_confirmation=True)

            try:
                run_result = await client.beta.conversations.run_async(
                    run_ctx=run_ctx,
                    inputs=pending_inputs,
                    instructions=(
                        "You are a crypto portfolio assistant. "
                        "You have access to: get_portfolio (local holdings), "
                        "CoinGecko tools (get_coins_markets for market data), "
                        "web_search_premium (for news), and execute_trade (for transactions)."
                    ),
                    tools=[{"type": "web_search_premium"}],
                )

                print("\nFinal Response:")
                print(run_result.output_as_text)
                break

            except DeferredToolCallsException as deferred:
                conversation_id = deferred.conversation_id
                print(f"\n{len(deferred.deferred_calls)} tool(s) need approval:")

                confirmations = [
                    dc.confirm() if request_approval(dc) else dc.reject("Denied by user")
                    for dc in deferred.deferred_calls
                ]
                pending_inputs = deferred.executed_results + confirmations


asyncio.run(main())
```

Expected flow:

```
> get_portfolio() → auto-executed
> get_coins_markets(...) → auto-executed (MCP)
> web_search("bitcoin news") → auto-executed
> execute_trade("bitcoin", "buy", 0.1) → APPROVAL REQUIRED
  Approve? (y/n): y
> Final Response: Your portfolio analysis...
```

---

### 6. Streaming with Confirmation

**Goal:** `run_stream_async` should yield streaming events, then raise `DeferredToolCallsException` mid-stream when a deferred tool call is encountered. Resuming should work the same as `run_async`.

```python
import asyncio
import os
import random

from mistralai_private import MistralPrivate
from mistralai_private.extra.run.context import RunContext
from mistralai_private.extra.run.result import RunResult
from mistralai_private.extra.exceptions import DeferredToolCallsException

MODEL = "mistral-large-latest"


def get_weather(city: str) -> str:
    """Get the current weather for a city."""
    temp = random.randint(10, 30)
    conditions = random.choice(["sunny", "cloudy", "partly cloudy"])
    return f"The weather in {city} is {conditions}, {temp}C"


def book_flight(destination: str, date: str) -> str:
    """Book a flight to a destination."""
    return f"Flight booked to {destination} on {date}. Confirmation: FL-{random.randint(10000, 99999)}"


def request_approval(dc) -> bool:
    print(f"\n[APPROVAL REQUIRED] {dc.tool_name}")
    print(f"  Arguments: {dc.arguments}")
    return input("  Approve? (y/n): ").strip().lower() == "y"


async def main():
    client = MistralPrivate(api_key=os.environ["MISTRAL_API_KEY"])

    conversation_id = None
    pending_inputs = [
        {"role": "user", "content": "I need a vacation somewhere warm next Friday. Can you help?"}
    ]

    while True:
        async with RunContext(model=MODEL) as run_ctx:
            run_ctx.conversation_id = conversation_id
            run_ctx.register_func(get_weather, requires_confirmation=False)
            run_ctx.register_func(book_flight, requires_confirmation=True)

            try:
                events = await client.beta.conversations.run_stream_async(
                    run_ctx=run_ctx,
                    inputs=pending_inputs,
                    instructions="You are a travel assistant. Available destinations are: Guingamp, Aurillac, Brive-la-Gaillarde, Rodez, and Millau. Check the weather and book a flight to the warmest one. Do not ask for confirmation, just book it.",
                )

                async for event in events:
                    if isinstance(event, RunResult):
                        print(f"\nFinal response: {event.output_entries}")
                        return
                    else:
                        print(f"Event: {event}")

            except DeferredToolCallsException as deferred:
                conversation_id = deferred.conversation_id

                pending_inputs = [
                    dc.confirm() if request_approval(dc) else dc.reject("Denied by user")
                    for dc in deferred.deferred_calls
                ]


asyncio.run(main())
```

---

### 7. Stateless / API-Friendly — Serialize and Resume

**Goal:** `DeferredToolCallsException.to_dict()` should produce a JSON-serializable dict, and `from_dict()` should reconstruct the full state so the conversation can resume from a different process/context.

Split into two scripts to simulate a real API boundary (e.g., backend returns deferred state to frontend, frontend sends back approvals).

#### Script 1: Start the conversation, catch the deferral, serialize it

```python
import asyncio
import json
import os

from mistralai_private import MistralPrivate
from mistralai_private.extra.run.context import RunContext
from mistralai_private.extra.exceptions import DeferredToolCallsException


def book_flight(destination: str, date: str) -> str:
    """Book a flight to a destination."""
    return f"Flight booked to {destination} on {date}"


async def main():
    client = MistralPrivate(api_key=os.environ["MISTRAL_API_KEY"])

    async with RunContext(model="mistral-large-latest") as run_ctx:
        run_ctx.register_func(book_flight, requires_confirmation=True)

        try:
            result = await client.beta.conversations.run_async(
                run_ctx=run_ctx,
                inputs=[{"role": "user", "content": "Book me a flight to Paris next Friday."}],
                instructions="You are a travel assistant. Book the flight directly.",
            )
            print("No confirmation needed:", result.output_as_text)

        except DeferredToolCallsException as deferred:
            state = deferred.to_dict()
            serialized = json.dumps(state)

            print("Deferred state (send this to your frontend / store it):")
            print(serialized)


asyncio.run(main())
```

#### Script 2: Receive approvals, deserialize, and resume

```python
import asyncio
import json
import os

from mistralai_private import MistralPrivate
from mistralai_private.extra.run.context import RunContext
from mistralai_private.extra.exceptions import DeferredToolCallsException, DeferredToolCallEntry


def book_flight(destination: str, date: str) -> str:
    """Book a flight to a destination."""
    return f"Flight booked to {destination} on {date}"


async def main():
    # In a real app: receive this from the frontend / load from DB
    serialized = os.environ["DEFERRED_STATE"]  # the JSON string from Script 1
    state = json.loads(serialized)

    # Reconstruct the exception from the serialized state
    deferred = DeferredToolCallsException.from_dict(state)

    # Build confirmations (in a real app, the frontend tells you which to approve/reject)
    pending_inputs = []
    for dc in deferred.deferred_calls:
        print(f"Tool: {dc.tool_name}, Args: {dc.arguments}")
        pending_inputs.append(dc.confirm())

    # Include any already-executed results
    pending_inputs = list(deferred.executed_results) + pending_inputs

    # Resume the conversation
    client = MistralPrivate(api_key=os.environ["MISTRAL_API_KEY"])

    async with RunContext(model="mistral-large-latest") as run_ctx:
        run_ctx.conversation_id = deferred.conversation_id
        run_ctx.register_func(book_flight, requires_confirmation=True)

        result = await client.beta.conversations.run_async(
            run_ctx=run_ctx,
            inputs=pending_inputs,
            instructions="You are a travel assistant. Book the flight directly.",
        )
        print("Final response:", result.output_as_text)


asyncio.run(main())
```
