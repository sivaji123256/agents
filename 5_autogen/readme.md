# **5_AUTOGEN - TECHNICAL DOCUMENTATION**

## **OVERVIEW**

### What is AutoGen?

AutoGen is a framework for building **multi-agent systems** with two main components:

1. **AgentChat** - High-level, easy-to-use agent framework (similar to CrewAI)
2. **AutoGen Core** - Low-level, flexible runtime infrastructure (similar to LangGraph)

**Key Insight**: AutoGen bridges simplicity (AgentChat) with power (Core Runtime).

---

## **CORE COMPONENTS**

### 1. **Model Clients**

AutoGen abstracts away LLM details:

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.models.ollama import OllamaChatCompletionClient

# OpenAI
model_client = OpenAIChatCompletionClient(model="gpt-4o-mini")

# Local (Ollama)
model_client = OllamaChatCompletionClient(model="llama3.2")
```

### 2. **Messages**

```python
from autogen_agentchat.messages import TextMessage, MultiModalMessage
from autogen_core import Image as AGImage

# Text message
message = TextMessage(content="I'd like to go to London", source="user")

# Multi-modal (text + images)
multi_modal = MultiModalMessage(
    content=["Describe this image", image_object],
    source="User"
)
```

### 3. **Agents**

```python
from autogen_agentchat.agents import AssistantAgent

agent = AssistantAgent(
    name="airline_agent",
    model_client=model_client,
    system_message="You are a helpful assistant for an airline.",
    tools=[some_tool],
    reflect_on_tool_use=True,  # Agent reviews its own tool use
    output_content_type=StructuredModel,  # Pydantic model
    model_client_stream=True  # Stream responses
)
```

### 4. **Tools**

```python
def get_city_price(city_name: str) -> float:
    """Get the roundtrip ticket price to a city"""
    # Implementation
    return price

# Tools are just functions! AutoGen converts them automatically
agent = AssistantAgent(
    name="agent",
    model_client=model_client,
    tools=[get_city_price]  # Just pass the function
)
```

---

## **LAB 1: AGENTCHAT BASICS**

### Learning Objectives
- Model clients
- Messages
- Basic agents
- Tool integration with database

### Quick Start Code

```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_agentchat.agents import AssistantAgent
from autogen_agentchat.messages import TextMessage
from autogen_core import CancellationToken

# 1. Create model client
model_client = OpenAIChatCompletionClient(model="gpt-4o-mini")

# 2. Create agent
agent = AssistantAgent(
    name="airline_agent",
    model_client=model_client,
    system_message="You are helpful. Give short, humorous answers.",
)

# 3. Create message
message = TextMessage(content="I'd like to go to London", source="user")

# 4. Send message
response = await agent.on_messages([message], cancellation_token=CancellationToken())
print(response.chat_message.content)
```

### Database Integration Pattern

```python
import sqlite3

# Setup database
conn = sqlite3.connect("tickets.db")
c = conn.cursor()
c.execute("CREATE TABLE cities (city_name TEXT PRIMARY KEY, round_trip_price REAL)")
conn.commit()
conn.close()

# Helper functions
def save_city_price(city_name, round_trip_price):
    conn = sqlite3.connect("tickets.db")
    c = conn.cursor()
    c.execute("REPLACE INTO cities (city_name, round_trip_price) VALUES (?, ?)", 
              (city_name.lower(), round_trip_price))
    conn.commit()
    conn.close()

def get_city_price(city_name: str) -> float | None:
    """Get the roundtrip ticket price to travel to the city"""
    conn = sqlite3.connect("tickets.db")
    c = conn.cursor()
    c.execute("SELECT round_trip_price FROM cities WHERE city_name = ?", 
              (city_name.lower(),))
    result = c.fetchone()
    conn.close()
    return result[0] if result else None

# Populate database
save_city_price("London", 299)
save_city_price("Paris", 399)
save_city_price("Rome", 499)
```

### Tool-Using Agent

```python
agent = AssistantAgent(
    name="smart_airline_agent",
    model_client=model_client,
    system_message="You are helpful. Include the price of a roundtrip ticket.",
    tools=[get_city_price],  # Pass function directly
    reflect_on_tool_use=True  # Agent reflects on tool results
)

