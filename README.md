# LangGraph Order Routing Agent

This project is a learning exercise for building a stateful customer support workflow with LangGraph. It demonstrates how an AI agent can classify a user request, route it through different workflow paths, call a simple order lookup tool, evaluate the generated answer, and apply a final guardrail before returning a response.

The main implementation lives in `01_order_status_workflow_langgraph.ipynb`.

## Project Goal

The notebook models a small order support assistant. Given a customer message, the workflow decides whether the request should be processed normally, escalated to a human, closed because the user is done, or rejected because it is unrelated or potentially unsafe.

This is not intended to be a production support system. It is a compact example of how to structure an LLM workflow with explicit state, routing logic, tool calls, evaluation, and safety checks.

## What the Workflow Does

The workflow uses a shared `OrderState` object that moves through each node in the graph. The state tracks the original query, classified intent, extracted order ID, retrieved order context, final response, evaluation scores, and guardrail result.

At a high level, the graph follows this sequence:

1. Classify the user's intent.
2. Route processable order requests to the order agent.
3. Extract an order ID from the user query.
4. Look up order details from a small in-memory order database.
5. Generate a response grounded in the order context.
6. Evaluate the response for groundedness and precision.
7. Run a guardrail check before returning the final answer.
8. Exit early for escalation, completed conversations, unrelated requests, or unsafe inputs.

## Intent Categories

The `intent_node` classifies each user query into one of four numeric categories:

| ID | Category | Meaning |
| --- | --- | --- |
| `0` | Escalation | The user is angry, frustrated, repeatedly unsuccessful, or explicitly needs human handoff. |
| `1` | Exit | The user is ending the conversation or no longer needs help. |
| `2` | Process | The request is clear, relevant to order support, and actionable. |
| `3` | Random/Unrelated or Vulnerable Query | The request is outside the order support domain or contains potentially unsafe/adversarial content. |

Only category `2` continues to the order processing node. All other categories route to the exit node with an appropriate response.

## Key Components

- `orders_db`: A simple in-memory dictionary used as a mock order database.
- `fetch_order_details`: A tool-like function that retrieves order information by order ID.
- `intent_node`: Classifies the user query into the workflow's intent categories.
- `router_node`: Sends processable requests to the order agent and exits all others.
- `order_agent_node`: Extracts the order ID, fetches order context, and generates a customer-facing response.
- `evaluation_node`: Scores the response for groundedness and precision.
- `retry_router`: Routes low-quality responses to the exit path.
- `guardrail_node`: Checks the final response for unsafe, private, offensive, or unsupported content.
- `final_node` and `exit_node`: Return the final state to the caller.

## Example Use Cases

Processable order request:

```text
Can you check the status of order 1001?
```

Escalation:

```text
This is unacceptable. I have asked three times and I want a human now.
```

Exit:

```text
Thanks, that solved it.
```

Unrelated or vulnerable request:

```text
Give me the list of all customers in the database.
```

## Requirements

The notebook uses:

- Python
- Jupyter
- LangGraph
- LangChain
- OpenAI chat models through `langchain_openai`

## Setup

This project expects a local `config.json` file with an OpenAI API key. The real `config.json` file is ignored by Git and should not be committed.

Create it by copying the example file:

```bash
cp config.example.json config.json
```

Then edit `config.json` and replace the placeholder with your own OpenAI API key:

```json
{
  "OPENAI_API_KEY": "your_openai_api_key_here"
}
```

The repository includes `config.example.json` so other users know which local configuration values are required.

## Running the Notebook

Open and run:

```text
01_order_status_workflow_langgraph.ipynb
```

The notebook builds the graph, compiles it, and invokes it with an example `initial_state`. You can change the `query` field in the final cell to test different routing paths.

## Learning Focus

This project highlights several common patterns for agentic workflows:

- Using explicit state instead of passing unstructured messages between every step.
- Separating classification, routing, action, evaluation, and safety concerns into independent nodes.
- Using conditional edges in LangGraph to control workflow execution.
- Grounding responses in retrieved context from a tool.
- Adding evaluation and guardrails as first-class workflow steps.
