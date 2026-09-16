# Agentic AI Workflows on AWS

**One agent, built three ways — to see what you give up each time you let something else drive the loop.**

---

## What an agent is

Not a chatbot. A loop.

```mermaid
flowchart TD
    Q["User question<br/>What's the weather in Seattle?"] --> M
    M["Model<br/>picks a tool"] --> T["Tool runs<br/>calls the weather API"]
    T --> M
    M --> A["Answer<br/>5-day forecast"]
```

The model isn't told which tool to use. It reads the question, decides, calls, looks at the result, and goes around again until it can answer. No `if` statements anywhere.

## The question this project asks

Someone has to write that loop. The interesting part is **who**.

```mermaid
flowchart LR
    L1["Lab 1<br/>You write the loop<br/>Step Functions + Lambda"] --> L2["Lab 2<br/>A framework writes it<br/>Strands SDK"] --> L3["Lab 3<br/>AWS writes it<br/>Bedrock Agents"]
```

Left to right: less code to maintain, less visibility into what happened.

| | Lab 1 | Lab 2 | Lab 3 |
|---|---|---|---|
| **Orchestration** | Your own `while` loop | Strands Agents SDK | Bedrock Agents (Supervisor) |
| **You control** | Every state and retry | Tools and prompt | Configuration only |
| **Code to write** | Most | Some | Almost none |
| **When it breaks** | You can see exactly where | Framework logs | Hardest to trace |
| **Best for** | Auditability | Everyday building | Scale |

Labs 2 and 3 plan trips. Lab 1 runs a content pipeline — the comparison is about the loop, not the domain.

---

## What it actually does

Real exchanges with Lab 2, through `POST /chat` — Cognito-authenticated, session keyed to the username.

| Ask | What happens underneath |
|---|---|
| *What are the flight options to Seattle?* | Model picks `flight_search` and passes `"Seattle"` on its own |
| *What's the weather forecast for Seattle?* | Two chained calls to the National Weather Service, returned as one answer |
| *Any local things to do?* | Seattle is never repeated — session memory in S3 carries it |
| *Tour operators with water activities?* | Answers only from the S3-indexed partner list, not general knowledge |
| *Reserve the Space Needle, Dec 20, 9 AM* | Creates a real reservation over MCP — the one that writes |

That last row is the one worth testing hardest. Everything above it is reversible; that one isn't.

---

## Lab 1 — You write the loop

A Lambda calls the Bedrock Converse API, checks whether the model asked for a tool, runs it, feeds the result back, and repeats. Step Functions draws the whole thing as a state machine, so every transition is visible.

```mermaid
sequenceDiagram
    participant L as Lambda (while loop)
    participant B as Bedrock Converse
    participant T as Tool

    loop until done
        L->>B: messages + available tools
        B-->>L: "call this tool" or final answer
        L->>T: run it
        T-->>L: result
    end
```

**Result:** routed correctly through three tool calls in one execution, each step visible in the graph and the logs. The model's choices are still non-deterministic — what this buys you is a complete record of what it chose.

![Step Functions execution graph](docs/screenshots/lab1-stepfunctions-graph-view.png)
![CloudWatch tool invocation trace](docs/screenshots/lab1-cloudwatch-tool-trace.png)

## Lab 2 — A framework writes it

Declare a model, a prompt, and a list of tools. The SDK handles reasoning, tool dispatch, and memory.

```python
travel_agent = Agent(
    model="us.amazon.nova-lite-v1:0",
    system_prompt=TRAVEL_AGENT_PROMPT,
    tools=[flight_search, http_request, retrieve, *mcp_tools],
    session_manager=session_manager,  # S3-backed memory
)
```

Two things make this more than a tool-calling demo:

- **RAG** — a Bedrock Knowledge Base over private S3 data, reached through the built-in `retrieve` tool
- **MCP** — tools discovered at runtime from an external server (AgentCore Gateway, OAuth2 via Cognito), so new tools appear without redeploying

**Result:** authenticated to the gateway, pulled flights and a multi-day forecast, and answered with both — from one prompt, with no orchestration code.

![Strands agent CLI output](docs/screenshots/lab2-strands-agent-cli-output.png)

## Lab 3 — AWS writes it

A Supervisor agent, configured rather than coded, delegates to two specialists.

```mermaid
flowchart TD
    U["Plan SF → LA, Dec 1–2"] --> S["Supervisor"]
    S --> F["Flight Agent"]
    S --> W["Weather Agent"]
    F --> S
    W --> S
    S --> R["Flights + forecast +<br/>a warning to move the date"]
```

**Result — the interesting one:** the Weather Agent returned a severe thunderstorm warning for Dec 1. The Supervisor didn't just print both answers side by side — it noticed the conflict with the flight options and recommended changing the date.

![Bedrock Agents supervisor chat](docs/screenshots/lab3-bedrock-supervisor-chat.png)

That's the behaviour worth building for: not chaining tool calls, but weighing their results against each other. It's also the behaviour that's hardest to test for, since nothing in the configuration guarantees it happens next time.

---

## Evaluation notes

Testing this taught me more than building it did. What I'd build next, in order:

- **Trajectory assertions, not text matching.** Assert on which tool was called with which arguments. An agent that returns the right answer without calling the tool is reciting, and it will go stale silently the moment the data changes.
- **Side-effect counting.** The reservation path writes. Verifying it means counting reservations created, not trusting the sentence claiming one was. Repeating the same request currently creates a second booking.
- **Error isolation.** An unsupported city raises inside the tool and comes back as a generic 500 — indistinguishable from a throttle or an auth failure. Tool errors, model reasoning failures, and throttling need to be separable before any of this is operable.
- **Token caching.** Lab 2 re-authenticates to the MCP gateway on every invocation.

## Built with

Amazon Bedrock (Nova Lite, Claude 3.5) · AWS Lambda · Step Functions · Strands Agents SDK · Bedrock Knowledge Bases · S3 · MCP via AgentCore Gateway · Bedrock Agents · API Gateway · Cognito · CloudWatch

## What's in this repo

```
README.md
docs/screenshots/    execution evidence for all three labs
```

The labs were built and run in AWS following the workshops linked below, then torn down to avoid standing costs. Deployable source lives in those workshops. What's mine is the comparison, the testing, and the analysis.

## Attribution

Based on AWS's official agentic AI workshops (Strands Agents SDK, Bedrock, Step Functions, Bedrock Agents). The comparison, testing, and analysis are my own.

Built by [Mayu Kataoka](https://github.com/mayukataoka) — quality engineering background, exploring applied AI systems.
