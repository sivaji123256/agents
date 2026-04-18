# Lab 2: Advanced OpenAI Features - Technical Reference

---

## **LAB 1: OpenAI Agents SDK Basics**

### Overview
Introduction to OpenAI's native Agents SDK - a lightweight framework for building agentic applications directly with OpenAI's latest tools.

### Quick Start Code
```python
from dotenv import load_dotenv
from agents import Agent, Runner, trace

load_dotenv(override=True)

# Create agent with name, instructions, and model
agent = Agent(
    name="Jokester",
    instructions="You are a joke teller",
    model="gpt-4o-mini"
)

# Run the agent
with trace("Telling a joke"):
    result = await Runner.run(agent, "Tell a joke about Autonomous AI Agents")
    print(result.final_output)

# View trace at: https://platform.openai.com/traces
```

### Core Concepts

#### **Agent Creation**
```python
agent = Agent(
    name="AgentName",              # Identifier for the agent
    instructions="...",            # System instructions/persona
    model="gpt-4o-mini"            # Model to use
)
```

**Parameters:**
- **`name`**: Display name for the agent
- **`instructions`**: System prompt defining agent behavior
- **`model`**: LLM model (gpt-4o-mini, gpt-4o, etc.)

#### **Runner.run() - Synchronous Execution**
```python
result = await Runner.run(agent, "Your prompt")
response = result.final_output  # Get the response
```

**Returns:**
- `result.final_output`: The agent's text response
- `result`: Full result object with metadata

#### **Runner.run_streamed() - Streaming**
```python
result = Runner.run_streamed(agent, "Your prompt")
async for event in result.stream_events():
    if event.type == "raw_response_event" and isinstance(event.data, ResponseTextDeltaEvent):
        print(event.data.delta, end="", flush=True)  # Stream tokens as they arrive
```

#### **trace() - Observability**
```python
with trace("Trace name"):
    result = await Runner.run(agent, "prompt")
```

**What it does:**
- Records agent execution for viewing at https://platform.openai.com/traces
- Tracks all calls, tokens, latency
- Essential for debugging and monitoring

### Key Takeaways
✓ OpenAI Agents SDK is minimal - just Agent + Runner
✓ Use `trace()` context manager for observability
✓ `await` is required (async/await pattern)
✓ Check traces at platform.openai.com for debugging

---

## **LAB 2: Multi-Agent Workflows with Tools & Handoffs**

### Overview
Build a sales automation system with multiple agents, tools, and agent-to-agent handoffs. Demonstrates agent collaboration patterns.

### Quick Start Code
```python
from agents import Agent, Runner, trace, function_tool
import asyncio

# Define agent with tool
@function_tool
def send_email(body: str):
    """Send email with given body"""
    # Implementation
    return {"status": "success"}

# Create multiple agents with different styles
sales_agent1 = Agent(
    name="Professional Sales Agent",
    instructions="Write professional, serious cold emails",
    model="gpt-4o-mini"
)

sales_agent2 = Agent(
    name="Engaging Sales Agent",
    instructions="Write witty, engaging cold emails",
    model="gpt-4o-mini"
)

sales_agent3 = Agent(
    name="Busy Sales Agent",
    instructions="Write concise, to-the-point emails",
    model="gpt-4o-mini"
)

# Convert agents to tools
tool1 = sales_agent1.as_tool(tool_name="sales_agent1", tool_description="Write a cold sales email")
tool2 = sales_agent2.as_tool(tool_name="sales_agent2", tool_description="Write a cold sales email")
tool3 = sales_agent3.as_tool(tool_name="sales_agent3", tool_description="Write a cold sales email")

tools = [tool1, tool2, tool3, send_email]

# Create manager agent
sales_manager = Agent(
    name="Sales Manager",
    instructions="Generate emails using tools, select best, and send it",
    tools=tools,
    model="gpt-4o-mini"
)

# Execute
result = await Runner.run(sales_manager, "Send cold sales email")
```

### Core Concepts

#### **1. Parallel Agent Execution**
```python
message = "Write a cold sales email"

# Run all 3 agents in parallel
results = await asyncio.gather(
    Runner.run(sales_agent1, message),
    Runner.run(sales_agent2, message),
    Runner.run(sales_agent3, message),
)

# Extract outputs
outputs = [result.final_output for result in results]
```

**Why parallel?**
- Faster than sequential
- Demonstrates agent orchestration
- Pattern: "generate multiple options, pick best"

