# **6_MCP - TECHNICAL DOCUMENTATION**

## **OVERVIEW**

### What is MCP?

MCP is a **standardized protocol** for connecting agents/clients with tools, resources, and prompt templates.

**Key Insight**: MCP is not an agent framework (like CrewAI/LangGraph). Instead, it's a **protocol for sharing capabilities** between systems.

**Architecture:**
```
Client (Agent)
    ↓ (JSON-RPC over stdio)
MCP Server
    ↓
Tools / Resources / Prompts
```

---

## **CORE CONCEPTS**

### 1. **Tools**

Functions exposed by MCP servers that agents can call.

```python
# Example: Fetch tool
def fetch(url: str) -> str:
    """Fetch content from a URL"""
    # Implementation
    return content
```

### 2. **Resources**

Read-only data exposed by MCP servers (like file system access).

```python
# Example: Account resource
async def get_account(name: str) -> str:
    """Get account information for a user"""
    return account_info
```

### 3. **Prompts**

Pre-written system prompts and templates that agents can use.

---

## **THREE TYPES OF MCP SERVERS**

### Type 1: Local, Everything Local
**Example**: Memory server (knowledge graph stored locally)
```python
params = {
    "command": "npx",
    "args": ["-y", "mcp-memory-libsql"],
    "env": {"LIBSQL_URL": "file:./memory/ed.db"}
}
```

### Type 2: Local, Calls Web Service
**Example**: Brave Search (local server, calls web API)
```python
params = {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-brave-search"],
    "env": {"BRAVE_API_KEY": "..."}
}
```

### Type 3: Remote/Hosted
**Example**: Cloudflare deployed MCP servers
```
Less common, not well standardized yet
```

---

## **LAB 1: USING PUBLIC MCP SERVERS**

### Learning Objectives
- Discover MCP servers
- Integrate with OpenAI Agents SDK
- Use multiple MCP servers together

### Quick Start: OpenAI Agents SDK + MCP

```python
from dotenv import load_dotenv
from agents import Agent, Runner, trace
from agents.mcp import MCPServerStdio

load_dotenv(override=True)

# Create MCP server (spawns process)
fetch_params = {"command": "uvx", "args": ["mcp-server-fetch"]}

async with MCPServerStdio(params=fetch_params, client_session_timeout_seconds=60) as server:
    # List tools provided by server
    fetch_tools = await server.list_tools()
    print(fetch_tools)
    
    # Create agent with MCP server
    instructions = "You browse the internet to accomplish your instructions."
    agent = Agent(
        name="investigator",
        instructions=instructions,
        model="gpt-4o-mini",
        mcp_servers=[server]  # Pass server to agent
    )
    
    # Agent now has access to all tools from server
    with trace("investigate"):
        result = await Runner.run(agent, "Find a recipe for Banoffee Pie")
        print(result.final_output)
```

### Multiple MCP Servers

```python
# Combine multiple servers
async with MCPServerStdio(params=fetch_params, ...) as fetch_server:
    async with MCPServerStdio(params=playwright_params, ...) as browser_server:
        async with MCPServerStdio(params=files_params, ...) as files_server:
            # All tools from all servers available to agent
            agent = Agent(
                name="agent",
                instructions="Browse web, extract text, save files",
                model="gpt-4o-mini",
                mcp_servers=[fetch_server, browser_server, files_server]
            )
```

### Available Public MCP Servers

| Server | Type | Use Case |
|--------|------|----------|
| **fetch** | Local | Fetch web content |
| **Playwright** | Local | Browser automation |
| **Filesystem** | Local | File operations |
| **Memory** | Local | Persistent knowledge graph |
| **Brave Search** | Local + Web API | Web search |
| **Polygon.io** | Local + Web API | Financial data |

### MCP Marketplaces

- https://mcp.so
- https://glama.ai/mcp
- https://smithery.ai/

---

## **LAB 2: CREATING YOUR OWN MCP SERVER**