# Agent automatically:
# 1. Converts function to tool schema
# 2. Calls function when needed
# 3. Reflects on results (if reflect_on_tool_use=True)
```

---

## **LAB 2: ADVANCED AGENTCHAT**

### Learning Objectives
- Multi-modal messages (images)
- Structured outputs
- LangChain tool adapter
- Team interactions
- MCP (Model Context Protocol)

### Multi-Modal Example

```python
from autogen_agentchat.messages import MultiModalMessage
from autogen_core import Image as AGImage
from PIL import Image
import requests
from io import BytesIO

# Download image
url = "https://example.com/image.jpg"
pil_image = Image.open(BytesIO(requests.get(url).content))
ag_image = AGImage(pil_image)

# Create multi-modal message
multi_modal = MultiModalMessage(
    content=["Describe this image in detail", ag_image],
    source="User"
)

# Send to agent
agent = AssistantAgent(
    name="describer",
    model_client=model_client,
    system_message="You are good at describing images"
)
response = await agent.on_messages([multi_modal], cancellation_token=CancellationToken())
print(response.chat_message.content)
```

### Structured Outputs

```python
from pydantic import BaseModel, Field
from typing import Literal

class ImageDescription(BaseModel):
    scene: str = Field(description="Overall scene of the image")
    message: str = Field(description="Point the image conveys")
    style: str = Field(description="Artistic style")
    orientation: Literal["portrait", "landscape", "square"] = Field(
        description="Image orientation"
    )

# Agent returns structured data, not string!
agent = AssistantAgent(
    name="describer",
    model_client=model_client,
    system_message="Describe images in detail",
    output_content_type=ImageDescription  # Enforce structure
)

response = await agent.on_messages([multi_modal], cancellation_token=CancellationToken())
result = response.chat_message.content
print(result.scene)  # Access as object!
print(result.orientation)
```

### LangChain Tool Adapter

```python
from autogen_ext.tools.langchain import LangChainToolAdapter
from langchain_community.utilities import GoogleSerperAPIWrapper
from langchain.agents import Tool

# Create LangChain tool
serper = GoogleSerperAPIWrapper()
langchain_tool = Tool(
    name="internet_search",
    func=serper.run,
    description="Search the internet"
)

# Adapt to AutoGen
autogen_tool = LangChainToolAdapter(langchain_tool)

# Use in agent
agent = AssistantAgent(
    name="searcher",
    model_client=model_client,
    tools=[autogen_tool],
    reflect_on_tool_use=True
)
```

### Team Interactions

```python
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import TextMentionTermination

# Create agents
primary = AssistantAgent(
    "primary",
    model_client=model_client,
    tools=[autogen_tool],
    system_message="Research and find good deals"
)

evaluator = AssistantAgent(
    "evaluator",
    model_client=model_client,
    system_message="Provide feedback. Reply with 'APPROVE' when satisfied"
)

# Create team
termination = TextMentionTermination("APPROVE")
team = RoundRobinGroupChat(
    [primary, evaluator],
    termination_condition=termination,
    max_turns=20  # Prevent infinite loops
)

# Run team
result = await team.run(task="Find a flight from JFK to LHR in June 2025")
for message in result.messages:
    print(f"{message.source}: {message.content}")
```

**RoundRobinGroupChat Pattern:**
```
Round 1: Agent1 speaks → Agent2 responds
Round 2: Agent1 speaks → Agent2 responds
...
Until termination condition met
```

### MCP (Model Context Protocol)

```python
from autogen_ext.tools.mcp import StdioServerParams, mcp_server_tools

# Get tools from MCP server
fetch_server = StdioServerParams(
    command="uvx",
    args=["mcp-server-fetch"],
    read_timeout_seconds=30
)
mcp_tools = await mcp_server_tools(fetch_server)

# Use MCP tools in agent
agent = AssistantAgent(
    name="fetcher",
    model_client=model_client,
    tools=mcp_tools,
    reflect_on_tool_use=True
)

