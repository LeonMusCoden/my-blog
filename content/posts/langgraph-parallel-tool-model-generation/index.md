---
title: 'LangGraph parallel tool execution and LLM generation'
date: '2025-04-30'
draft: false
categories:
  - default
tags:
  - LLM
  - LangGraph
---

Recently, I was working with [LangGraph](https://langchain-ai.github.io/langgraph/) to build a chatbot, and encountered a limitation: **You cannot invoke an LLM before completing all open toolcalls**.

I found a way to work around that, and I'm sharing my solution since I couldn't find anything online addressing this issue.

# The Problem: Sequential Execution Bottleneck

My chatbot needed to generate charts as part of its reponses. I wanted to have both a chart and an accompaning text by an LLM.

My initial implementation had a straightforward but slow flow:

1. LLM in *model* node makes a toolcall to generate a chart
2. Conditional edge directs to *chart* node
3. *chart* node runs the chart generation and inserts a ToolMessage with the result
4. Execution returns to *model* node and LLM is invoked again

The problem with this approach is that users had to wait for the chart to fully generate *before* the LLM could continue generating text. Ideally, chart generation and text generation should happen in parallel, reducing the total response time.

# The Solution: Placeholder Messages and Parallel Execution

I wanted to keep using toolcalls since they're the safest way to have an LLM interact with tools. My solution uses placeholder messages to enable parallelization:

## The Architecture

1. When the LLM makes a chart toolcall, the *chart* node creates an placeholder ToolMessage
2. The graph then branches execution to both:
   - Return to the *model* node so the LLM can continue generating text
   - Send to a new *update_chart* node that handles the actual chart generation
3. The *update_chart* node replaces the placeholder with the final chart result when ready

The result: The LLM generates text while the chart is being created in the background.

## Implementation Details

Here's how the code works:

```py
async def chart(self, state: State) -> State:
    """Schedule chart generation and create filler messages."""
    if not state.messages or not isinstance(state.messages[-1], AIMessage):
        return state

    tool_calls = cast(AIMessage, state.messages[-1]).tool_calls
    # Collect all tool calls with name "chart"
    chart_calls = [call for call in tool_calls if call.get("name") == "chart"]
    if not chart_calls:
        return state
    new_messages = []
    for chart_call in chart_calls:
        params = chart_call.get("args", {})
        task = params.get("task")
        if task is None:
            raise ValueError("Missing params for chart tool call")
        tool_call_id = chart_call.get("id")

        # Build a temporary artifact to pass the necessary info along the branch.
        artifact = {"task": task, "tool_call_id": tool_call_id}
        # Return a filler message immediately.
        filler_msg = ToolMessage(
            content="Generating chart.",
            tool_call_id=tool_call_id,
            artifact=artifact,
        )
        new_messages.append(filler_msg)
    return {"messages": new_messages}
```

The *chart* node doesn't actually generate the chart - it just creates a placeholder message with all the data required for generating the chart. The actual chart generation happens in the *update_chart* node:

```py
async def update_chart(self, state: State):
    """Replace the temporary ToolMessage with the final result."""
    for i, msg in enumerate(state.messages):
        if isinstance(msg, ToolMessage) and msg.tool_call_id is not None:
            artifact = msg.artifact
            if isinstance(artifact, dict) and "task" in artifact:
                task = artifact["task"]
                tool_call_id = artifact["tool_call_id"]
                status: Literal["success", "error"] = "success"
                try:
                    # Run the chart subgraph using the provided parameters.
                    chart_state: ChartCreationState = ChartCreationState(
                        task=task
                    )
                    chart_state = _safe_init_dataclass(
                        ChartCreationState,
                        await self.chart_graph.ainvoke(
                            chart_state, self.config.__dict__
                        ),
                    )
                    fig = chart_state.fig
                    new_content = "Graph generated successfully."
                except Exception as e:
                    print(str(e))
                    new_content = "Error generating chart, please reply to the user via text instead. You must start your new message with 'Sorry, I encountered an error. '"
                    status = "error"
                    fig = None

                # Replace the temporary ToolMessage with the updated one.
                state.messages[i] = ToolMessage(
                    content=new_content,
                    tool_call_id=tool_call_id,
                    artifact=plotly.io.to_json(fig) if fig is not None else None,
                    status=status,
                    name="chart",
                )
```

## Handling Errors Gracefully

What's interesting about this is how the chatbot handles errors. If chart generation fails, the content of the ToolMessage contains instructions on how to handle the error. We can then use a conditional edge to invoke the LLM again to respond to the user accordingly:

```py
def _should_prompt_model_again(self, state: State):
    if (
        not isinstance(state.messages[-1], ToolMessage)
        or state.messages[-1].artifact
    ):
        return END

    return "model"
```

With this conditional routing, a failing chart generation follows this improved flow:

1. LLM makes a toolcall to generate a chart
2. *chart* node inserts a placeholder message
3. Execution branches to:
   - *model* node where the LLM continues generating text assuming the chart will be available
   - *update_chart* node which attempts to generate the chart but fails and inserts error handling instructions
4. Conditional edge redirects to *model* node
5. LLM apologizes and responds using text instead of a chart

# Conclusion

This pattern allows for much more responsive LLM applications. The user experience is improved significantly as they don't have to wait for long-running tools to complete before seeing any text response.