### Learning Objectives
- Create MCP server from Python code
- Create MCP client
- Integrate custom server with agents

### Step 1: Create Server Module

```python
# accounts.py - Business logic
class Account:
    def __init__(self, name: str, balance: float = 10000):
        self.name = name
        self.balance = balance
        self.holdings = {}
    
    def buy_shares(self, symbol: str, quantity: int, reason: str):
        """Buy shares in a stock"""
        # Implementation
        pass
    
    def get_balance(self) -> float:
        """Get account balance"""
        return self.balance
    
    def report(self) -> str:
        """Get full account report"""
        # Return formatted report
        return report
```

### Step 2: Wrap as MCP Server

```python
# accounts_server.py - MCP Server
from mcp.server import Server
from mcp.types import Tool, TextContent, ToolUseBlock
from accounts import Account

server = Server("accounts_server")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="get_balance",
            description="Get account balance",
            inputSchema={"type": "object", "properties": {...}}
        ),
        Tool(
            name="buy_shares",
            description="Buy shares in a stock",
            inputSchema={"type": "object", "properties": {...}}
        ),
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    account = Account.get(arguments["account_name"])
    
    if name == "get_balance":
        balance = account.get_balance()
        return [TextContent(type="text", text=str(balance))]
    
    elif name == "buy_shares":
        account.buy_shares(
            arguments["symbol"],
            arguments["quantity"],
            arguments["reason"]
        )
        return [TextContent(type="text", text="Shares purchased")]

if __name__ == "__main__":
    import asyncio
    asyncio.run(server.run(sys.stdin.buffer, sys.stdout.buffer))
```

### Step 3: Run Server in Agent

```python
from agents import Agent, Runner
from agents.mcp import MCPServerStdio

params = {"command": "uv", "args": ["run", "accounts_server.py"]}

async with MCPServerStdio(params=params, client_session_timeout_seconds=30) as mcp_server:
    agent = Agent(
        name="account_manager",
        instructions="You manage accounts. Answer questions about accounts.",
        model="gpt-4o-mini",
        mcp_servers=[mcp_server]
    )
    
    result = await Runner.run(agent, "What's my balance?")
    print(result.final_output)
```

### Step 4: Create MCP Client (Optional)

```python
# accounts_client.py - Direct client without going through agent
from mcp.client.session import ClientSession
from mcp.client.stdio import StdioClientTransport

async def get_accounts_tools():
    """Get tools from accounts server"""
    transport = StdioClientTransport(
        command="uv",
        args=["run", "accounts_server.py"]
    )
    
    async with ClientSession(transport) as session:
        tools = await session.list_tools()
        return tools

async def read_accounts_resource(account_name: str) -> str:
    """Read account details as resource"""
    # Implementation
    pass

# Usage
tools = await get_accounts_tools()
for tool in tools:
    print(tool.name)
```

---

## **LAB 3: MCP SERVER EXPLORATION**

### Learning Objectives
- Use memory MCP server (knowledge graph)
- Use web search MCP server
- Use financial data MCP server (Polygon.io)

### Memory Server (Persistent Knowledge Graph)

```python
params = {
    "command": "npx",
    "args": ["-y", "mcp-memory-libsql"],
    "env": {"LIBSQL_URL": "file:./memory/ed.db"}
}

async with MCPServerStdio(params=params, ...) as mcp_server:
    agent = Agent(
        name="agent",
        instructions="Use entity tools to store and recall information",
        model="gpt-4o-mini",
        mcp_servers=[mcp_server]
    )
    
    # First call - agent stores info about "Ed"
    result1 = await Runner.run(
        agent,
        "My name's Ed. I'm an LLM engineer teaching a course on AI Agents."
    )
    
    # Second call - agent retrieves stored info
    result2 = await Runner.run(
        agent,
        "What do you know about me?"  # Retrieves from knowledge graph!
    )
```

