# LangGraph Framework - Technical Reference

---

## **OVERVIEW**

### What is LangGraph?

LangGraph is a framework for building **graph-based agent systems** where:
- Agents execute nodes in a **directed acyclic graph (DAG)**
- **State** is maintained and passed between nodes
- **Reducers** automatically merge state updates
- **Checkpoints** enable memory persistence across invocations
- **Conditional routing** controls flow based on state

---

## **CORE CONCEPTS**

### 1. **State (TypedDict or BaseModel)**

State is the shared data structure flowing through the graph:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph.message import add_messages

# Option 1: TypedDict (preferred)
class State(TypedDict):
    messages: Annotated[list, add_messages]  # Reducer: add_messages
    some_value: str                          # Regular value

# Option 2: Pydantic BaseModel
from pydantic import BaseModel

class State(BaseModel):
    messages: Annotated[list, add_messages]
    some_value: str
```

**Key Concept: Annotated[Type, Reducer]**
- `Annotated` = Python type hints + extra metadata
- Second argument = **reducer function** (how to merge updates)
- `add_messages` = built-in reducer for message lists
- Without `Annotated` = value is replaced (not merged)

#### **Reducers: How State Updates Work**

```python
# Example 1: add_messages reducer
old_state = {"messages": [msg1, msg2]}
new_update = {"messages": [msg3, msg4]}
# Result: messages = [msg1, msg2, msg3, msg4]  ← APPENDED

# Example 2: Regular field (no reducer)
old_state = {"value": "old"}
new_update = {"value": "new"}
# Result: value = "new"  ← REPLACED

# Example 3: Custom reducer
def custom_reducer(old, new):
    return old + new

class State(TypedDict):
    counter: Annotated[int, custom_reducer]
```

### 2. **Nodes (Python Functions)**

A node is any Python function that:
- Takes `State` as input
- Returns updated `State` (or dict that updates state)
- Can be sync or async

```python
# Simple node
def my_node(state: State) -> dict:
    # Do work
    result = "some output"
    
    # Return dict to update state
    return {
        "messages": [{"role": "assistant", "content": result}]
    }

# LLM node
def llm_node(state: State) -> dict:
    response = llm.invoke(state["messages"])
    return {"messages": [response]}

# Tool-using node (with ToolNode)
from langgraph.prebuilt import ToolNode
tool_node = ToolNode(tools=[tool1, tool2])
```

### 3. **Edges (Connections)**

Edges connect nodes and control flow:

```python
from langgraph.graph import START, END

# Simple edge: A → B
graph_builder.add_edge("node_a", "node_b")

# START edge: Begin with node
graph_builder.add_edge(START, "node_a")

# END edge: Exit graph
graph_builder.add_edge("node_z", END)

# Conditional edge: A → B or C (based on state)
def router(state: State) -> str:
    if state["value"] == "x":
        return "node_b"
    else:
        return "node_c"

graph_builder.add_conditional_edges("node_a", router, {
    "node_b": "node_b",
    "node_c": "node_c"
})
```

### 4. **Graph Compilation**

```python
# Create graph
graph = graph_builder.compile()

# With memory (checkpointing)
from langgraph.checkpoint.memory import MemorySaver
memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)

# With SQLite persistence
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3
conn = sqlite3.connect("memory.db", check_same_thread=False)
sql_saver = SqliteSaver(conn)
graph = graph_builder.compile(checkpointer=sql_saver)
```

### 5. **Execution**

```python
# Synchronous
result = graph.invoke(
    {"messages": [{"role": "user", "content": "Hello"}]},
    config={"configurable": {"thread_id": "user123"}}
)

# Asynchronous
result = await graph.ainvoke(
    {"messages": [{"role": "user", "content": "Hello"}]},
    config={"configurable": {"thread_id": "user123"}}
)
```

---

## **LAB 1: BASICS - SIMPLE GRAPH**

### Learning Objectives
- Define State object
- Create nodes
- Add edges
- Compile and invoke graph

### Quick Start Code

```python
from typing import Annotated
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from pydantic import BaseModel

# Step 1: Define State
class State(BaseModel):
    messages: Annotated[list, add_messages]

# Step 2: Create graph builder
graph_builder = StateGraph(State)