#### **2. @function_tool Decorator**
```python
@function_tool
def send_email(body: str):
    """Send out an email with the given body"""
    # Implementation here
    return {"status": "success"}
```

**What it does:**
- Automatically converts function to tool schema
- No JSON boilerplate needed
- Type hints become parameter definitions
- Docstring becomes tool description

#### **3. Agent.as_tool() - Agent as Tool**
```python
tool = sales_agent1.as_tool(
    tool_name="sales_agent1",
    tool_description="Write a cold sales email"
)

# Now this tool can be used by other agents
tools = [tool1, tool2, tool3, send_email]
```

**Benefits:**
- Compose agents into larger agents
- Agent can decide which sub-agent to use
- Enables hierarchical agent structures

#### **4. Handoffs - Agent Delegation**
```python
# Email formatter agent with tools
emailer_agent = Agent(
    name="Email Manager",
    instructions="Format and send emails",
    tools=[subject_tool, html_tool, send_html_email],
    model="gpt-4o-mini",
    handoff_description="Convert an email to HTML and send it"
)

# Sales manager can hand off to email manager
sales_manager = Agent(
    name="Sales Manager",
    instructions="...",
    tools=tools,
    handoffs=[emailer_agent],  # Pass control to email manager
    model="gpt-4o-mini"
)
```

**Tools vs Handoffs:**
| Feature | Tools | Handoffs |
|---------|-------|----------|
| Returns Control | Yes | No |
| Agent Responsible | Caller | Recipient |
| Use Case | Specific tasks | Full delegation |

#### **5. SendGrid Integration**
```python
import sendgrid
from sendgrid.helpers.mail import Mail, Email, To, Content

def send_email(body: str):
    sg = sendgrid.SendGridAPIClient(api_key=os.environ.get('SENDGRID_API_KEY'))
    from_email = Email("your@email.com")
    to_email = To("recipient@email.com")
    content = Content("text/plain", body)
    mail = Mail(from_email, to_email, "Subject", content).get()
    response = sg.client.mail.send.post(request_body=mail)
    return {"status": "success"}
```

**Setup:**
1. Create SendGrid account (free tier available)
2. Create API key in Settings → API Keys
3. Verify sender email in Settings → Sender Authentication
4. Add to `.env`: `SENDGRID_API_KEY=xxxx`

### Design Patterns

#### **Pattern 1: Multi-Agent Comparison**
```
User Request
    ↓
[Agent1, Agent2, Agent3] (parallel)
    ↓
Judge Agent (picks best)
    ↓
Result
```

#### **Pattern 2: Agent + Tool + Handoff**
```
Manager Agent (uses tools)
    ├→ Tool: Sub-agent (returns result)
    └→ Handoff: Specialized Agent (takes control)
```

---

## **LAB 3: Multi-Model, Structured Outputs & Guardrails**

### Overview
Advanced OpenAI features: multiple model support, structured JSON outputs, and input guardrails for safety.

### Quick Start Code
```python
from agents import Agent, Runner, trace, input_guardrail, GuardrailFunctionOutput
from pydantic import BaseModel
from openai import AsyncOpenAI

# Setup multi-model clients
gemini_client = AsyncOpenAI(
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/",
    api_key=os.getenv("GOOGLE_API_KEY")
)
deepseek_client = AsyncOpenAI(
    base_url="https://api.deepseek.com/v1",
    api_key=os.getenv("DEEPSEEK_API_KEY")
)

from agents.model_settings import OpenAIChatCompletionsModel

gemini_model = OpenAIChatCompletionsModel(model="gemini-2.0-flash", openai_client=gemini_client)
deepseek_model = OpenAIChatCompletionsModel(model="deepseek-chat", openai_client=deepseek_client)

# Create agents with different models
agent1 = Agent(name="Agent1", instructions="...", model=gemini_model)
agent2 = Agent(name="Agent2", instructions="...", model=deepseek_model)

# Structured outputs
class EmailStructure(BaseModel):
    subject: str
    body: str
    is_professional: bool

agent_with_structure = Agent(
    name="Email Writer",
    instructions="Write an email",
    output_type=EmailStructure,  # Enforce output structure
    model="gpt-4o-mini"
)

# Run with structured output
result = await Runner.run(agent_with_structure, "Write email")
print(result.final_output.subject)  # Access as object, not string!

# Input guardrails
class NameCheckOutput(BaseModel):
    is_name_in_message: bool
    name: str

guardrail_agent = Agent(
    name="Name Check",
    instructions="Check if message contains a personal name",
    output_type=NameCheckOutput,
    model="gpt-4o-mini"
)

@input_guardrail
async def check_for_names(ctx, agent, message):
    result = await Runner.run(guardrail_agent, message)
    return GuardrailFunctionOutput(
        output_info={"found_name": result.final_output},
        tripwire_triggered=result.final_output.is_name_in_message
    )

# Agent with guardrails
protected_agent = Agent(
    name="Safe Agent",
    instructions="...",
    model="gpt-4o-mini",
    input_guardrails=[check_for_names]  # Block if name detected
)
```