**Tools Available:**
- `create_entities` - Add entities to graph
- `add_observations` - Add facts about entities
- `add_relations` - Add relationships between entities
- `search_entities` - Query graph

### Brave Search Server

```python
env = {"BRAVE_API_KEY": os.getenv("BRAVE_API_KEY")}
params = {
    "command": "npx",
    "args": ["-y", "@modelcontextprotocol/server-brave-search"],
    "env": env
}

async with MCPServerStdio(params=params, ...) as mcp_server:
    agent = Agent(
        name="agent",
        instructions="Search web for latest news and summarize",
        model="gpt-4o-mini",
        mcp_servers=[mcp_server]
    )
    
    result = await Runner.run(agent, "What's latest news on Amazon stock?")
```

### Polygon.io Financial Data Server

#### Free Plan Setup
```python
# 1. Sign up at polygon.io
# 2. Get API key from dashboard
# 3. Add to .env: POLYGON_API_KEY=xxx

from polygon import RESTClient
client = RESTClient(os.getenv("POLYGON_API_KEY"))
data = client.get_previous_close_agg("AAPL")[0]  # End of day price
```

#### As MCP Server (Free)
```python
from market import get_share_price  # Wrapper with caching

price = get_share_price("AAPL")  # Cached to avoid rate limiting

# Expose as MCP server
params = {"command": "uv", "args": ["run", "market_server.py"]}

async with MCPServerStdio(params=params, ...) as mcp_server:
    agent = Agent(
        name="agent",
        instructions="Answer questions about stock market",
        model="gpt-4o-mini",
        mcp_servers=[mcp_server]
    )
    
    result = await Runner.run(agent, "What's the share price of Apple?")
```

#### Paid Plan (Optional)
```
# If you subscribe to paid plan:
# 1. Set POLYGON_API_KEY in .env
# 2. Set POLYGON_PLAN=paid or POLYGON_PLAN=realtime
# 3. Use full Polygon.io MCP server

params = {
    "command": "uvx",
    "args": ["--from", "git+https://github.com/polygon-io/mcp_polygon@v0.1.0", "mcp_polygon"],
    "env": {"POLYGON_API_KEY": polygon_api_key}
}

# Provides 40+ additional tools for market data
```

---

## **LAB 4: CAPSTONE - AUTONOMOUS TRADERS (SINGLE TRADER)**

### Learning Objectives
- Combine multiple MCP servers
- Create trading agent with tools, resources, and instructions
- Build Python module for agent

### Architecture

```
Trader Agent
    ├─ Tools (from MCP servers)
    │  ├ Search web
    │  ├ Check stock prices
    │  ├ Buy/sell shares
    │  └ Save to memory
    │
    ├─ Resources (read-only)
    │  ├ Account details
    │  └ Trading strategy
    │
    └─ Tool: Researcher Agent
       ├ Search for news
       └ Find opportunities
```

### MCP Servers for Trader

```python
# Accounts (buy/sell shares)
{"command": "uv", "args": ["run", "accounts_server.py"]}

# Market data (check prices)
{"command": "uv", "args": ["run", "market_server.py"]}

# Memory (remember research)
{"command": "npx", "args": ["-y", "mcp-memory-libsql"], "env": {...}}

# Web search
{"command": "npx", "args": ["-y", "@modelcontextprotocol/server-brave-search"], "env": {...}}

# Notifications
{"command": "uv", "args": ["run", "push_server.py"]}
```

### Create Trader Agent

```python
async def create_trader(agent_name: str, mcp_servers: list):
    # Read account info as resource
    account_details = await read_accounts_resource(agent_name)
    strategy = await read_strategy_resource(agent_name)
    
    instructions = f"""
You are a trader managing a portfolio. Your name is {agent_name}.
Your investment strategy: {strategy}
Your current holdings: {account_details}

You have tools to:
- Search web for news
- Check stock prices  
- Buy and sell shares
- Save to memory

Make trades based on your strategy.
    """
    
    # Create researcher as tool
    researcher_tool = await get_researcher_tool(researcher_servers)
    
    trader = Agent(
        name=agent_name,
        instructions=instructions,
        tools=[researcher_tool],  # Call researcher
        mcp_servers=trader_mcp_servers,
        model="gpt-4o-mini"
    )
    
    return trader

# Run trader
trader = await create_trader("Ed", mcp_servers)
result = await Runner.run(trader, "Manage your portfolio")
```

