# CrewAI Framework - Technical Reference

---

## **OVERVIEW**

### What is CrewAI?

CrewAI is a framework for building **multi-agent systems** where:
- Each agent has a specific **role** and **goal**
- Agents execute **tasks** in defined sequences
- Agents can **delegate**, **communicate**, and **collaborate**
- All controlled via **YAML configuration** (agents.yaml, tasks.yaml)


---



## **PROJECT 1: CODER CREW**

### Purpose
Autonomous code generation and execution. Write code, execute it safely in Docker.

### Key Features
```python
@agent
def coder(self) -> Agent:
    return Agent(
        config=self.agents_config['coder'],
        verbose=True,
        allow_code_execution=True,      # Can write & run code
        code_execution_mode="safe",     # Uses Docker for isolation
        max_execution_time=30,          # Timeout protection
        max_retry_limit=3               # Retry on failure
    )
```

**Capabilities:**
- Write Python code
- Execute code safely in containerized environment
- Self-correct on errors
- Timeout & retry protection

### Example Task
```python
assignment = '''
Write a python program to calculate the first 10,000 terms 
of this series, multiplying the total by 4: 1 - 1/3 + 1/5 - 1/7 + ...
'''

result = Coder().crew().kickoff(inputs={'assignment': assignment})
```

### YAML Configuration Example
```yaml
# config/agents.yaml
coder:
  role: >
    Expert Python Developer
  goal: >
    Write clean, efficient Python code to solve complex mathematical problems
  backstory: >
    You are a world-class Python developer...
  tools:
    - code_executor_tool

# config/tasks.yaml
coding_task:
  description: >
    Write and execute Python code for: {assignment}
  expected_output: >
    Working Python code that solves the assignment
  agent: coder
```

### Key Concepts

#### **Code Execution Mode: "safe"**
```python
code_execution_mode="safe"  # Docker isolation
# vs
code_execution_mode="local" # Direct execution (DANGEROUS!)
```

**Docker isolation:**
- Code runs in isolated container
- Can't access host system files
- Automatic cleanup after execution
- Requires Docker Desktop installed

#### **Agent Configuration from YAML**
```python
@agent
def coder(self) -> Agent:
    return Agent(
        config=self.agents_config['coder'],  # Loads from agents.yaml
        verbose=True,                         # Print logs
        allow_code_execution=True
    )
```

**agents.yaml maps to:**
```
{
  'role': '...',
  'goal': '...',
  'backstory': '...',
  'tools': [...]
}
```

### Running the Crew
```bash
# Install dependencies
uv install

# Run crew
crewai run
# OR
python -m coder.main

# Create output file
python -m coder.main > output/result.md
```

---

## **PROJECT 2: DEBATE CREW**

### Purpose
Multi-agent debate system. Arguments for/against a proposition.

### Architecture
- **Proposer**: Makes the case for
- **Opposer**: Makes the case against  
- **Judge**: Evaluates both sides

### Key Patterns

#### **Sequential Processing**
```python
@crew
def crew(self) -> Crew:
    return Crew(
        agents=self.agents,
        tasks=self.tasks,
        process=Process.sequential,  # Run tasks one by one
        verbose=True
    )
```

**Flow:**
```
Task 1 (Agent 1) → Task 2 (Agent 2) → Task 3 (Agent 3)
```

#### **Output Passing**
```yaml
# tasks.yaml
argument_for:
  description: "Generate arguments supporting {proposition}"
  expected_output: "Compelling arguments for the proposition"
  agent: proposer

argument_against:
  description: "Generate counter-arguments against: {previous_task_output}"
  expected_output: "Strong counter-arguments"
  agent: opposer
  
judgment:
  description: "Judge: {argument_for_output} vs {argument_against_output}"
  agent: judge
```

**Task outputs** become available to subsequent tasks!

---

## **PROJECT 3: ENGINEERING TEAM CREW**

### Purpose
Simulate software engineering team. Architecture, implementation, testing.

### Typical Agents
1. **Architect**: Design system
2. **Developer**: Implement code
3. **QA Engineer**: Test code
4. **DevOps**: Deploy

### Process Types

#### **Sequential**
```python
process=Process.sequential
# Task 1 → Task 2 → Task 3
```

#### **Hierarchical** (with manager)
```python
manager = Agent(
    config=self.agents_config['manager'],
    allow_delegation=True
)

return Crew(
    agents=self.agents,
    tasks=self.tasks,
    process=Process.hierarchical,
    manager_agent=manager,  # Coordinator
    verbose=True
)
```

**Hierarchical flow:**
```
User Request
    ↓
Manager (delegates)
    ├→ Agent 1 (Task 1)
    ├→ Agent 2 (Task 2)
    └→ Agent 3 (Task 3)
    ↓
Manager (synthesizes results)
    ↓
Final Output
```