# Agent can fetch web content via MCP
result = await agent.run(task="Summarize edwarddonner.com")
```

---

## **LAB 3: AUTOGEN CORE (STANDALONE RUNTIME)**

### Learning Objectives
- AutoGen Core architecture
- Custom message types
- RoutedAgent
- SingleThreadedAgentRuntime
- Inter-agent communication

### Architecture

**AgentChat** (Batteries included)
```
AssistantAgent → On Messages → LLM → Result
```

**AutoGen Core** (Framework + flexibility)
```
RoutedAgent (you define)
    ↓
Message Handler (you decide what to do)
    ↓
Runtime (SingleThreaded or Distributed)
    ↓
Agent-to-Agent Communication
```

### Custom Message Type

```python
from dataclasses import dataclass

@dataclass
class Message:
    content: str
```

### RoutedAgent

```python
from autogen_core import RoutedAgent, MessageContext, message_handler, AgentId

class SimpleAgent(RoutedAgent):
    def __init__(self):
        super().__init__("SimpleAgent")  # Agent type name
    
    @message_handler  # Marks this as a message handler
    async def on_my_message(self, message: Message, ctx: MessageContext) -> Message:
        """Handle incoming messages"""
        return Message(content=f"You said '{message.content}' and I disagree.")
```

**Key Concepts:**
- **agent.id.type**: Type of agent ("SimpleAgent")
- **agent.id.key**: Unique identifier (e.g., "default")
- **@message_handler**: Async method that handles messages
- **MessageContext**: Context with cancellation token

### SingleThreadedAgentRuntime

```python
from autogen_core import SingleThreadedAgentRuntime

# Create runtime
runtime = SingleThreadedAgentRuntime()

# Register agent type
await SimpleAgent.register(
    runtime,
    "simple_agent",  # Agent type name
    lambda: SimpleAgent()  # Factory function
)

# Start runtime (processes messages in background)
runtime.start()

# Send message
agent_id = AgentId("simple_agent", "default")
response = await runtime.send_message(Message("Hello!"), agent_id)
print(response.content)

# Stop gracefully
await runtime.stop()
await runtime.close()
```

### LLM Agent with Core

```python
class MyLLMAgent(RoutedAgent):
    def __init__(self):
        super().__init__("LLMAgent")
        model_client = OpenAIChatCompletionClient(model="gpt-4o-mini")
        # Delegate to AssistantAgent
        self._delegate = AssistantAgent("LLMAgent", model_client=model_client)
    
    @message_handler
    async def handle_message(self, message: Message, ctx: MessageContext) -> Message:
        # Convert to AgentChat format
        text_message = TextMessage(content=message.content, source="user")
        
        # Use delegate
        response = await self._delegate.on_messages(
            [text_message],
            ctx.cancellation_token
        )
        
        # Convert back to our Message type
        return Message(content=response.chat_message.content)
```

### Inter-Agent Communication

```python
class JudgeAgent(RoutedAgent):
    def __init__(self):
        super().__init__("Judge")
    
    @message_handler
    async def judge(self, message: Message, ctx: MessageContext) -> Message:
        # Send messages to other agents
        player1_id = AgentId("player1", "default")
        player2_id = AgentId("player2", "default")
        
        response1 = await self.send_message(
            Message("Make choice 1"),
            player1_id
        )
        response2 = await self.send_message(
            Message("Make choice 2"),
            player2_id
        )
        
        # Judge the results
        result = f"Player 1: {response1.content}\nPlayer 2: {response2.content}"
        return Message(content=result)
```

**Rock-Paper-Scissors Example:**
```python
# Judge sends instructions to both players
# Players respond independently
# Judge evaluates and decides winner
```

---

## **LAB 4: AUTOGEN CORE (DISTRIBUTED RUNTIME)**

### Learning Objectives
- Distributed runtime architecture
- gRPC communication
- Multiple workers
- Scalable agent deployment

### Architecture

**SingleThreaded** (Local):
```
All agents in one process
```

**Distributed** (Scalable):
```
Host (GrpcWorkerAgentRuntimeHost)
    ↓
Worker 1 (Player1Agent) -- gRPC
Worker 2 (Player2Agent) -- gRPC
Worker 3 (JudgeAgent) -- gRPC