### Core Concepts

#### **1. OpenAI-Compatible Clients**
```python
from openai import AsyncOpenAI

# Google Gemini
gemini = AsyncOpenAI(
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/",
    api_key=os.getenv("GOOGLE_API_KEY")
)

# DeepSeek
deepseek = AsyncOpenAI(
    base_url="https://api.deepseek.com/v1",
    api_key=os.getenv("DEEPSEEK_API_KEY")
)

# Groq
groq = AsyncOpenAI(
    base_url="https://api.groq.com/openai/v1",
    api_key=os.getenv("GROQ_API_KEY")
)
```

**Why?**
- These providers support OpenAI API format
- Same code works with different models
- Mix and match providers in single workflow

#### **2. OpenAIChatCompletionsModel Wrapper**
```python
from agents.model_settings import OpenAIChatCompletionsModel

gemini_model = OpenAIChatCompletionsModel(
    model="gemini-2.0-flash",
    openai_client=gemini_client
)

agent = Agent(
    name="Multi-Model Agent",
    instructions="...",
    model=gemini_model  # Use custom model object
)
```

**Advantage:**
- Use any OpenAI-compatible provider
- Agents SDK handles the rest

#### **3. Structured Outputs (Pydantic)**
```python
from pydantic import BaseModel, Field

class EmailOutput(BaseModel):
    subject: str = Field(description="Email subject line")
    body: str = Field(description="Email body content")
    tone: str = Field(description="Tone: professional/casual/humorous")

agent = Agent(
    name="Email Writer",
    instructions="Write an email",
    output_type=EmailOutput,  # Force this structure
    model="gpt-4o-mini"
)

result = await Runner.run(agent, "Write email")

# Result is now a structured object, not a string!
print(result.final_output.subject)  # String
print(result.final_output.body)     # String
print(result.final_output.tone)     # String
```

**Benefits:**
- Type safety
- Validated outputs
- Programmatic access (not string parsing)
- API calls auto-check schema

#### **4. Input Guardrails**
```python
@input_guardrail
async def guardrail_against_names(ctx, agent, message):
    # Check if message violates policy
    result = await Runner.run(guardrail_agent, message)
    
    return GuardrailFunctionOutput(
        output_info={"found_name": result.final_output},
        tripwire_triggered=result.final_output.is_name_in_message  # True = block
    )

agent = Agent(
    name="Safe Agent",
    instructions="...",
    model="gpt-4o-mini",
    input_guardrails=[guardrail_against_names]
)

# If guardrail triggers, agent won't execute
result = await Runner.run(agent, "Send email to Alice")  # Blocked!
```

**GuardrailFunctionOutput:**
- `output_info`: Data about what was checked
- `tripwire_triggered`: True = block agent execution

### Multi-Model Best Practices

| Provider | Best For | Cost | Speed |
|----------|----------|------|-------|
| GPT-4o | Reasoning, complex tasks | $$$ | Medium |
| Gemini | Cost-effective, fast | $ | Fast |
| DeepSeek | Budget agents | $ | Medium |
| Groq | Real-time, fast | $ | FASTEST |

---

## **LAB 4: Deep Research Agent with Web Search**

### Overview
Build a comprehensive research agent that plans web searches, gathers information, writes reports, and emails results.