#### **Parallel** 
```python
process=Process.hierarchical  # Allows some parallelism
# or custom parallel implementation
```

### Example YAML Structure
```yaml
# agents.yaml
architect:
  role: Software Architect
  goal: Design scalable system architecture
  backstory: Expert system designer with 15+ years experience

developer:
  role: Senior Developer
  goal: Implement the architecture
  backstory: Full-stack developer, Python expert

qa_engineer:
  role: QA Engineer
  goal: Test implementation thoroughly
  backstory: Rigorous QA specialist
```

---

## **PROJECT 4: FINANCIAL RESEARCHER CREW**

### Purpose
Research companies, analyze financial health, make investment recommendations.

### Key Agents
1. **Financial Analyst**: Analyzes financials
2. **Market Researcher**: Studies market trends
3. **Risk Analyst**: Identifies risks
4. **Investment Advisor**: Makes recommendations

### Tools Integration
```python
from crewai_tools import SerperDevTool

@agent
def market_researcher(self) -> Agent:
    return Agent(
        config=self.agents_config['market_researcher'],
        tools=[SerperDevTool()]  # Web search capability
    )
```

**SerperDevTool:**
- Google Search API integration
- Web research capability
- Real-time information

### Typical Workflow
```
1. Search for financial news
   ↓
2. Analyze company financials
   ↓
3. Research market trends
   ↓
4. Assess risks
   ↓
5. Generate recommendation
```

---

## **PROJECT 5: STOCK PICKER CREW** 

### Purpose
Find trending companies, research them, pick best investment opportunity.

### Architecture Overview

#### **Agents with Tools**
```python
@agent
def trending_company_finder(self) -> Agent:
    return Agent(
        config=self.agents_config['trending_company_finder'],
        tools=[SerperDevTool()],
        memory=True  # Remembers what found earlier
    )

@agent
def financial_researcher(self) -> Agent:
    return Agent(
        config=self.agents_config['financial_researcher'],
        tools=[SerperDevTool()]
    )

@agent
def stock_picker(self) -> Agent:
    return Agent(
        config=self.agents_config['stock_picker'],
        tools=[PushNotificationTool()],  # Custom tool
        memory=True
    )
```

#### **Structured Outputs (Pydantic)**
```python
from pydantic import BaseModel, Field
from typing import List

class TrendingCompany(BaseModel):
    name: str = Field(description="Company name")
    ticker: str = Field(description="Stock ticker symbol")
    reason: str = Field(description="Why this company is trending")

class TrendingCompanyList(BaseModel):
    companies: List[TrendingCompany] = Field(description="List of companies")

@task
def find_trending_companies(self) -> Task:
    return Task(
        config=self.tasks_config['find_trending_companies'],
        output_pydantic=TrendingCompanyList  # Enforce structure!
    )
```

**Benefits:**
- Type-safe output
- Can parse as objects, not strings
- Validation before passing to next task

#### **Memory Systems**

```python
from crewai.memory import LongTermMemory, ShortTermMemory, EntityMemory
from crewai.memory.storage.ltm_sqlite_storage import LTMSQLiteStorage
from crewai.memory.storage.rag_storage import RAGStorage

return Crew(
    agents=self.agents,
    tasks=self.tasks,
    process=Process.hierarchical,
    manager_agent=manager,
    memory=True,
    
    # Persistent storage across sessions
    long_term_memory=LongTermMemory(
        storage=LTMSQLiteStorage(
            db_path="./memory/long_term_memory_storage.db"
        )
    ),
    
    # Current context using RAG (Retrieval-Augmented Generation)
    short_term_memory=ShortTermMemory(
        storage=RAGStorage(
            embedder_config={
                "provider": "openai",
                "config": {"model": "text-embedding-3-small"}
            },
            type="short_term",
            path="./memory/"
        )
    ),
    
    # Track key entities (companies, people, etc.)
    entity_memory=EntityMemory(
        storage=RAGStorage(
            embedder_config={
                "provider": "openai",
                "config": {"model": "text-embedding-3-small"}
            },
            type="entity",
            path="./memory/"
        )
    ),
)a
```

**Memory Types:**

| Type | Purpose | Storage |
|------|---------|---------|
| Long-Term | Persistent facts across sessions | SQLite DB |
| Short-Term | Current context & facts | Vector DB (RAG) |
| Entity | Track key entities & relationships | Vector DB |

#### **Hierarchical Process with Manager**
```python
manager = Agent(
    config=self.agents_config['manager'],
    allow_delegation=True
)

return Crew(
    agents=self.agents,
    tasks=self.tasks,
    process=Process.hierarchical,
    manager_agent=manager,
    verbose=True
)
```

