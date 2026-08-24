---
title: Build a Ready-to-Use AI-Powered Project Management Tool from Scratch
date: 2026-08-24T09:00:00.000Z
author: Jyothish Narayan
category: AI
summary: A step-by-step developer guide to building a full-stack AI-powered project management tool — with natural language task creation, smart prioritisation, deadline estimation, and a clean React frontend backed by Supabase and FastAPI.
---

Project management tools are everywhere. But most of them treat AI as a marketing badge — a thin layer of autocomplete on top of a glorified spreadsheet. In this post, we're building something different: a PM tool where AI is woven into the core workflow. Tasks get created from plain English. Priorities are scored automatically. Deadlines are estimated from historical patterns. And the whole thing is yours to own, deploy, and extend.

By the end of this guide you'll have a running application with:
- Natural language → structured task conversion
- AI-powered priority scoring (urgency + impact matrix)
- Deadline estimation based on task complexity
- A clean React frontend
- Supabase as the backend database
- FastAPI handling AI orchestration
- Deployable on Cloudflare Pages + a $5/month VPS

---

## Architecture overview

Before touching code, let's be clear about what we're building and why each piece exists.

```
┌─────────────────────────────────────────────────────────┐
│                     React Frontend                       │
│              (Cloudflare Pages — free)                   │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTP / REST
┌──────────────────────▼──────────────────────────────────┐
│                  FastAPI Backend                         │
│         (VPS — handles AI orchestration)                 │
│                                                          │
│  ┌─────────────────┐    ┌──────────────────────────┐    │
│  │  Task Parser    │    │   Priority Scorer         │    │
│  │  (LLM → JSON)  │    │   (urgency × impact)      │    │
│  └─────────────────┘    └──────────────────────────┘    │
│  ┌─────────────────┐    ┌──────────────────────────┐    │
│  │ Deadline Engine │    │   Progress Analyser       │    │
│  │ (complexity est)│    │   (velocity tracking)     │    │
│  └─────────────────┘    └──────────────────────────┘    │
└──────────────────────┬──────────────────────────────────┘
                       │ Supabase client
┌──────────────────────▼──────────────────────────────────┐
│                    Supabase                              │
│         PostgreSQL + pgvector + Realtime                 │
│   projects | tasks | sprints | time_logs | embeddings    │
└─────────────────────────────────────────────────────────┘
```

The frontend is a static React app — no server needed for it. The FastAPI backend is the only piece that needs a server, because it holds your LLM API keys and does the heavy AI lifting. Supabase handles persistence and real-time updates.

---

## Tech stack

| Layer | Tool | Why |
|---|---|---|
| Frontend | React + Vite + Tailwind | Fast builds, great DX |
| Backend | FastAPI (Python) | Async, clean, great for AI |
| Database | Supabase (PostgreSQL) | Built-in auth, realtime, pgvector |
| AI | OpenAI GPT-4o | Function calling for structured output |
| Embeddings | `text-embedding-3-small` | Semantic task search |
| Deploy (frontend) | Cloudflare Pages | Free, global CDN |
| Deploy (backend) | Any VPS (Hetzner/DigitalOcean) | $5–6/month |

---

## Step 1 — Database schema

Create a new Supabase project, open the SQL editor, and run:

