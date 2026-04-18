# Foundations Labs Reference Guide

---

## LAB 1: Getting Started with LLM APIs

### Quick Start
```python
from dotenv import load_dotenv
from openai import OpenAI

load_dotenv(override=True)
openai = OpenAI()

messages = [{"role": "user", "content": "What is 2+2?"}]
response = openai.chat.completions.create(model="gpt-4.1-mini", messages=messages)
print(response.choices[0].message.content)
```

### Key Concepts
- **Environment Variables**: `load_dotenv(override=True)` loads from `.env`
- **Message Format**: `[{"role": "user/system/assistant", "content": "text"}]`
- **API Call**: `openai.chat.completions.create(model="...", messages=[...])`
- **Response**: `response.choices[0].message.content`
- **Chaining**: Use response from first call as input for second call

### Parameters
| Parameter | Purpose |
|-----------|---------|
| `model` | Which LLM to use (nano/mini/5-mini) |
| `messages` | Conversation history |
| `temperature` | 0=deterministic, 2=creative |
| `max_tokens` | Response length limit |

### Common Errors
| Error | Fix |
|-------|-----|
| `AuthenticationError` | Check API key in `.env` |
| `NameError: 'openai'` | Initialize with `openai = OpenAI()` |
| `FileNotFoundError: .env` | Create `.env` file in project root |

---

## LAB 2: Multi-Model Comparison & Evaluation

### Quick Start
```python
competitors = []
answers = []

# OpenAI
response = openai.chat.completions.create(model="gpt-5-nano", messages=messages)
competitors.append("gpt-5-nano")
answers.append(response.choices[0].message.content)

# Claude
response = claude.messages.create(model="claude-sonnet-4-5", messages=messages, max_tokens=1000)
competitors.append("claude-sonnet-4-5")
answers.append(response.content[0].text)

# Judge responses
results = json.loads(judge_response.choices[0].message.content)
```

### Provider Setups
```python
# OpenAI
openai = OpenAI()

# Anthropic (note: max_tokens REQUIRED)
claude = Anthropic()
response = claude.messages.create(model="...", messages=messages, max_tokens=1000)

# Google Gemini
gemini = OpenAI(api_key=os.getenv("GOOGLE_API_KEY"), 
    base_url="https://generativelanguage.googleapis.com/v1beta/openai/")

# DeepSeek
deepseek = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"), 
    base_url="https://api.deepseek.com/v1")

# Groq
groq = OpenAI(api_key=os.getenv("GROQ_API_KEY"), 
    base_url="https://api.groq.com/openai/v1")

# Ollama (local)
ollama = OpenAI(base_url='http://localhost:11434/v1', api_key='ollama')
```

### Judge Pattern
```python
# Combine all responses
together = "\n".join([f"Response {i+1}:\n{ans}" for i, ans in enumerate(answers)])

# Judge them
judge_prompt = f"Rank these: {together}\nRespond with JSON: {{\"results\": [1, 2, 3]}}"
response = openai.chat.completions.create(model="gpt-5-mini", messages=[{"role": "user", "content": judge_prompt}])

# Parse results
results_dict = json.loads(response.choices[0].message.content)
ranks = results_dict["results"]
```

### Provider Comparison
| Provider | Cost | Speed | Best For |
|----------|------|-------|----------|
| OpenAI | Medium | Medium | Reliability |
| Claude | High | Slow | Complex reasoning |
| Gemini | Cheap | Fast | Cost-effective |
| DeepSeek | Very Low | Fast | Learning |
| Groq | Cheap | FASTEST | Real-time |
| Ollama | FREE | Slow | Private/offline |

---

## LAB 3: Chat Interface with Quality Control & Retry

### Quick Start
```python
from pypdf import PdfReader
from pydantic import BaseModel
import gradio as gr

# Extract PDF
reader = PdfReader("me/linkedin.pdf")
linkedin = ""
for page in reader.pages:
    text = page.extract_text()
    if text:
        linkedin += text

# System prompt
system_prompt = f"You are Ed. Background: {linkedin}. Be professional."

# Pydantic model
class Evaluation(BaseModel):
    is_acceptable: bool
    feedback: str

# Evaluate with LLM
evaluation = gemini.beta.chat.completions.parse(
    model="gemini-2.5-flash",
    messages=[{"role": "user", "content": "Evaluate: ..."}],
    response_format=Evaluation
)

# Chat function
def chat(message, history):
    messages = [{"role": "system", "content": system_prompt}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model="gpt-4o-mini", messages=messages)
    return response.choices[0].message.content

gr.ChatInterface(chat, type="messages").launch()
```

### Key Concepts
- **PDF Extraction**: `PdfReader()` reads text from PDFs
- **System Prompt**: Embed context directly in prompt with f-strings
- **Gradio Chat**: `gr.ChatInterface(function, type="messages").launch()`
- **Pydantic Models**: Enforce structure on LLM responses
- **`.beta.chat.completions.parse()`**: Get structured outputs (not `.create()`)
- **Evaluation Loop**: Evaluate response, retry if poor

