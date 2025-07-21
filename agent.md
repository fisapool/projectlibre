# 🧠 ProjectLibre Agent

## Overview
This agent uses OpenAI LLM to reason about project management tasks by calling ProjectLibre API endpoints (`/export`, `/tasks`, `/run`). It follows a ReAct-style loop: **Reason → Act → Observe → Repeat**.

---

## Setup

```bash
pip install langchain openai requests
```

Create `.env`:

```
OPENAI_API_KEY=your_key_here
PROJECTLIBRE_API_URL=http://localhost:3000
```

---

## Agent Implementation (`agent.py`)

```python
import os
import requests
from langchain import OpenAI, LLMChain
from langchain.agents import Tool, initialize_agent, AgentType
from langchain.prompts import PromptTemplate

# Load keys & API
llm = OpenAI(temperature=0)
API_URL = os.getenv("PROJECTLIBRE_API_URL")

# Tools for agent
def fetch_project_state(project_id: str) -> dict:
    resp = requests.get(f"{API_URL}/export/{project_id}")
    return resp.json()

def add_task(project_id: str, task: dict) -> dict:
    resp = requests.post(f"{API_URL}/tasks", json={"projectId": project_id, "task": task})
    return resp.json()

def run_scheduler(project_id: str) -> dict:
    resp = requests.post(f"{API_URL}/run", json={"projectId": project_id})
    return resp.json()

tools = [
    Tool(name="get_state", func=fetch_project_state, description="Get current project JSON"),
    Tool(name="add_task", func=add_task, description="Add a task dict"),
    Tool(name="run", func=run_scheduler, description="Re-calc schedule")
]

# Agent prompt template
template = """
You are a project management assistant. Use the tools to modify or inspect the project.

User request: "{input}"

When making a change:
- Use get_state to inspect current plan.
- Use add_task to schedule tasks (pass name, start, duration, dependencies).
- Use run to recalc schedule.

Return final schedule JSON or confirmation.
"""
prompt = PromptTemplate(template=template, input_variables=["input"])
chain = LLMChain(llm=llm, prompt=prompt)

agent = initialize_agent(
    tools, chain, agent=AgentType.ZERO_SHOT_REACT_DESCRIPTION, verbose=True
)

# Run agent example
if __name__ == "__main__":
    project_id = "my_project"
    user_input = "Add a 5-day QA task after 'Develop feature X'."
    result = agent.run({"input": user_input})
    print(result)
```

---

## 🧩 How It Works

1. **Query**: Gets project state via `get_state`.
2. **Plan**: LLM reasons and decides to call `add_task`.
3. **Act**: Calls API to mutate project.
4. **Finalize**: Uses `run` to re-schedule, then returns results.

---

## 🔧 Next Steps

* Tailor the prompt instructions (e.g., enforce structured JSON output).
* Add tools for updating tasks, deleting tasks, or adding resources.
* Add memory or long-term planning (via embeddings or chain-of-thought).
* Monitor logs and error handling for production use.

---

## ⚙️ Agent Loop Pattern

```
[
  Thought: what’s current state?
  Action: get_state
  Observation: {...}
  Thought: need to add task
  Action: add_task
  Observation: {...}
  Thought: recalc schedule
  Action: run
  Observation: {...}
  Final Answer: ✅ Task added and schedule updated.
]
```

---

Drop this `agent.md` into your repo. It gives you a working LLM agent that can:

* Plan actions
* Execute via your ProjectLibre API
* React and iterate until logic is satisfied

If you want a Spring Boot version of tools or structured JSON validation, just say the word.