```sql
-- Enable pgvector for semantic search
create extension if not exists vector;

-- Projects
create table projects (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  description text,
  status text default 'active' check (status in ('active','paused','completed','archived')),
  owner_id uuid references auth.users(id),
  created_at timestamptz default now()
);

-- Tasks
create table tasks (
  id uuid primary key default gen_random_uuid(),
  project_id uuid references projects(id) on delete cascade,
  title text not null,
  description text,
  status text default 'todo' check (status in ('todo','in_progress','review','done','blocked')),
  priority int default 50 check (priority between 0 and 100),
  ai_priority_score float,
  estimated_hours float,
  actual_hours float default 0,
  deadline timestamptz,
  ai_suggested_deadline timestamptz,
  assignee_id uuid references auth.users(id),
  tags text[] default '{}',
  embedding vector(1536),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Sprints
create table sprints (
  id uuid primary key default gen_random_uuid(),
  project_id uuid references projects(id) on delete cascade,
  name text not null,
  start_date date,
  end_date date,
  velocity float,
  status text default 'planned'
);

-- Sprint tasks (many-to-many)
create table sprint_tasks (
  sprint_id uuid references sprints(id) on delete cascade,
  task_id uuid references tasks(id) on delete cascade,
  primary key (sprint_id, task_id)
);

-- Time logs
create table time_logs (
  id uuid primary key default gen_random_uuid(),
  task_id uuid references tasks(id) on delete cascade,
  user_id uuid references auth.users(id),
  hours float not null,
  logged_at timestamptz default now(),
  note text
);

-- Index for fast vector search
create index on tasks using ivfflat (embedding vector_cosine_ops) with (lists = 50);

-- Index for common queries
create index on tasks(project_id, status);
create index on tasks(priority desc);

-- Auto-update updated_at
create or replace function update_updated_at()
returns trigger as $$
begin new.updated_at = now(); return new; end;
$$ language plpgsql;

create trigger tasks_updated_at
  before update on tasks
  for each row execute function update_updated_at();

-- Row-level security
alter table projects enable row level security;
alter table tasks enable row level security;

create policy "Users see their projects" on projects
  for all using (owner_id = auth.uid());

create policy "Users see tasks in their projects" on tasks
  for all using (
    project_id in (select id from projects where owner_id = auth.uid())
  );
```

---

## Step 2 — FastAPI backend setup

```bash
mkdir ai-pm-backend && cd ai-pm-backend
python -m venv venv && source venv/bin/activate
pip install fastapi uvicorn openai supabase python-dotenv pydantic
```

Project structure:

```
ai-pm-backend/
├── main.py
├── routers/
│   ├── tasks.py
│   ├── projects.py
│   └── ai.py
├── services/
│   ├── task_parser.py
│   ├── priority_scorer.py
│   ├── deadline_engine.py
│   └── embedding_service.py
├── models.py
└── .env
```

`.env`:
```
OPENAI_API_KEY=sk-...
SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_KEY=eyJ...
FRONTEND_URL=https://nexamindlabs.com
```

`main.py`:
```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from routers import tasks, projects, ai
import os

app = FastAPI(title="AI PM Tool API", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=[os.getenv("FRONTEND_URL", "*")],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(tasks.router, prefix="/tasks", tags=["tasks"])
app.include_router(projects.router, prefix="/projects", tags=["projects"])
app.include_router(ai.router, prefix="/ai", tags=["ai"])

@app.get("/health")
def health(): return {"status": "ok"}
```

---

## Step 3 — The AI task parser

This is the most important piece. The user types a sentence like:

> *"Need to fix the checkout bug — it's breaking on mobile Safari, blocking the launch, should take about 3 hours"*

And the parser converts it into a structured task object.

`services/task_parser.py`:

```python
from openai import AsyncOpenAI
import json

client = AsyncOpenAI()

PARSE_SYSTEM_PROMPT = """
You are a project management assistant. Extract structured task data from 
natural language input. Always respond with valid JSON only — no markdown, 
no explanation, just the JSON object.

Extract:
- title: concise task title (max 80 chars)
- description: detailed description of what needs to be done
- estimated_hours: numeric estimate (float), null if unclear
- tags: array of relevant tags (tech, bug, feature, design, etc.)
- urgency: 1-5 scale (5 = blocking/critical, 1 = nice to have)
- impact: 1-5 scale (5 = affects all users, 1 = minor internal)
- complexity: 1-5 scale (5 = very complex, 1 = trivial)
- suggested_deadline_days: days from today, null if not mentioned

Respond ONLY with JSON like:
{
  "title": "...",
  "description": "...",
  "estimated_hours": 3.0,
  "tags": ["bug", "mobile"],
  "urgency": 5,
  "impact": 4,
  "complexity": 2,
  "suggested_deadline_days": 1
}
"""

async def parse_task_from_text(raw_text: str) -> dict:
    """Convert natural language to structured task data."""
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": PARSE_SYSTEM_PROMPT},
            {"role": "user", "content": raw_text}
        ],
        temperature=0.1,  # Low temp for consistent structured output
        max_tokens=500
    )
    
    content = response.choices[0].message.content.strip()
    
    try:
        return json.loads(content)
    except json.JSONDecodeError:
        # Fallback: extract JSON from response if model added text
        import re
        match = re.search(r'\{.*\}', content, re.DOTALL)
        if match:
            return json.loads(match.group())
        raise ValueError(f"Could not parse JSON from model response: {content}")
```

---

## Step 4 — Priority scoring engine

Once we have urgency and impact from the parser, we calculate a 0–100 priority score. This score is what determines task ordering in the UI.

`services/priority_scorer.py`:

```python
from datetime import datetime, timezone
from typing import Optional

def calculate_priority_score(
    urgency: int,          # 1-5
    impact: int,           # 1-5
    complexity: int,       # 1-5 (higher = harder = lower priority bump)
    deadline: Optional[datetime] = None,
    estimated_hours: Optional[float] = None,
) -> float:
    """
    Priority score 0-100.
    
    Formula:
    - Base score from urgency × impact matrix (0-100)
    - Deadline pressure modifier (+0 to +20)
    - Complexity penalty (-0 to -10, very complex tasks get slightly deprioritised
      unless urgency is max)
    """
    # Urgency × Impact matrix → base score
    # Max: 5×5 = 25, normalised to 80 (leaving 20 for deadline pressure)
    base = (urgency * impact / 25) * 80

    # Deadline pressure: tasks due soon get bumped up
    deadline_bonus = 0.0
    if deadline:
        now = datetime.now(timezone.utc)
        if deadline.tzinfo is None:
            deadline = deadline.replace(tzinfo=timezone.utc)
        days_until = (deadline - now).total_seconds() / 86400

        if days_until <= 0:
            deadline_bonus = 20  # Overdue — max bonus
        elif days_until <= 1:
            deadline_bonus = 18
        elif days_until <= 3:
            deadline_bonus = 12
        elif days_until <= 7:
            deadline_bonus = 6
        elif days_until <= 14:
            deadline_bonus = 3

    # Complexity penalty (only if not critical urgency)
    complexity_penalty = 0.0
    if urgency < 5:
        complexity_penalty = (complexity - 1) * 2.5  # max -10

    score = base + deadline_bonus - complexity_penalty
    return round(max(0, min(100, score)), 2)


def suggest_deadline(
    complexity: int,
    estimated_hours: Optional[float],
    urgency: int
) -> int:
    """
    Suggest a deadline in calendar days from today.
    Based on complexity + estimated hours + urgency.
    """
    if urgency == 5:
        return 1  # Critical → tomorrow

    # Base days from complexity
    base_days = {1: 3, 2: 5, 3: 7, 4: 10, 5: 14}[complexity]

    # Adjust for hour estimate
    if estimated_hours:
        if estimated_hours <= 2:
            base_days = max(1, base_days - 2)
        elif estimated_hours > 8:
            base_days = int(base_days * 1.5)

    # Urgency discount
    urgency_factor = {1: 1.4, 2: 1.2, 3: 1.0, 4: 0.8, 5: 0.5}[urgency]
    return max(1, int(base_days * urgency_factor))
```

---

## Step 5 — AI router (the main API endpoint)

`routers/ai.py`:

```python
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
from datetime import datetime, timedelta, timezone
from services.task_parser import parse_task_from_text
from services.priority_scorer import calculate_priority_score, suggest_deadline
from services.embedding_service import generate_embedding
from supabase import create_client
import os

router = APIRouter()
supabase = create_client(os.getenv("SUPABASE_URL"), os.getenv("SUPABASE_SERVICE_KEY"))


class TaskInput(BaseModel):
    raw_text: str
    project_id: str
    user_id: str


class BulkTaskInput(BaseModel):
    description: str   # Project/epic description
    project_id: str
    user_id: str
    num_tasks: int = 5


@router.post("/parse-task")
async def parse_and_create_task(payload: TaskInput):
    """
    Accept natural language, return a structured task ready to insert.
    Also calculates priority score and suggested deadline.
    """
    try:
        parsed = await parse_task_from_text(payload.raw_text)
    except Exception as e:
        raise HTTPException(status_code=422, detail=f"Parse error: {str(e)}")

    # Calculate priority score
    priority = calculate_priority_score(
        urgency=parsed.get("urgency", 3),
        impact=parsed.get("impact", 3),
        complexity=parsed.get("complexity", 3),
    )

    # Suggest deadline
    deadline_days = parsed.get("suggested_deadline_days") or suggest_deadline(
        complexity=parsed.get("complexity", 3),
        estimated_hours=parsed.get("estimated_hours"),
        urgency=parsed.get("urgency", 3),
    )

    suggested_deadline = (
        datetime.now(timezone.utc) + timedelta(days=deadline_days)
    ).isoformat()

    # Generate embedding for semantic search
    embedding = await generate_embedding(
        f"{parsed['title']} {parsed.get('description', '')}"
    )

    task_data = {
        "project_id": payload.project_id,
        "title": parsed["title"],
        "description": parsed.get("description", ""),
        "estimated_hours": parsed.get("estimated_hours"),
        "tags": parsed.get("tags", []),
        "ai_priority_score": priority,
        "priority": int(priority),
        "ai_suggested_deadline": suggested_deadline,
        "embedding": embedding,
        "status": "todo",
    }

    # Insert into Supabase
    result = supabase.table("tasks").insert(task_data).execute()
    
    if not result.data:
        raise HTTPException(status_code=500, detail="Failed to insert task")

    return {
        "task": result.data[0],
        "ai_insights": {
            "urgency": parsed.get("urgency"),
            "impact": parsed.get("impact"),
            "complexity": parsed.get("complexity"),
            "priority_score": priority,
            "suggested_deadline_days": deadline_days,
            "explanation": _explain_priority(parsed, priority)
        }
    }


@router.post("/breakdown-epic")
async def breakdown_epic(payload: BulkTaskInput):
    """
    Given a feature/epic description, generate a list of sub-tasks using AI.
    """
    from openai import AsyncOpenAI
    import json

    client = AsyncOpenAI()

    prompt = f"""
    Break down the following feature/epic into {payload.num_tasks} specific, 
    actionable developer tasks. Each task should be completable in 1-2 days.
    
    Feature/Epic: {payload.description}
    
    Respond ONLY with a JSON array:
    [
      {{
        "title": "...",
        "description": "...",
        "estimated_hours": 4,
        "tags": ["backend"],
        "urgency": 3,
        "impact": 4,
        "complexity": 2,
        "order": 1
      }},
      ...
    ]
    """

    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.3,
        max_tokens=2000
    )

    try:
        tasks = json.loads(response.choices[0].message.content)
    except:
        raise HTTPException(status_code=422, detail="Could not parse AI response")

    created_tasks = []
    for t in tasks:
        priority = calculate_priority_score(
            urgency=t.get("urgency", 3),
            impact=t.get("impact", 3),
            complexity=t.get("complexity", 3),
        )
        embedding = await generate_embedding(t["title"])

        task_data = {
            "project_id": payload.project_id,
            "title": t["title"],
            "description": t.get("description", ""),
            "estimated_hours": t.get("estimated_hours"),
            "tags": t.get("tags", []),
            "ai_priority_score": priority,
            "priority": int(priority),
            "embedding": embedding,
            "status": "todo",
        }
        result = supabase.table("tasks").insert(task_data).execute()
        if result.data:
            created_tasks.append(result.data[0])

    return {"tasks": created_tasks, "count": len(created_tasks)}


@router.get("/similar-tasks/{task_id}")
async def find_similar_tasks(task_id: str, limit: int = 5):
    """Find semantically similar tasks using pgvector."""
    # Get the task's embedding
    task = supabase.table("tasks").select("embedding, project_id").eq("id", task_id).single().execute()
    if not task.data:
        raise HTTPException(status_code=404, detail="Task not found")

    embedding = task.data["embedding"]
    project_id = task.data["project_id"]

    # Use Supabase RPC for vector similarity search
    result = supabase.rpc("match_tasks", {
        "query_embedding": embedding,
        "match_count": limit + 1,
        "project_filter": project_id
    }).execute()

    # Filter out the task itself
    similar = [t for t in (result.data or []) if t["id"] != task_id][:limit]
    return {"similar_tasks": similar}


def _explain_priority(parsed: dict, score: float) -> str:
    u, i = parsed.get("urgency", 3), parsed.get("impact", 3)
    if score >= 80:
        return f"Critical priority — high urgency ({u}/5) and high impact ({i}/5). Address immediately."
    elif score >= 60:
        return f"High priority — urgency {u}/5, impact {i}/5. Schedule this sprint."
    elif score >= 40:
        return f"Medium priority — urgency {u}/5, impact {i}/5. Plan for next sprint."
    else:
        return f"Low priority — urgency {u}/5, impact {i}/5. Backlog for now."
```