All communicate via gRPC network
```

### Host and Workers

```python
from autogen_ext.runtimes.grpc import GrpcWorkerAgentRuntimeHost, GrpcWorkerAgentRuntime

# Create host (listens for agents)
host = GrpcWorkerAgentRuntimeHost(address="localhost:50051")
host.start()

# Create worker 1
worker1 = GrpcWorkerAgentRuntime(host_address="localhost:50051")
await worker1.start()
await Player1Agent.register(worker1, "player1", lambda: Player1Agent("player1"))

# Create worker 2
worker2 = GrpcWorkerAgentRuntime(host_address="localhost:50051")
await worker2.start()
await Player2Agent.register(worker2, "player2", lambda: Player2Agent("player2"))

# Create worker 3
worker3 = GrpcWorkerAgentRuntime(host_address="localhost:50051")
await worker3.start()
await JudgeAgent.register(worker3, "judge", lambda: JudgeAgent("judge"))

# Send message to any worker
response = await worker1.send_message(Message("go"), AgentId("judge", "default"))

# Cleanup
await worker1.stop()
await worker2.stop()
await worker3.stop()
await host.stop()
```

### All-in-One Worker

```python
# Alternative: All agents in one worker
worker = GrpcWorkerAgentRuntime(host_address="localhost:50051")
await worker.start()

await Player1Agent.register(worker, "player1", lambda: Player1Agent("player1"))
await Player2Agent.register(worker, "player2", lambda: Player2Agent("player2"))
await JudgeAgent.register(worker, "judge", lambda: JudgeAgent("judge"))

# Single worker handles all agents
response = await worker.send_message(Message("go"), AgentId("judge", "default"))
```

---

## **SYNTAX REFERENCE**

### Model Client
```python
from autogen_ext.models.openai import OpenAIChatCompletionClient
from autogen_ext.models.ollama import OllamaChatCompletionClient

openai_client = OpenAIChatCompletionClient(model="gpt-4o-mini")
ollama_client = OllamaChatCompletionClient(model="llama3.2")
```

### Messages
```python
from autogen_agentchat.messages import TextMessage, MultiModalMessage
from autogen_core import Image as AGImage

text_msg = TextMessage(content="...", source="user")
multi_msg = MultiModalMessage(content=["text", image], source="user")
```

### Agents
```python
from autogen_agentchat.agents import AssistantAgent

agent = AssistantAgent(
    name="name",
    model_client=model_client,
    system_message="...",
    tools=[tool1, tool2],
    reflect_on_tool_use=True,
    output_content_type=StructuredModel
)

# Invoke
response = await agent.on_messages([message], cancellation_token=CancellationToken())
final_output = response.chat_message.content
```

### Tools
```python
# As function
def my_tool(param: str) -> str:
    """Tool description"""
    return "result"

agent = AssistantAgent(tools=[my_tool])

# As LangChain adapter
from autogen_ext.tools.langchain import LangChainToolAdapter
autogen_tool = LangChainToolAdapter(langchain_tool)

agent = AssistantAgent(tools=[autogen_tool])

# As MCP
from autogen_ext.tools.mcp import StdioServerParams, mcp_server_tools
mcp_tools = await mcp_server_tools(StdioServerParams(...))

agent = AssistantAgent(tools=mcp_tools)
```

### Teams
```python
from autogen_agentchat.teams import RoundRobinGroupChat
from autogen_agentchat.conditions import TextMentionTermination

termination = TextMentionTermination("DONE")
team = RoundRobinGroupChat(
    [agent1, agent2],
    termination_condition=termination,
    max_turns=20
)

result = await team.run(task="task description")
```

### AutoGen Core
```python
from autogen_core import RoutedAgent, MessageContext, message_handler, AgentId
from autogen_core import SingleThreadedAgentRuntime

class MyAgent(RoutedAgent):
    def __init__(self):
        super().__init__("MyAgent")
    
    @message_handler
    async def handler(self, message: Message, ctx: MessageContext) -> Message:
        return Message("response")