### Convert to Python Module

```python
# traders.py
class Trader:
    def __init__(self, name: str):
        self.name = name
        self.mcp_servers = []
    
    async def connect_servers(self):
        """Connect to all MCP servers"""
        # Setup all servers
        pass
    
    async def run(self):
        """Execute trader agent"""
        agent = await self.create_agent()
        result = await Runner.run(agent, self.trading_prompt())
        return result
    
    async def disconnect_servers(self):
        """Cleanup"""
        pass

# Usage
trader = Trader("Ed")
await trader.connect_servers()
result = await trader.run()
```

---

## **LAB 5: CAPSTONE - AUTONOMOUS TRADERS (TEAM)**

### Four Traders

```python
traders = [
    Trader("Warren"),  # Value investor (Buffett-inspired)
    Trader("George"),  # Macro trader (Soros-inspired)
    Trader("Ray"),     # Systematic trader (Dalio-inspired)
    Trader("Cathie"),  # Growth investor (Wood-inspired)
]

# Each with different investment strategy
```

### Trading Floor (Event Loop)

```python
# trading_floor.py
import asyncio

RUN_EVERY_N_MINUTES = 60  # From .env
RUN_EVEN_WHEN_MARKET_IS_CLOSED = False  # From .env

async def trading_loop():
    traders = [
        Trader("Warren"),
        Trader("George"),
        Trader("Ray"),
        Trader("Cathie"),
    ]
    
    while True:
        # Run all traders in parallel
        await asyncio.gather(*[trader.run() for trader in traders])
        
        # Wait before next round
        await asyncio.sleep(RUN_EVERY_N_MINUTES * 60)

# Start: uv run trading_floor.py
```

### Web UI (Gradio/Streamlit)

```python
# app.py
import gradio as gr

def get_trader_status(trader_name):
    """Get current holdings and performance"""
    account = Account.get(trader_name)
    return account.report()

with gr.Blocks() as demo:
    gr.Markdown("# Autonomous Traders")
    
    with gr.Row():
        warren_display = gr.Markdown()
        george_display = gr.Markdown()
        ray_display = gr.Markdown()
        cathie_display = gr.Markdown()
    
    # Update displays periodically
    demo.load(
        update_displays,
        outputs=[warren_display, george_display, ray_display, cathie_display],
        every=10
    )

demo.launch()
```

### Custom Tracer (Optional)

```python
# tracers.py - Record agent thinking
class DatabaseTracer(Tracer):
    async def log_message(self, level, message):
        """Log agent's internal thoughts to database"""
        # Save to database for later inspection
        pass

# Use tracer to monitor agent thinking and display in UI
```

---

## **SYNTAX REFERENCE**

### Using MCP with OpenAI Agents SDK

```python
from agents import Agent, Runner, trace
from agents.mcp import MCPServerStdio

# Define MCP server parameters
params = {
    "command": "uvx",  # or "npx", "uv", etc.
    "args": ["mcp-server-fetch"],
    "env": {"API_KEY": "..."}  # Optional
}

# Connect to server
async with MCPServerStdio(params=params, client_session_timeout_seconds=60) as server:
    # List available tools
    tools = await server.list_tools()
    
    # Create agent with server
    agent = Agent(
        name="name",
        instructions="...",
        model="gpt-4o-mini",
        mcp_servers=[server]  # Pass server(s)
    )
    
    # Run agent (has access to all server tools)
    result = await Runner.run(agent, "prompt")
    print(result.final_output)

# Automatically closes server on exit
```