---

## Step 6 — Embedding service

`services/embedding_service.py`:

```python
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def generate_embedding(text: str) -> list[float]:
    """Generate a 1536-dimension embedding for semantic search."""
    response = await client.embeddings.create(
        model="text-embedding-3-small",
        input=text[:8000]  # Truncate to model limit
    )
    return response.data[0].embedding
```

Add the Supabase RPC function for similarity search (run in SQL editor):

```sql
create or replace function match_tasks(
  query_embedding vector(1536),
  match_count int,
  project_filter uuid
)
returns table (
  id uuid,
  title text,
  status text,
  priority int,
  similarity float
)
language sql stable as $$
  select
    id, title, status, priority,
    1 - (embedding <=> query_embedding) as similarity
  from tasks
  where project_id = project_filter
    and embedding is not null
  order by embedding <=> query_embedding
  limit match_count;
$$;
```

---

## Step 7 — React frontend

```bash
npm create vite@latest ai-pm-frontend -- --template react
cd ai-pm-frontend
npm install @supabase/supabase-js axios @tanstack/react-query
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

The key component is the natural language task input:

`src/components/TaskInput.jsx`:

```jsx
import { useState } from 'react';
import axios from 'axios';

const API_URL = import.meta.env.VITE_API_URL;

export default function TaskInput({ projectId, userId, onTaskCreated }) {
  const [text, setText] = useState('');
  const [loading, setLoading] = useState(false);
  const [preview, setPreview] = useState(null);
  const [error, setError] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!text.trim()) return;

    setLoading(true);
    setError(null);

    try {
      const { data } = await axios.post(`${API_URL}/ai/parse-task`, {
        raw_text: text,
        project_id: projectId,
        user_id: userId,
      });

      setPreview(data);
      setText('');
      onTaskCreated(data.task);
    } catch (err) {
      setError(err.response?.data?.detail || 'Something went wrong');
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="bg-white rounded-2xl border border-slate-200 p-6 shadow-sm">
      <label className="block text-sm font-semibold text-slate-500 uppercase 
                        tracking-wider mb-3">
        Describe a task in plain English
      </label>
      <form onSubmit={handleSubmit} className="flex gap-3">
        <input
          type="text"
          value={text}
          onChange={(e) => setText(e.target.value)}
          placeholder="e.g. Fix the payment bug — it's breaking on Safari, blocking launch, about 2 hours"
          className="flex-1 px-4 py-3 rounded-xl border border-slate-200 
                     text-sm focus:outline-none focus:border-indigo-500 
                     focus:ring-2 focus:ring-indigo-100"
          disabled={loading}
        />
        <button
          type="submit"
          disabled={loading || !text.trim()}
          className="px-5 py-3 bg-indigo-600 text-white rounded-xl text-sm 
                     font-semibold disabled:opacity-50 hover:bg-indigo-700 
                     transition-colors"
        >
          {loading ? 'Thinking…' : 'Add task →'}
        </button>
      </form>

      {error && (
        <p className="mt-3 text-sm text-red-500">{error}</p>
      )}

      {preview && (
        <div className="mt-4 p-4 bg-indigo-50 rounded-xl border border-indigo-100">
          <p className="text-xs font-semibold text-indigo-400 uppercase 
                        tracking-wider mb-2">AI insights</p>
          <p className="text-sm font-semibold text-slate-800 mb-1">
            {preview.task.title}
          </p>
          <div className="flex gap-3 flex-wrap mt-2">
            <PriorityBadge score={preview.ai_insights.priority_score} />
            {preview.task.tags?.map(tag => (
              <span key={tag} className="px-2 py-0.5 bg-white border border-slate-200 
                                         rounded-full text-xs text-slate-500">
                {tag}
              </span>
            ))}
            {preview.task.estimated_hours && (
              <span className="text-xs text-slate-500">
                ⏱ ~{preview.task.estimated_hours}h
              </span>
            )}
          </div>
          <p className="mt-2 text-xs text-indigo-600">
            {preview.ai_insights.explanation}
          </p>
        </div>
      )}
    </div>
  );
}

