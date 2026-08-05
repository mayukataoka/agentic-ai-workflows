# Agentic AI Workflows on AWS

**Comparing three ways to build an AI agent — a hand-rolled reasoning loop, an SDK-managed agent, and a fully managed multi-agent system — through a working travel-planning assistant.**

Most tutorials show you one way to build an agent. This project builds the *same* agent three times, using three different orchestration approaches, to understand the actual trade-offs between control, development speed, and visibility — not just read about them.

---

## Overview

An "agent" is a loop: a model reasons about a request, decides whether it needs a tool, calls that tool, incorporates the result, and repeats until the task is done. How much of that loop *you* write, versus how much a framework or managed service writes for you, is one of the more consequential decisions in building an agent system — it trades off control against velocity against operational visibility.

This project implements that loop three ways on AWS:

| # | Pattern | Orchestration | Key takeaway |
|---|---|---|---|
| 1 | **Manual reasoning loop** | Raw Python `while` loop + AWS Step Functions, calling the Bedrock Converse API directly | You own every state, retry, and tool call. Maximum control, maximum responsibility. |
| 2 | **SDK-driven agent** | [Strands Agents SDK](https://strandsagents.com) on AWS Lambda | The framework manages the loop, session memory, and tool-calling. Clean code, fast iteration. |
| 3 | **Managed multi-agent** | Amazon Bedrock Agents, Supervisor + collaborator pattern | AWS manages orchestration between a Supervisor and specialized sub-agents. High scalability, less visibility into the reasoning path. |

## Architecture

```mermaid
flowchart LR
    U[User] --> A[Agent]
    A <--> M["Foundation Model\n(Amazon Nova Lite / Claude 3.5)"]
    A --> T[Lambda Tools\nflight_search, weather, etc.]
    A --> KB[(Bedrock Knowledge Base\nRAG over S3 data)]
    A --> GW[MCP Gateway\nattractions / booking tools]
    T --> EXT[External API\nNational Weather Service]
```

Every pattern shares this same shape — a model deciding what to do, tools doing the work, a knowledge base grounding it in real data. What differs is *who's responsible for driving the loop*.

## The three patterns, in more detail

### Pattern 1 — Manual reasoning loop (Step Functions + Lambda)

The most explicit version: a Lambda function calls the Bedrock Converse API directly, inspects the response for a tool-use request, invokes the matching tool (a Content Summarizer, Tone Adapter, or Post Writer, in this case), feeds the result back in, and loops until the model signals it's done. AWS Step Functions visualizes this as an explicit state machine, which makes the loop auditable step-by-step — useful when you need to reason about exactly what happened and why.

```mermaid
sequenceDiagram
    participant U as User
    participant L as Lambda (while loop)
    participant B as Bedrock Converse API
    participant T as Tool (Summarizer / Adapter / Writer)

    U->>L: prompt
    loop until stopReason == end_turn
        L->>B: Converse(messages, tools)
        B-->>L: tool_use request or final text
        alt model requested a tool
            L->>T: invoke tool
            T-->>L: tool result
            L->>B: append result, continue reasoning
        else model produced final answer
            L-->>U: return response
        end
    end
```
**Result:** the state machine correctly routed between tool calls and looped until a finished output was produced, with each transition visible in the Step Functions graph and CloudWatch logs.

![Step Functions execution graph](docs/screenshots/lab1-stepfunctions-graph-view.png)
![CloudWatch tool invocation trace](docs/screenshots/lab1-cloudwatch-tool-trace.png)

### Pattern 2 — SDK-driven agent (Strands Agents SDK)

Same underlying idea, but the loop itself is handled by the Strands SDK — you declare a model, a system prompt, and a set of tools, and the framework manages reasoning, tool dispatch, and (with a session manager) conversation memory. This version adds:
- **RAG**: a Bedrock Knowledge Base indexed from private S3 data (tour operator listings), queried via Strands' built-in `retrieve` tool
- **MCP**: dynamic tool discovery from an external MCP server (via Bedrock AgentCore Gateway, OAuth2/Cognito-secured), so the agent picks up new tools without redeploying

```python
# Simplified — see the AWS "Building GenAI Applications with AI Agents" workshop
# this lab is based on for the full, deployable version.
travel_agent = Agent(
    model="us.amazon.nova-lite-v1:0",
    system_prompt=TRAVEL_AGENT_PROMPT,
    tools=[flight_search, http_request, retrieve, *mcp_tools],
    session_manager=session_manager,  # S3-backed, for cross-session memory
)
response = travel_agent(user_prompt)
```

```mermaid
sequenceDiagram
    participant U as User
    participant A as Strands Agent
    participant KB as Knowledge Base (RAG)
    participant MCP as MCP Gateway
    participant W as Weather API

    U->>A: "What are my options for Seattle?"
    par
        A->>KB: retrieve(tour operators)
        KB-->>A: relevant docs
    and
        A->>MCP: list & call remote tools
        MCP-->>A: attraction / booking data
    and
        A->>W: http_request(forecast)
        W-->>A: weather data
    end
    A->>A: synthesize all tool results
    A-->>U: single natural-language answer
```
**Result:** the agent authenticated against the MCP gateway, pulled live flight options and a multi-day weather forecast, and synthesized both into a single natural-language recommendation — from one prompt, with no explicit orchestration code.

![Strands agent CLI output](docs/screenshots/lab2-strands-agent-cli-output.png)

### Pattern 3 — Managed multi-agent supervisor (Bedrock Agents)

The most abstracted version: a **Supervisor** agent, configured (not coded) in Bedrock Agents, delegates to two specialized collaborators — a Flight Agent and a Weather Agent — and AWS manages the delegation logic internally.

```mermaid
sequenceDiagram
    participant U as User
    participant S as Supervisor Agent
    participant F as Flight Agent
    participant W as Weather Agent

    U->>S: "Plan my SF → LA trip, Dec 1–2"
    par
        S->>F: get flight options
        F-->>S: Delta / United / American + prices
    and
        S->>W: get forecast
        W-->>S: severe thunderstorm warning, Dec 1
    end
    S->>S: synthesize — flag the conflict
    S-->>U: flight options + weather + recommendation to reconsider the date
```
**Result — the key test:** asked to plan a same-day round trip, the Supervisor queried both sub-agents in parallel, and *synthesized their outputs against each other*: it noticed a severe thunderstorm warning from the Weather Agent and proactively flagged the conflict with the Flight Agent's results, recommending reconsidering the travel date rather than just listing both facts side by side.

![Bedrock Agents supervisor chat](docs/screenshots/lab3-bedrock-supervisor-chat.png)

That's the difference this project was built to explore: not just chaining tool calls, but combining their outputs into a judgment a simple pipeline wouldn't produce on its own.

## Comparison

```mermaid
flowchart LR
    subgraph L1["Lab 1 — Manual Loop"]
        direction TB
        A1["Full control"]
        A2["Full responsibility"]
    end
    subgraph L2["Lab 2 — Strands SDK"]
        direction TB
        B1["Framework-managed loop"]
        B2["Fast iteration"]
    end
    subgraph L3["Lab 3 — Bedrock Agents"]
        direction TB
        C1["AWS-managed orchestration"]
        C2["Least visibility into the loop"]
    end
    L1 -- less code --> L2 -- less code --> L3
```
| Lab | Pattern | Orchestration method | Key learning |
|---|---|---|---|
| 1 | Manual loop | Standard code (Python) + Step Functions | You're responsible for every state, retry, and tool call — high control, high complexity. |
| 2 | Agentic SDK | Strands SDK | The library manages the loop and memory — clean code, fast development. |
| 3 | Managed service | Bedrock Agents | AWS manages the supervisor and sub-agents — high scalability, lower visibility into the loop. |

## Tech stack

- **Models:** Amazon Bedrock — Amazon Nova Lite, Anthropic Claude 3.5 Haiku/Sonnet
- **Compute:** AWS Lambda, AWS Step Functions
- **Agent framework:** Strands Agents SDK
- **Knowledge & memory:** Bedrock Knowledge Bases (RAG, S3-backed), S3 session storage
- **Tool connectivity:** Model Context Protocol (MCP) via Bedrock AgentCore Gateway
- **Multi-agent orchestration:** Amazon Bedrock Agents (Supervisor pattern)
- **API & auth:** Amazon API Gateway, Amazon Cognito
- **Observability:** Amazon CloudWatch (Application Signals / X-Ray trace maps)

## Repository structure

```
.
├── README.md
├── lab-1-manual-loop/        # Step Functions state machine + Lambda reasoning loop
├── lab-2-strands-sdk/        # Strands agent, RAG, MCP client integration
├── lab-3-bedrock-agents/     # Supervisor + collaborator agent configuration
└── docs/
    ├── architecture-notes.md
    └── screenshots/
```

## What I'd explore next

- Caching the MCP OAuth2 token instead of fetching it on every invocation (Lab 2 currently re-authenticates per request)
- Extending the flight/weather tools beyond their current hardcoded destinations
- Adding automated evaluation of agent responses rather than manual console testing

## Attribution

The three labs are based on AWS's official agentic AI workshops (Strands Agents SDK, Amazon Bedrock, AWS Step Functions, Amazon Bedrock Agents). The architecture comparison, testing, and analysis above are my own.

## About

Built by [Mayu Kataoka](https://github.com/mayukataoka) — quality engineering background, exploring applied AI/agent systems.