### Evaluation & Retry
```python
def evaluate(reply, message, history) -> Evaluation:
    prompt = f"Is this good? {reply}"
    response = gemini.beta.chat.completions.parse(model="gemini-2.5-flash", 
        messages=[{"role": "user", "content": prompt}], response_format=Evaluation)
    return response.choices[0].message.parsed

def rerun(reply, message, history, feedback):
    updated_system = system_prompt + f"\nFeedback: {feedback}\nTry again."
    messages = [{"role": "system", "content": updated_system}] + history + [{"role": "user", "content": message}]
    response = openai.chat.completions.create(model="gpt-4o-mini", messages=messages)
    return response.choices[0].message.content

# In chat function:
evaluation = evaluate(reply, message, history)
if evaluation.is_acceptable:
    return reply
else:
    return rerun(reply, message, history, evaluation.feedback)
```

---

## LAB 4: Tools, Agents & Deployment

### Quick Start
```python
import json

# Tool definition
tool_def = {
    "name": "record_user",
    "description": "Record user contact",
    "parameters": {
        "type": "object",
        "properties": {"email": {"type": "string"}, "name": {"type": "string"}},
        "required": ["email"],
        "additionalProperties": False
    }
}

# Tool function
def record_user(email, name="Unknown"):
    return {"recorded": "ok"}

# Handler
def handle_tool_calls(tool_calls):
    results = []
    for call in tool_calls:
        name = call.function.name
        args = json.loads(call.function.arguments)
        result = globals()[name](**args)
        results.append({"role": "tool", "content": json.dumps(result), "tool_call_id": call.id})
    return results

# Agent loop
def chat(message, history):
    messages = [{"role": "system", "content": system}] + history + [{"role": "user", "content": message}]
    
    done = False
    while not done:
        response = openai.chat.completions.create(model="gpt-4o-mini", messages=messages, tools=[{"type": "function", "function": tool_def}])
        
        if response.choices[0].finish_reason == "tool_calls":
            msg = response.choices[0].message
            results = handle_tool_calls(msg.tool_calls)
            messages.append(msg)
            messages.extend(results)
        else:
            done = True
    
    return response.choices[0].message.content
```

### Tool Definition Schema
```python
tool = {
    "name": "function_name",              # Must match Python function
    "description": "What it does",        # LLM reads this
    "parameters": {
        "type": "object",
        "properties": {
            "param1": {"type": "string", "description": "..."},
            "param2": {"type": "integer"}
        },
        "required": ["param1"],            # Mandatory parameters
        "additionalProperties": False      # Only defined params allowed
    }
}
```

### Agent Loop Pattern
```python
while not done:
    response = openai.chat.completions.create(..., tools=tools)
    
    if response.choices[0].finish_reason == "tool_calls":
        execute_tools()
        add_results_to_messages()
    else:
        done = True
```

**finish_reason values:**
- `"tool_calls"`: LLM wants to use a tool
- `"stop"`: LLM finished responding

### Pushover Notifications
```python
import requests

def push(message):
    requests.post("https://api.pushover.net/1/messages.json",
        data={"user": os.getenv("PUSHOVER_USER"), "token": os.getenv("PUSHOVER_TOKEN"), "message": message})

# In tools:
def record_user(email, name="Unknown"):
    push(f"New signup: {name}")
    return {"recorded": "ok"}
```

### Deployment to HuggingFace
```bash
# Install CLI
uv tool install 'huggingface_hub[cli]'
hf auth login --token hf_xxxxx

# Deploy
cd 1_foundations
uv run gradio deploy

# Answer prompts: career_conversation, app.py, cpu-basic
# Add secrets: OPENAI_API_KEY, PUSHOVER_USER, PUSHOVER_TOKEN
```

### Class-Based Architecture
```python
class Agent:
    def __init__(self):
        self.openai = OpenAI()
        # Load context...
    
    def system_prompt(self):
        return f"You are..."
    
    def handle_tool_calls(self, tool_calls):
        # Execute tools...
    
    def chat(self, message, history):
        # Agent loop...

agent = Agent()
gr.ChatInterface(agent.chat, type="messages").launch()
```

---

## LAB 5: Agent Loops from First Principles

### Quick Start
```python
from rich.console import Console
import json

# State
todos = []
completed = []

# Tools
def create_todos(descriptions: list[str]) -> str:
    todos.extend(descriptions)
    completed.extend([False] * len(descriptions))
    return get_report()

def mark_complete(index: int, notes: str) -> str:
    completed[index - 1] = True
    return get_report()

# Handler
def handle_tool_calls(tool_calls):
    results = []
    for call in tool_calls:
        name = call.function.name
        args = json.loads(call.function.arguments)
        result = globals()[name](**args)
        results.append({"role": "tool", "content": json.dumps(result), "tool_call_id": call.id})
    return results

# Agent loop
def agent_loop(messages):
    done = False
    while not done:
        response = openai.chat.completions.create(model="gpt-5.2", messages=messages, tools=tools)
        
        if response.choices[0].finish_reason == "tool_calls":
            msg = response.choices[0].message
            results = handle_tool_calls(msg.tool_calls)
            messages.append(msg)
            messages.extend(results)
        else:
            done = True
    
    Console().print(response.choices[0].message.content)
```