**Manager responsibilities:**
- Delegates tasks to agents
- Coordinates between agents
- Synthesizes final results
- Handles failures & retries

#### **Custom Tool: PushNotificationTool**
```python
from crewai_tools import BaseTool

class PushNotificationTool(BaseTool):
    name = "push_notification"
    description = "Send push notification about stock pick"
    
    def _run(self, message: str) -> str:
        # Implementation
        return "Notification sent"
```

### Task Sequence
```python
@task
def find_trending_companies(self) -> Task:
    return Task(
        config=self.tasks_config['find_trending_companies'],
        output_pydantic=TrendingCompanyList  # Output: List of companies
    )

@task
def research_trending_companies(self) -> Task:
    return Task(
        config=self.tasks_config['research_trending_companies'],
        output_pydantic=TrendingCompanyResearchList  # Output: Research details
    )

@task
def pick_best_company(self) -> Task:
    return Task(
        config=self.tasks_config['pick_best_company']  # Output: Final recommendation
    )
```

**Data Flow:**
```
Find Companies (List[Company])
    ↓
Research Each (List[Research])
    ↓
Pick Best (str recommendation)
```

### Running Stock Picker
```python
def run():
    inputs = {
        'sector': 'Technology',
        "current_date": str(datetime.now())
    }
    
    result = StockPicker().crew().kickoff(inputs=inputs)
    print(result.raw)

if __name__ == "__main__":
    run()
```

---

## **CREWAI SYNTAX REFERENCE**

### Agent Definition (YAML)
```yaml
agent_name:
  role: >
    Your role in one sentence
  goal: >
    What you're trying to achieve
  backstory: >
    Your background and personality
  tools:
    - tool_name_1
    - tool_name_2
  max_iter: 15                 # Max iterations before timeout
  max_execution_time: 30       # Timeout in seconds
  allow_code_execution: true   # For coder only
```

### Task Definition (YAML)
```yaml
task_name:
  description: >
    What the agent needs to do
  expected_output: >
    What success looks like
  agent: agent_name           # Which agent runs this
  tools:                      # Task-specific tools
    - tool_name
  output_file: output.txt     # Save output to file
  output_pydantic: OutputModel # Enforce structure
  context:                    # Reference other tasks
    - previous_task_name
```

### Crew Definition (Python)
```python
from crewai import Agent, Task, Crew, Process

@CrewBase
class MyCrew:
    agents_config = 'config/agents.yaml'
    tasks_config = 'config/tasks.yaml'
    
    @agent
    def my_agent(self) -> Agent:
        return Agent(
            config=self.agents_config['my_agent'],
            tools=[tool1, tool2],
            verbose=True
        )
    
    @task
    def my_task(self) -> Task:
        return Task(
            config=self.tasks_config['my_task'],
            output_pydantic=OutputModel
        )
    
    @crew
    def crew(self) -> Crew:
        return Crew(
            agents=self.agents,
            tasks=self.tasks,
            process=Process.sequential,  # or hierarchical
            manager_agent=manager_agent,  # if hierarchical
            memory=True,
            long_term_memory=ltm,
            short_term_memory=stm,
            entity_memory=em,
            verbose=True
        )
```

### Running the Crew
```python
# In main.py
def run():
    inputs = {
        'key1': 'value1',
        'key2': 'value2'
    }
    result = MyCrew().crew().kickoff(inputs=inputs)
    print(result.raw)

if __name__ == "__main__":
    run()
```

### CLI Commands
```bash
# Run crew
crewai run

# Install dependencies
crewai install

# Test crew
crewai test

# View traces
crewai logs
```

---

## **CORE CONCEPTS**

### 1. **Roles vs Goals vs Backstory**

```yaml
agent:
  role: >
    Data Scientist        # What are you?
  
  goal: >
    Extract insights      # What do you want to achieve?
  
  backstory: >
    MIT PhD, 10 years    # Why are you good at this?
```

### 2. **Process Types**

| Process | Use Case | Flow |
|---------|----------|------|
| **Sequential** | Tasks depend on each other | Task1 → Task2 → Task3 |
| **Hierarchical** | Manager coordinates agents | Manager → [Agents] |
| **Parallel** | Independent tasks | All at once |

### 3. **Tools**

```python
# Built-in tools
from crewai_tools import SerperDevTool, FileWriterTool

# Custom tools
from crewai import Tool

@tool
def my_tool(input: str) -> str:
    """Tool description"""
    return "result"

# Use in agent
Agent(tools=[SerperDevTool(), my_tool])
```

### 4. **Memory Types**

| Type | Purpose | Example |
|------|---------|---------|
| **Long-Term** | Facts across sessions | "Company X revenue is $1B" |
| **Short-Term** | Current task context | "We're researching tech stocks" |
| **Entity** | Track entities | Company profiles, person info |

### 5. **Output Formats**