### Multiple Servers

```python
async with MCPServerStdio(params=params1) as server1:
    async with MCPServerStdio(params=params2) as server2:
        agent = Agent(
            name="agent",
            instructions="...",
            model="gpt-4o-mini",
            mcp_servers=[server1, server2]  # All tools available
        )
        result = await Runner.run(agent, "prompt")
```

### Creating Custom MCP Server

```python
# server.py
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("my_server")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="my_tool",
            description="What it does",
            inputSchema={
                "type": "object",
                "properties": {
                    "param1": {"type": "string"}
                },
                "required": ["param1"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "my_tool":
        result = my_function(arguments["param1"])
        return [TextContent(type="text", text=result)]

if __name__ == "__main__":
    import sys
    import asyncio
    asyncio.run(server.run(sys.stdin.buffer, sys.stdout.buffer))
```

### Context Manager for Multiple Servers

```python
from contextlib import AsyncExitStack
from agents.mcp import MCPServerStdio

async with AsyncExitStack() as stack:
    servers = [
        await stack.enter_async_context(MCPServerStdio(params))
        for params in [params1, params2, params3]
    ]
    
    # All servers connected, agents have access to all tools
    agent = Agent(name="agent", mcp_servers=servers, ...)
    result = await Runner.run(agent, "prompt")
    
# All servers automatically cleaned up on exit
```


## **KEY CONCEPTS**

### 1. **Server Spawning**
MCP servers are **spawned as separate processes** via stdio communication.

```
Client ←→ (JSON-RPC via stdio) ←→ Server Process
```

### 2. **Tool Discovery**
Agents call `list_tools()` to discover what's available.

```python
tools = await server.list_tools()
# Returns list of Tool objects with schema
```

### 3. **Resource Access**
Resources are read-only data (not functions).

```python
# Resource example: account information
account_info = await server.read_resource("ed")
```

### 4. **Prompt Templates**
Servers can provide pre-written prompts.

```python
# Get system prompt from server
system_prompt = await server.get_prompt("trading_strategy")
```

---

## **PRODUCTION CONSIDERATIONS**

1. **API Rate Limiting**
   - Polygon.io: Free tier is heavily rate limited
   - Solution: Cache results locally

2. **Cost Management**
   - Web searches cost tokens
   - Financial data might require paid plan
   - Monitor API usage

3. **Error Handling**
   ```python
   try:
       async with MCPServerStdio(params, timeout=60) as server:
           # Use server
   except Exception as e:
       logger.error(f"Server failed: {e}")
       # Fallback logic
   ```

4. **Cleanup**
   - Always use `async with` for automatic cleanup
   - Don't leave server processes hanging

---

## **KEY DIFFERENCES: MCP VS AGENTS**

| Aspect | MCP | Agent Framework |
|--------|-----|-----------------|
| **Scope** | Tool sharing protocol | Full agent system |
| **Purpose** | Standardize tool access | Build agents |
| **Integration** | Via servers | Monolithic |
| **Reusability** | High (protocol-based) | Lower (framework-specific) |
| **Learning Curve** | Steep (protocol details) | Medium |

---

## **KEY TAKEAWAYS**

✓ **MCP** = Protocol for sharing tools/resources
✓ **Not** an agent framework
✓ **Standardized** JSON-RPC communication
✓ **Three types**: Local, Local+Web API, Remote
✓ **Integration** via MCPServerStdio
✓ **Tool discovery** via list_tools()
✓ **Create servers** from any Python code
✓ **Multiple servers** combine tools
✓ **Memory servers** provide persistent knowledge graphs
✓ **Financial data** via Polygon.io
✓ **Web search** via Brave Search
✓ **Autonomy** agents manage their own decisions
✓ **Scalability** parallel traders via asyncio.gather()
✓ **Monitoring** via custom tracers
✓ **Production** requires UI, database, monitoring