# Step 3: Add node (simple function)
def first_node(state: State) -> State:
    reply = "Hello from node!"
    messages = [{"role": "assistant", "content": reply}]
    return State(messages=messages)

graph_builder.add_node("first_node", first_node)

# Step 4: Add edges
graph_builder.add_edge(START, "first_node")
graph_builder.add_edge("first_node", END)

# Step 5: Compile
graph = graph_builder.compile()

# Execute
result = graph.invoke(State(messages=[{"role": "user", "content": "Hi"}]))
print(result["messages"][-1].content)
```

### Key Patterns

#### **Non-LLM Example (Random adjectives)**
```python
import random

nouns = ["Cabbages", "Unicorns", "Toasters"]
adjectives = ["outrageous", "smelly", "pedantic"]

def random_node(state: State) -> State:
    reply = f"{random.choice(nouns)} are {random.choice(adjectives)}"
    return State(messages=[{"role": "assistant", "content": reply}])
```

**Insight**: LangGraph works with ANY Python functions, not just LLMs!

#### **LLM Node**
```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

def chatbot_node(state: State) -> State:
    response = llm.invoke(state.messages)
    return State(messages=[response])
```

#### **Gradio Chat Interface**
```python
import gradio as gr

def chat(user_input: str, history):
    initial_state = State(messages=[{"role": "user", "content": user_input}])
    result = graph.invoke(initial_state)
    return result['messages'][-1].content

gr.ChatInterface(chat, type="messages").launch()
```

### Architecture: Simple Flow
```
START → first_node → END
```

---

## **LAB 2: TOOLS & MEMORY**

### Learning Objectives
- Add tools to graph
- Conditional routing
- Checkpointing (memory)
- State persistence

### Tool Integration

#### **Create Tool from Function**
```python
from langchain.agents import Tool

def my_function(input: str) -> str:
    """Do something"""
    return "result"

my_tool = Tool(
    name="my_tool_name",
    func=my_function,
    description="What this tool does"
)
```

#### **Bind Tools to LLM**
```python
from langgraph.prebuilt import ToolNode, tools_condition

llm = ChatOpenAI(model="gpt-4o-mini")
tools = [search_tool, notification_tool]
llm_with_tools = llm.bind_tools(tools)
```

#### **Add Tool Node to Graph**
```python
from langgraph.prebuilt import ToolNode

def chatbot(state: State) -> dict:
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools=tools))

# Conditional routing
graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,  # Built-in: checks if tool_calls exist
    {"tools": "tools", END: END}
)

# After tool use, return to chatbot
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
```

### Architecture: Tool Loop
```
START → chatbot ←→ tools
            ↓
           END
```

### Memory/Checkpointing

#### **In-Memory Checkpointer**
```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)

# Pass thread_id in config to persist state
config = {"configurable": {"thread_id": "user123"}}
result = graph.invoke(state, config=config)

# State persists across invocations with same thread_id!
result2 = graph.invoke(new_state, config=config)
# Messages include history from first call
```

#### **SQLite Checkpointer**
```python
from langgraph.checkpoint.sqlite import SqliteSaver
import sqlite3

conn = sqlite3.connect("memory.db", check_same_thread=False)
sql_memory = SqliteSaver(conn)
graph = graph_builder.compile(checkpointer=sql_memory)

# Same usage as MemorySaver, but persistent to disk
config = {"configurable": {"thread_id": "user123"}}
result = graph.invoke(state, config=config)
```

#### **Access State History**
```python
# Get current state
current = graph.get_state(config)

# Get all checkpoint history (most recent first)
history = list(graph.get_state_history(config))

# Time travel: resume from checkpoint
old_config = {
    "configurable": {
        "thread_id": "user123",
        "checkpoint_id": "checkpoint_id_123"
    }
}
result = graph.invoke(None, config=old_config)
```

### Key Pattern: Super-Steps

**Super-Step**: One complete invocation of the graph

```
Invocation 1:
  START → chatbot → tools → chatbot → END
  (Messages update with reducer within this step)

Invocation 2:
  (New super-step with same thread_id)
  (State includes history from invocation 1!)
  START → chatbot → ... → END