# Standalone
runtime = SingleThreadedAgentRuntime()
await MyAgent.register(runtime, "my_agent", lambda: MyAgent())
runtime.start()
response = await runtime.send_message(Message("go"), AgentId("my_agent", "default"))
await runtime.stop()

# Distributed
from autogen_ext.runtimes.grpc import GrpcWorkerAgentRuntimeHost, GrpcWorkerAgentRuntime
host = GrpcWorkerAgentRuntimeHost(address="localhost:50051")
host.start()
worker = GrpcWorkerAgentRuntime(host_address="localhost:50051")
await worker.start()
# Register and use same as above
```

---

## **COMPARISON TABLE**

| Aspect | AgentChat | AutoGen Core |
|--------|-----------|--------------|
| **Level** | High-level | Low-level |
| **Ease** | Easy | Flexible |
| **Best For** | Simple tasks | Complex orchestration |
| **Customization** | Limited | Full |
| **Distributed** | Via Teams | Via gRPC |
| **Learning Curve** | Low | Steep |

---

## **AUTOGEN VS OTHER FRAMEWORKS**

| Framework | Strength | Weakness |
|-----------|----------|----------|
| **AutoGen** | Two-level (easy + powerful) | Newer, less adoption |
| **CrewAI** | Config-based, simple | Less flexible |
| **LangGraph** | Graph-based control | Code-heavy |
| **OpenAI Agents SDK** | Native OpenAI integration | OpenAI only |

---

## **KEY PATTERNS**

### 1. **Tool-Using Agent**
```python
agent = AssistantAgent(
    tools=[get_price, search_web],
    reflect_on_tool_use=True
)
```

### 2. **Multi-Agent Team**
```python
team = RoundRobinGroupChat(
    [researcher, evaluator],
    termination_condition=TextMentionTermination("APPROVE")
)
```

### 3. **Structured Output**
```python
agent = AssistantAgent(
    output_content_type=OutputModel
)
response = await agent.on_messages([msg])
typed_result = response.chat_message.content  # Is OutputModel
```

### 4. **Multi-Modal Input**
```python
multi = MultiModalMessage(
    content=["Analyze this", image],
    source="user"
)
```

### 5. **Inter-Agent Communication (Core)**
```python
response = await self.send_message(message, other_agent_id)
```

---

## **BEST PRACTICES**

1. **Use reflect_on_tool_use for better results**
   ```python
   agent = AssistantAgent(reflect_on_tool_use=True)
   ```

2. **Set max_turns to prevent infinite loops**
   ```python
   team = RoundRobinGroupChat(..., max_turns=20)
   ```

3. **Use structured outputs for type safety**
   ```python
   agent = AssistantAgent(output_content_type=MyModel)
   ```

4. **Adapt LangChain tools for reuse**
   ```python
   autogen_tool = LangChainToolAdapter(langchain_tool)
   ```

5. **Use distributed runtime for scalability**
   ```python
   # Not just SingleThreaded for production
   worker = GrpcWorkerAgentRuntime(...)
   ```

---

## **WHEN TO USE EACH**

### Use **AgentChat** for:
- Simple multi-agent tasks
- Teams with clear roles
- Quick prototypes
- Educational projects

### Use **AutoGen Core** for:
- Custom agent types
- Complex communication patterns
- Inter-agent messaging
- Distributed systems
- Production deployments

---

## **KEY TAKEAWAYS**

✓ **AgentChat** = High-level, simple teams
✓ **AutoGen Core** = Low-level, powerful runtime
✓ **Model Clients** abstract LLM details
✓ **Tools** are just Python functions (auto-converted)
✓ **Messages** can be text or multi-modal
✓ **Agents** handle their own logic
✓ **Teams** enable multi-agent coordination
✓ **RoundRobinGroupChat** = turn-based interaction
✓ **RoutedAgent** = custom agent types (Core)
✓ **Distributed Runtime** = gRPC-based scaling
✓ **LangChain Adapter** = reuse existing tools
✓ **MCP Support** = Model Context Protocol integration
✓ **Structured Outputs** = Type-safe responses
✓ **reflect_on_tool_use** = Better tool results
✓ **Two-level design** = Easy + Powerful