### Tool Definitions (Array Parameters)
```python
tools = [
    {"type": "function", "function": {
        "name": "create_todos",
        "description": "Create new todos",
        "parameters": {
            "type": "object",
            "properties": {
                "descriptions": {
                    "type": "array",
                    "items": {"type": "string"},
                    "description": "List of todos"
                }
            },
            "required": ["descriptions"]
        }
    }}
]
```

### Rich Console Output
```python
from rich.console import Console

console = Console()
console.print("[bold]Bold[/bold]")
console.print("[green]Success[/green]")
console.print("[red]Error[/red]")
console.print("[strike]Completed[/strike]")
console.print("[bold green]✓ Done[/bold green]")
```

### Why This Works (The Pattern)
```
LLM decides which tool to use
     ↓
Tool executes function
     ↓
Result fed back to LLM
     ↓
LLM reads result, decides next action
     ↓
Loop continues until LLM responds
```

**"Unreasonable Effectiveness"**: Simple loop + tools creates powerful autonomous behavior WITHOUT frameworks.

---

## SYNTAX REFERENCE - ALL LABS

### Initialization
```python
from dotenv import load_dotenv
from openai import OpenAI
import json, os

load_dotenv(override=True)
openai = OpenAI()
```

### API Calls
```python
# Basic
response = openai.chat.completions.create(model="gpt-4.1-mini", messages=[...])

# With tools
response = openai.chat.completions.create(model="...", messages=[...], tools=[...])

# Structured output
response = gemini.beta.chat.completions.parse(model="...", messages=[...], response_format=Model)

# Get response
text = response.choices[0].message.content
obj = response.choices[0].message.parsed
```

### Messages
```python
messages = [{"role": "system", "content": "..."}, {"role": "user", "content": "..."}]

# Build dynamically
messages.append({"role": "assistant", "content": "..."})
messages.extend([{"role": "user", "content": "..."}])
```

### PDF & JSON
```python
# PDF
from pypdf import PdfReader
reader = PdfReader("file.pdf")
text = ""
for page in reader.pages:
    text += page.extract_text() or ""

# JSON
result_dict = json.loads(json_string)
json_string = json.dumps(dict)
args = json.loads(tool_call.function.arguments)
```

### Pydantic
```python
from pydantic import BaseModel

class Model(BaseModel):
    field: str
    count: int

obj = response.choices[0].message.parsed
```

### Gradio
```python
import gradio as gr

def chat(message, history):
    return response_text

gr.ChatInterface(chat, type="messages").launch()
```

### Tools
```python
# Definition
tool = {"name": "func", "description": "...", "parameters": {"type": "object", "properties": {...}, "required": [...]}}

# Function
def func(param):
    return {"result": "..."}

# Handler
def handle_tool_calls(tool_calls):
    for call in tool_calls:
        name = call.function.name
        args = json.loads(call.function.arguments)
        result = globals()[name](**args)
```

### Providers
```python
openai = OpenAI()
claude = Anthropic()
gemini = OpenAI(api_key=os.getenv("GOOGLE_API_KEY"), base_url="https://generativelanguage.googleapis.com/v1beta/openai/")
deepseek = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"), base_url="https://api.deepseek.com/v1")
groq = OpenAI(api_key=os.getenv("GROQ_API_KEY"), base_url="https://api.groq.com/openai/v1")
ollama = OpenAI(base_url='http://localhost:11434/v1', api_key='ollama')
```

---

## THE BIG PICTURE

**Lab 1** → Basic API calls
↓
**Lab 2** → Multiple models + evaluation
↓
**Lab 3** → Chat UI + quality control
↓
**Lab 4** → Tools + agent loop (production)
↓
**Lab 5** → Agent loop from scratch (understanding)

**Core Pattern Every Lab Uses:**
```
while not done:
    response = LLM(messages, [optional: tools])
    if response.finish_reason == "tool_calls":
        execute_tools()
        add_to_messages()
    else:
        done = True
```

**This simple loop is the foundation of ALL agentic AI.**

---

## COMMON ERRORS & FIXES

| Error | Lab | Fix |
|-------|-----|-----|
| `AuthenticationError` | 1 | Check API key in `.env` |
| `json.JSONDecodeError` | 2 | LLM didn't return valid JSON |
| `ValueError: response_format` | 3 | Use `.beta.chat.completions.parse()` |
| `Tool not found` | 4 | Function name must match tool "name" |
| `FileNotFoundError: .env` | All | Create `.env` in project root |

---

## KEY TAKEAWAYS BY LAB

**Lab 1**: Messages are conversation history. Chain calls for multi-step workflows.

**Lab 2**: Different providers, same pattern. Use LLM to judge LLM outputs.

**Lab 3**: Embed context in prompts. Evaluate and retry improves quality.

**Lab 4**: Agent loop executes tools until LLM responds. This is agentic AI.

**Lab 5**: Simple loop + tools = autonomous behavior. No frameworks needed.