```

**Insight**: `invoke()` runs ONE super-step. Checkpointing connects super-steps into a conversation!

---

## **LAB 3: ASYNC & BROWSER AUTOMATION**

### Learning Objectives
- Async/await patterns
- Playwright browser tools
- Web scraping agents
- Real-world tools

### Async Patterns

#### **Async Execution**
```python
# Sync
result = graph.invoke(state)

# Async
result = await graph.ainvoke(state)
```

#### **Async Tools**
```python
# Sync tool
tool.run(input)

# Async tool
await tool.arun(input)
```

#### **Nested Async (Jupyter)**
```python
import nest_asyncio
nest_asyncio.apply()  # Allows nested event loops in Jupyter
```

### Playwright Browser Tools

#### **Setup Browser**
```python
from langchain_community.agent_toolkits import PlayWrightBrowserToolkit
from langchain_community.tools.playwright.utils import create_async_playwright_browser

# Create browser
async_browser = create_async_playwright_browser(headless=False)  # Show UI
toolkit = PlayWrightBrowserToolkit.from_browser(async_browser=async_browser)
tools = toolkit.get_tools()

# Available tools: navigate_browser, extract_text, click, type, etc.
```

#### **Use Browser Tools**
```python
# Get specific tool
navigate_tool = [t for t in tools if t.name == "navigate_browser"][0]
extract_tool = [t for t in tools if t.name == "extract_text"][0]

# Navigate to website
await navigate_tool.arun({"url": "https://www.example.com"})

# Extract text
text = await extract_tool.arun({})
print(text)
```

#### **Browser Tool Integration with Graph**
```python
llm_with_tools = llm.bind_tools(tools)

async def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

# Graph setup same as Lab 2
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_node("tools", ToolNode(tools=tools))
# ... add edges ...

# Execute async
result = await graph.ainvoke(state, config=config)
```

---

## **LAB 4: PRODUCTION - SIDEKICK PROJECT**

### Overview

**Sidekick**: An autonomous assistant that:
1. Takes a **task** and **success criteria**
2. **Works** on the task (using tools)
3. **Evaluates** if success criteria met
4. **Retries** if needed with feedback
5. **Asks user** if stuck

### Architecture

```
              ┌─────────────┐
              │   WORKER    │ (takes action)
              └──────┬──────┘
                     │
          ┌──────────┴──────────┐
          │                     │
       Tools              (success?)
          │                     │
          └──────────┬──────────┘
                     │
                ┌────v─────┐
                │ EVALUATOR │ (checks success)
                └────┬─────┘
                     │
         ┌───────────┼───────────┐
         │           │           │
      Success    More Work   User Input
         │           │           │
         v           v           v
        END       WORKER       DONE
```

### State Design

```python
from typing import Optional

class State(TypedDict):
    messages: Annotated[List[Any], add_messages]  # Conversation history
    success_criteria: str                          # What success looks like
    feedback_on_work: Optional[str]               # If previous attempt failed
    success_criteria_met: bool                    # Evaluator decision
    user_input_needed: bool                       # Need user clarification
```

### Structured Output (Evaluator)

```python
from pydantic import BaseModel, Field

class EvaluatorOutput(BaseModel):
    feedback: str = Field(description="Feedback on response")
    success_criteria_met: bool = Field(description="Met success criteria?")
    user_input_needed: bool = Field(description="Need user input?")

# Use with LLM
evaluator_llm = ChatOpenAI(model="gpt-4o-mini")
evaluator_with_structure = evaluator_llm.with_structured_output(EvaluatorOutput)

# Returns structured object, not string!
result = evaluator_with_structure.invoke(messages)
print(result.success_criteria_met)  # bool
```

### Worker Node

```python
def worker(state: State) -> dict:
    # Build system prompt with success criteria
    system_message = f"""You are a helpful assistant.
Success criteria: {state['success_criteria']}

If task is complete, provide final answer.
If stuck, ask clarifying question."""
    
    # Add feedback if previous attempt failed
    if state.get("feedback_on_work"):
        system_message += f"\nFeedback from previous attempt: {state['feedback_on_work']}"
    
    # Update/create system message in conversation
    messages = add_system_message(state["messages"], system_message)
    
    # Invoke LLM with tools
    response = llm_with_tools.invoke(messages)
    
    return {"messages": [response]}