### Quick Start Code
```python
from agents import Agent, WebSearchTool, Runner, trace, function_tool
from pydantic import BaseModel, Field

# 1. Search planner agent
class WebSearchItem(BaseModel):
    reason: str = Field(description="Why this search is important")
    query: str = Field(description="Search term")

class WebSearchPlan(BaseModel):
    searches: list[WebSearchItem]

planner = Agent(
    name="Planner",
    instructions="Create 3 web search queries to answer the question",
    model="gpt-4o-mini",
    output_type=WebSearchPlan
)

# 2. Web search agent
search_agent = Agent(
    name="Search Agent",
    instructions="Summarize search results concisely",
    tools=[WebSearchTool(search_context_size="low")],
    model="gpt-4o-mini",
    model_settings=ModelSettings(tool_choice="required")  # Must use tool
)

# 3. Report writer agent
class ReportData(BaseModel):
    short_summary: str = Field(description="2-3 sentence summary")
    markdown_report: str = Field(description="Full report (5-10 pages)")
    follow_up_questions: list[str] = Field(description="Topics to research further")

writer = Agent(
    name="Writer",
    instructions="Write a detailed 1000+ word report",
    model="gpt-4o-mini",
    output_type=ReportData
)

# 4. Orchestrate the workflow
async def plan_searches(query: str):
    result = await Runner.run(planner, f"Query: {query}")
    return result.final_output

async def perform_searches(search_plan: WebSearchPlan):
    tasks = [asyncio.create_task(search(item)) for item in search_plan.searches]
    results = await asyncio.gather(*tasks)
    return results

async def search(item: WebSearchItem):
    input_text = f"Search term: {item.query}\nReason: {item.reason}"
    result = await Runner.run(search_agent, input_text)
    return result.final_output

async def write_report(query: str, search_results: list[str]):
    input_text = f"Query: {query}\nResults: {search_results}"
    result = await Runner.run(writer, input_text)
    return result.final_output

# Execute
query = "Latest AI Agent frameworks in 2025"
with trace("Research"):
    search_plan = await plan_searches(query)
    search_results = await perform_searches(search_plan)
    report = await write_report(query, search_results)
    print(report.markdown_report)
```

### Core Concepts

#### **1. WebSearchTool - Hosted Search**
```python
search_agent = Agent(
    name="Search Agent",
    instructions="Search the web and summarize findings",
    tools=[WebSearchTool(search_context_size="low")],  # "low" or other sizes
    model="gpt-4o-mini"
)

result = await Runner.run(search_agent, "Search for AI frameworks")
```

**Important:**
- WebSearchTool costs ~$0.025 per call
- Part of OpenAI Agents SDK
- Alternative: Use free/cheap search APIs with other frameworks

**search_context_size:**
- `"low"`: Fewer results, cheaper
- `"medium"`: Balanced
- `"high"`: More results, more expensive

#### **2. ModelSettings - Control Agent Behavior**
```python
from agents.model_settings import ModelSettings

search_agent = Agent(
    name="Search Agent",
    instructions="...",
    tools=[WebSearchTool()],
    model="gpt-4o-mini",
    model_settings=ModelSettings(
        tool_choice="required"  # Must always use the tool
    )
)
```

**tool_choice options:**
- `"auto"`: Use tool if needed
- `"required"`: Must use tool
- `"none"`: Never use tools

#### **3. Async Orchestration with asyncio.gather()**
```python
async def execute_parallel_searches(search_plan):
    # Create list of tasks
    tasks = [
        asyncio.create_task(search(item))
        for item in search_plan.searches
    ]
    
    # Wait for all to complete
    results = await asyncio.gather(*tasks)
    return results
```

**Why?**
- 3 searches in parallel = 1/3 the time
- Essential for performance in production

#### **4. Full Agent Workflow Pipeline**
```
User Query
    ↓
Planner Agent (output: WebSearchPlan)
    ↓
Parallel Search Agents (output: list[summaries])
    ↓
Writer Agent (output: ReportData)
    ↓
Email Agent (output: success)
```

**Pattern:**
- Plan → Execute → Synthesize → Action
- Each agent has clear responsibility
- Output of one = input to next

### OpenAI Hosted Tools Summary

| Tool | Cost | Use Case |
|------|------|----------|
| WebSearchTool | $0.025/call | Current web info |
| FileSearchTool | Included | Query vector stores |
| ComputerTool | $0.02/action | Automate computer tasks |

---

## **SYNTAX REFERENCE - 2_OPENAI**

### Imports
```python
from agents import Agent, Runner, trace, function_tool
from agents import WebSearchTool, input_guardrail, GuardrailFunctionOutput
from agents.model_settings import OpenAIChatCompletionsModel, ModelSettings
from pydantic import BaseModel, Field
from openai import AsyncOpenAI
```