```python
# String output
@task
def task1(self) -> Task:
    return Task(config=...)
    # Returns: str

# Structured output
from pydantic import BaseModel

class Result(BaseModel):
    name: str
    score: int

@task
def task2(self) -> Task:
    return Task(
        config=...,
        output_pydantic=Result  # Returns: Result object
    )

# JSON output
@task
def task3(self) -> Task:
    return Task(
        config=...,
        output_json={"key": "value"}
    )

# File output
@task
def task4(self) -> Task:
    return Task(
        config=...,
        output_file="results.md"
    )
```

---

## **PROJECT COMPARISON**

| Project | Focus | Process | Complexity | Key Feature |
|---------|-------|---------|-----------|------------|
| **Coder** | Code generation | Sequential | Low-Medium | Docker execution |
| **Debate** | Multi-viewpoint | Sequential | Low | Task chaining |
| **Engineering** | System design | Hierarchical | Medium | Team simulation |
| **Financial** | Investment research | Sequential | Medium | Search tools |
| **StockPicker** | Stock selection | Hierarchical | High | Memory systems |

---

## **INSTALLATION & SETUP**

### Prerequisites
```bash
# Install uv (if not already)
pip install uv

# For coder crew: Docker Desktop
# Download from: https://www.docker.com/products/docker-desktop
```

### Install & Run a Crew
```bash
cd 3_crew/coder

# Install dependencies
uv install

# Run with crewai CLI
crewai run

# Or run directly
python -m coder.main
```

### Environment Variables
```bash
# .env file
OPENAI_API_KEY=sk-...
SERPER_API_KEY=...        # For web search
```

---

## **DEBUGGING & MONITORING**

### Enable Verbose Logging
```python
@crew
def crew(self) -> Crew:
    return Crew(
        agents=self.agents,
        tasks=self.tasks,
        verbose=True  # Print detailed logs
    )
```

### View Execution Traces
```bash
# Check logs
tail -f crewai.log

# Or via CrewAI dashboard (if available)
crewai dashboard
```

### Handle Agent Failures
```python
@agent
def my_agent(self) -> Agent:
    return Agent(
        config=self.agents_config['my_agent'],
        max_iter=15,              # Max iterations before timeout
        max_execution_time=30,    # Timeout in seconds
        max_retry_limit=3         # Retry failed tasks
    )
```

---

## **ADVANCED PATTERNS**

### Conditional Task Execution
```yaml
task_a:
  description: Research phase
  agent: researcher

task_b:
  description: Analysis phase (depends on task_a)
  agent: analyst
  context:
    - task_a  # Waits for task_a to complete
```

### Parallel Agent Execution
```python
# Use asyncio for parallel crews
import asyncio

async def run_crews_parallel():
    results = await asyncio.gather(
        asyncio.create_task(Coder().crew().kickoff_async(inputs=inputs1)),
        asyncio.create_task(Debate().crew().kickoff_async(inputs=inputs2)),
    )
    return results
```

### Inter-Agent Communication
```yaml
task_with_context:
  description: >
    Based on the research from task1: {task_1_output}
    Provide your analysis
  context:
    - task_1  # Reference previous task
```

## **PRODUCTION BEST PRACTICES**

1. **Use Configuration Files**
   ```bash
   # Don't hardcode - use config/
   ✓ agents.yaml, tasks.yaml
   ✗ Agent(role="...", goal="...")
   ```

2. **Enable Memory Systems**
   ```python
   # For any production crew
   memory=True,
   long_term_memory=LongTermMemory(...),
   short_term_memory=ShortTermMemory(...)
   ```

3. **Add Error Handling**
   ```python
   try:
       result = crew.kickoff(inputs=inputs)
   except Exception as e:
       logger.error(f"Crew failed: {e}")
       # Fallback logic
   ```

4. **Monitor & Log**
   ```python
   verbose=True,  # Always in production
   # Plus external monitoring (DataDog, etc.)
   ```

5. **Security**
   ```python
   # For coder crew
   code_execution_mode="safe",  # Docker isolation
   max_execution_time=30,       # Timeout protection
   ```

---

---

## **KEY TAKEAWAYS**

✓ CrewAI separates **configuration** (YAML) from **orchestration** (Python)
✓ Agents have **roles, goals, backstory** - defined in YAML
✓ Tasks are units of work with **expected outputs** and **agent assignments**
✓ **Processes** control execution flow (sequential, hierarchical, etc.)
✓ **Memory systems** enable learning and context persistence
✓ **Tools** extend agent capabilities (search, code, custom)
✓ **Structured outputs** (Pydantic) ensure type safety
✓ **Hierarchical process** with manager agent for complex orchestration
✓ Perfect for **production** multi-agent systems
✓ Higher learning curve than raw API, but more powerful at scale