```

### Router Functions

#### **Worker Router: Tool Use Detection**
```python
def worker_router(state: State) -> str:
    last_message = state["messages"][-1]
    
    # Check if LLM called tools
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"  # Go execute tools
    else:
        return "evaluator"  # Go evaluate response
```

#### **Evaluator Router: Evaluation-Based Routing**
```python
def route_based_on_evaluation(state: State) -> str:
    if state["success_criteria_met"] or state["user_input_needed"]:
        return "END"  # Task done or need user input
    else:
        return "worker"  # Continue working
```

### Evaluator Node

```python
def evaluator(state: State) -> dict:
    last_response = state["messages"][-1].content
    
    # Build evaluation prompt
    evaluation_prompt = f"""
Evaluate if this response meets the success criteria:
{state['success_criteria']}

Response: {last_response}

Conversation history: {format_conversation(state['messages'])}
"""
    
    # Get structured evaluation
    eval_result = evaluator_with_structure.invoke([
        SystemMessage(content="You are an evaluator..."),
        HumanMessage(content=evaluation_prompt)
    ])
    
    return {
        "messages": [{"role": "assistant", "content": f"Feedback: {eval_result.feedback}"}],
        "feedback_on_work": eval_result.feedback,
        "success_criteria_met": eval_result.success_criteria_met,
        "user_input_needed": eval_result.user_input_needed
    }
```

### Full Graph Assembly

```python
graph_builder = StateGraph(State)

# Add nodes
graph_builder.add_node("worker", worker)
graph_builder.add_node("tools", ToolNode(tools=tools))
graph_builder.add_node("evaluator", evaluator)

# Add edges
graph_builder.add_edge(START, "worker")
graph_builder.add_conditional_edges("worker", worker_router, {
    "tools": "tools",
    "evaluator": "evaluator"
})
graph_builder.add_edge("tools", "worker")  # Back to worker after tool use
graph_builder.add_conditional_edges("evaluator", route_based_on_evaluation, {
    "worker": "worker",  # Continue if not done
    "END": END
})

# Compile with memory
memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

### Gradio UI

```python
import gradio as gr
import uuid

def make_thread_id():
    return str(uuid.uuid4())

async def process_message(message, success_criteria, history, thread):
    config = {"configurable": {"thread_id": thread}}
    
    state = {
        "messages": [{"role": "user", "content": message}],
        "success_criteria": success_criteria,
        "feedback_on_work": None,
        "success_criteria_met": False,
        "user_input_needed": False
    }
    
    result = await graph.ainvoke(state, config=config)
    
    # Extract messages
    user_msg = {"role": "user", "content": message}
    worker_msg = {"role": "assistant", "content": result["messages"][-2].content}
    eval_msg = {"role": "assistant", "content": result["messages"][-1].content}
    
    return history + [user_msg, worker_msg, eval_msg]

with gr.Blocks() as demo:
    gr.Markdown("## Sidekick Personal Assistant")
    thread = gr.State(make_thread_id())
    
    chatbot = gr.Chatbot(type="messages", height=300)
    message = gr.Textbox(placeholder="Your task...")
    success_criteria = gr.Textbox(placeholder="Success criteria...")
    
    go_button = gr.Button("Go!", variant="primary")
    
    go_button.click(
        process_message,
        [message, success_criteria, chatbot, thread],
        [chatbot]
    )

demo.launch()
```

---

## **SYNTAX REFERENCE**

### Basic Graph Structure
```python
from langgraph.graph import StateGraph, START, END
from typing import Annotated
from langgraph.graph.message import add_messages

# 1. Define State
class State(TypedDict):
    messages: Annotated[list, add_messages]
    custom_field: str

# 2. Create builder
graph_builder = StateGraph(State)

# 3. Define nodes
def node_function(state: State) -> dict:
    return {"custom_field": "updated"}

# 4. Add nodes
graph_builder.add_node("node_name", node_function)

# 5. Add edges
graph_builder.add_edge(START, "node_name")
graph_builder.add_edge("node_name", END)

# 6. Compile
graph = graph_builder.compile()

# 7. Execute
result = graph.invoke({"messages": [...], "custom_field": "..."})
```