### Agent Creation Patterns
```python
# Basic agent
agent = Agent(name="...", instructions="...", model="gpt-4o-mini")

# Agent with tools
agent = Agent(name="...", instructions="...", tools=[tool1, tool2], model="...")

# Agent with handoffs
agent = Agent(name="...", instructions="...", tools=[...], handoffs=[agent2], model="...")

# Agent with structured output
agent = Agent(name="...", instructions="...", output_type=MyModel, model="...")

# Agent with guardrails
agent = Agent(name="...", instructions="...", input_guardrails=[guard1], model="...")

# Agent with custom model
custom_model = OpenAIChatCompletionsModel(model="...", openai_client=client)
agent = Agent(name="...", instructions="...", model=custom_model)
```

### Execution Patterns
```python
# Simple execution
result = await Runner.run(agent, "prompt")
output = result.final_output

# Streaming
result = Runner.run_streamed(agent, "prompt")
async for event in result.stream_events():
    # Process event

# Parallel execution
results = await asyncio.gather(
    Runner.run(agent1, msg),
    Runner.run(agent2, msg),
    Runner.run(agent3, msg)
)

# With tracing
with trace("Operation name"):
    result = await Runner.run(agent, "prompt")
```

### Tool Definition Patterns
```python
# Function as tool
@function_tool
def my_function(param: str) -> str:
    """Description of what function does"""
    return "result"

# Agent as tool
tool = agent.as_tool(tool_name="name", tool_description="...")

# Hosted tools
tools = [WebSearchTool(search_context_size="low")]
```

### Multi-Model Setup
```python
from openai import AsyncOpenAI
from agents.model_settings import OpenAIChatCompletionsModel

# Gemini
gemini_client = AsyncOpenAI(
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/",
    api_key=os.getenv("GOOGLE_API_KEY")
)
gemini_model = OpenAIChatCompletionsModel(model="gemini-2.0-flash", openai_client=gemini_client)

# DeepSeek
deepseek_client = AsyncOpenAI(
    base_url="https://api.deepseek.com/v1",
    api_key=os.getenv("DEEPSEEK_API_KEY")
)
deepseek_model = OpenAIChatCompletionsModel(model="deepseek-chat", openai_client=deepseek_client)
```

### Guardrail Patterns
```python
@input_guardrail
async def my_guardrail(ctx, agent, message):
    result = await Runner.run(checker_agent, message)
    return GuardrailFunctionOutput(
        output_info={"check_result": result.final_output},
        tripwire_triggered=should_block  # True = block
    )

# Use guardrail
agent = Agent(name="...", input_guardrails=[my_guardrail], model="...")
```

## **LEARNING PROGRESSION**

```
Lab 1: Basics
  ↓
Lab 2: Workflows & Multi-Agent
  ↓
Lab 3: Advanced (Models, Structure, Guards)
  ↓
Lab 4: Production (Research Automation)
```

**Each lab builds on previous:**
- Lab 1 = Simple agent
- Lab 2 = Agent + Tools + Collaboration
- Lab 3 = Advanced features
- Lab 4 = Real-world complex system

---

## **PRODUCTION CONSIDERATIONS**

1. **Cost Management**
   - WebSearchTool: ~$0.025/call (expensive)
   - Use `search_context_size="low"` to reduce cost
   - Monitor token usage

2. **Error Handling**
   - SSL/Certificate issues: `pip install --upgrade certifi`
   - SendGrid failures: Check API key and verified sender
   - Tool failures: Agent retries automatically

3. **Debugging**
   - Always use `trace()` context manager
   - View traces at https://platform.openai.com/traces
   - Check agent.final_output for errors

4. **Scalability**
   - Use `asyncio.gather()` for parallel execution
   - Rate limiting: Space out API calls
   - Consider using cheaper models (gpt-4o-mini)

---

## **KEY TAKEAWAYS**

✓ OpenAI Agents SDK handles tool boilerplate
✓ @function_tool eliminates JSON schema writing
✓ Agent.as_tool() enables agent composition
✓ Handoffs for full delegation, tools for returning control
✓ Multiple models via OpenAI-compatible clients
✓ Structured outputs via Pydantic (type-safe)
✓ Guardrails for safety and policy enforcement
✓ WebSearchTool for current information
✓ trace() essential for production observability
✓ Async/await for performance (parallel execution)