function PriorityBadge({ score }) {
  const config = score >= 80
    ? { label: 'Critical', cls: 'bg-red-100 text-red-600' }
    : score >= 60
    ? { label: 'High', cls: 'bg-orange-100 text-orange-600' }
    : score >= 40
    ? { label: 'Medium', cls: 'bg-yellow-100 text-yellow-700' }
    : { label: 'Low', cls: 'bg-green-100 text-green-600' };

  return (
    <span className={`px-2 py-0.5 rounded-full text-xs font-semibold ${config.cls}`}>
      {config.label} ({Math.round(score)})
    </span>
  );
}
```

`src/components/TaskBoard.jsx` — Kanban view:

```jsx
import { useQuery } from '@tanstack/react-query';
import { supabase } from '../lib/supabase';

const COLUMNS = ['todo', 'in_progress', 'review', 'done'];
const COLUMN_LABELS = {
  todo: 'To Do',
  in_progress: 'In Progress',
  review: 'Review',
  done: 'Done'
};

export default function TaskBoard({ projectId }) {
  const { data: tasks = [], refetch } = useQuery({
    queryKey: ['tasks', projectId],
    queryFn: async () => {
      const { data } = await supabase
        .from('tasks')
        .select('*')
        .eq('project_id', projectId)
        .order('priority', { ascending: false });
      return data || [];
    }
  });

  const updateStatus = async (taskId, newStatus) => {
    await supabase.from('tasks').update({ status: newStatus }).eq('id', taskId);
    refetch();
  };

  const byStatus = (status) => tasks.filter(t => t.status === status);

  return (
    <div className="grid grid-cols-4 gap-4 mt-6">
      {COLUMNS.map(col => (
        <div key={col} className="bg-slate-50 rounded-2xl p-4">
          <div className="flex items-center justify-between mb-4">
            <h3 className="text-sm font-semibold text-slate-600">
              {COLUMN_LABELS[col]}
            </h3>
            <span className="text-xs bg-slate-200 text-slate-500 
                             rounded-full px-2 py-0.5">
              {byStatus(col).length}
            </span>
          </div>
          <div className="space-y-3">
            {byStatus(col).map(task => (
              <TaskCard key={task.id} task={task} onStatusChange={updateStatus} />
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}

function TaskCard({ task, onStatusChange }) {
  const priorityColor = task.priority >= 80 ? 'border-l-red-400'
    : task.priority >= 60 ? 'border-l-orange-400'
    : task.priority >= 40 ? 'border-l-yellow-400'
    : 'border-l-green-400';

  return (
    <div className={`bg-white rounded-xl p-3.5 border border-slate-200 
                     border-l-4 ${priorityColor} shadow-sm`}>
      <p className="text-sm font-medium text-slate-800 leading-snug mb-2">
        {task.title}
      </p>
      <div className="flex items-center justify-between">
        <div className="flex gap-1 flex-wrap">
          {task.tags?.slice(0, 2).map(tag => (
            <span key={tag} className="text-xs text-slate-400 bg-slate-100 
                                        px-1.5 py-0.5 rounded">
              {tag}
            </span>
          ))}
        </div>
        {task.estimated_hours && (
          <span className="text-xs text-slate-400">
            {task.estimated_hours}h
          </span>
        )}
      </div>
      <select
        value={task.status}
        onChange={(e) => onStatusChange(task.id, e.target.value)}
        className="mt-2 w-full text-xs border border-slate-200 rounded-lg 
                   px-2 py-1 text-slate-500 focus:outline-none"
      >
        <option value="todo">To Do</option>
        <option value="in_progress">In Progress</option>
        <option value="review">Review</option>
        <option value="done">Done</option>
        <option value="blocked">Blocked</option>
      </select>
    </div>
  );
}
```

---

## Step 8 — Connect to Supabase Realtime

Subscribe to task changes so the board updates instantly when a teammate moves a card — no polling, no refresh.

`src/lib/supabase.js`:

```javascript
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY
);
```

In `TaskBoard.jsx`, add real-time subscription:

```jsx
import { useEffect } from 'react';

// Inside TaskBoard component, after the useQuery:
useEffect(() => {
  const channel = supabase
    .channel(`tasks-${projectId}`)
    .on(
      'postgres_changes',
      { event: '*', schema: 'public', table: 'tasks',
        filter: `project_id=eq.${projectId}` },
      () => refetch()
    )
    .subscribe();

  return () => supabase.removeChannel(channel);
}, [projectId]);
```

---

## Step 9 — Deploy

**Backend (FastAPI):**

```bash
# On your VPS
pip install gunicorn
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000

# Use nginx as reverse proxy + SSL via Certbot
# Point api.yourdomain.com → localhost:8000
```

**Frontend (Cloudflare Pages):**

```bash
# Build
npm run build

# Set environment variables in Cloudflare Pages dashboard:
# VITE_API_URL = https://api.yourdomain.com
# VITE_SUPABASE_URL = https://yourproject.supabase.co
# VITE_SUPABASE_ANON_KEY = eyJ...
```

---

## What to build next

This is a solid foundation. Once this is running, the natural extensions are:

**Sprint planning with AI** — describe the sprint goal, AI selects the most appropriate tasks from the backlog based on priority scores and estimated hours vs sprint capacity.

**Velocity tracking** — after a few sprints, the deadline engine can calibrate itself against actual vs estimated hours per developer and per task type.

**Slack/Teams integration** — post a message in your team channel, have it create a task automatically. The task parser works on any text input.

**Meeting notes → tasks** — paste meeting notes, get a list of action items as structured tasks, assigned and prioritised, in one click.

The full source code for this project is available to fork and extend. Drop a message at [info@nexamindlabs.com](mailto:info@nexamindlabs.com) if you'd like us to build or customise this for your team.