### Tools
```python
from langgraph.prebuilt import ToolNode, tools_condition
from langchain.agents import Tool

# Create tool
tool = Tool(name="...", func=function, description="...")

# Bind to LLM
llm_with_tools = llm.bind_tools([tool])

# Add to graph
graph_builder.add_node("tools", ToolNode(tools=[tool]))

# Conditional routing
graph_builder.add_conditional_edges("chatbot", tools_condition, {
    "tools": "tools",
    END: END
})
```

### Conditional Edges
```python
def router_function(state: State) -> str:
    if condition:
        return "node_a"
    else:
        return "node_b"

graph_builder.add_conditional_edges("source_node", router_function, {
    "node_a": "node_a",
    "node_b": "node_b"
})
```

### Memory/Checkpointing
```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver

# In-memory
memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)

# SQLite
import sqlite3
conn = sqlite3.connect("db.db", check_same_thread=False)
saver = SqliteSaver(conn)
graph = graph_builder.compile(checkpointer=saver)

# Use with config
config = {"configurable": {"thread_id": "user_id"}}
result = graph.invoke(state, config=config)
```

### Async
```python
# Async execution
result = await graph.ainvoke(state, config=config)

# Async tools
await tool.arun(input)

# In Jupyter
import nest_asyncio
nest_asyncio.apply()
```

### Structured Output
```python
from pydantic import BaseModel, Field

class Output(BaseModel):
    field1: str = Field(description="...")
    field2: bool = Field(description="...")

llm_structured = llm.with_structured_output(Output)
result = llm_structured.invoke(messages)
# result is Output instance, not string
```

---

## **COMPARISON: LABS 1-4**

| Lab | Focus | Complexity | Key Feature |
|-----|-------|-----------|------------|
| **Lab 1** | Graph basics | Low | Simple node flow |
| **Lab 2** | Tools & memory | Medium | Checkpointing |
| **Lab 3** | Async & browser | Medium-High | Web automation |
| **Lab 4** | Production | High | Worker-Evaluator pattern |

---

## **COMMON PATTERNS**

### 1. **Agent Loop (Auto-Tool Use)**
```
while loop:
  LLM decides
  if tool call:
    execute tool
  else:
    done
```

### 2. **Evaluation Loop (Sidekick)**
```
while loop:
  worker does task
  evaluator checks
  if success:
    done
  elif needs_input:
    ask user
  else:
    feedback → worker
```

### 3. **Multi-Agent Coordination**
```
router → agent_a ↘
              → synthesizer → END
         agent_b ↗
```

---

## **BEST PRACTICES**

1. **Always use reducers for append-only fields**
   ```python
   messages: Annotated[list, add_messages]  # ✓
   messages: list                           # ✗ (overwrites)
   ```

2. **Thread IDs for multi-user systems**
   ```python
   config = {"configurable": {"thread_id": unique_user_id}}
   ```

3. **Use SQLiteSaver for production**
   ```python
   # Not just MemorySaver (which resets on restart)
   saver = SqliteSaver(conn)
   ```

4. **Async for performance**
   ```python
   # Use ainvoke() for parallel tool execution
   result = await graph.ainvoke(state)
   ```

5. **Structured outputs for reliability**
   ```python
   # Don't parse strings, use Pydantic
   llm_with_structure = llm.with_structured_output(OutputModel)
   ```

---

## **KEY DIFFERENCES: LANGGRAPH VS CREWAI**

| Aspect | LangGraph | CrewAI |
|--------|-----------|--------|
| **Model** | Graph-based | Config-based |
| **State** | Explicit | Implicit |
| **Tools** | Via nodes | Declarative |
| **Memory** | Checkpointing | Auto |
| **Learning Curve** | Steeper | Gentler |
| **Control** | Fine-grained | High-level |
| **Scalability** | Better for complex | Better for simple |

---

## **KEY TAKEAWAYS**

✓ **Nodes** are Python functions (sync or async)
✓ **State** flows through graph with reducers
✓ **Annotated** enables automatic state merging
✓ **Conditional edges** control complex flows
✓ **Checkpointing** enables multi-turn conversations
✓ **ToolNode** handles all tool execution automatically
✓ **Structured outputs** ensure type safety
✓ **Async/await** enables parallel execution
✓ **Thread IDs** persist state across invocations
✓ **LangGraph is code-first** (vs config-first in CrewAI)
✓ **Super-steps** connect multiple invocations into conversations
✓ **Worker-Evaluator** pattern enables self-improving systems
