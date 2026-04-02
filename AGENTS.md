# Agent Guidelines for langchain-miniopenclaw

This file provides coding conventions and operational guidance for agentic coding systems operating in this repository.

---

## Project Overview

**langchain-OpenClaw** is a local-first, file-driven AI Agent workstation with transparent memory, skills, and prompt assembly.

```
mini-openclaw/
├── backend/           # Python 3.10+ FastAPI backend
│   ├── api/            # REST endpoints (chat, sessions, files, config)
│   ├── graph/          # Agent logic, prompt builder, session manager
│   ├── tools/          # Agent tools (terminal, python_repl, fetch_url, etc.)
│   ├── skills/         # Skill definitions (each skill = SKILL.md)
│   ├── memory/         # Long-term memory (MEMORY.md)
│   ├── knowledge/      # Local knowledge base
│   ├── sessions/       # Session JSON files
│   ├── storage/       # Index caches (gitkeep-tracked)
│   └── app.py          # FastAPI entry point
└── frontend/          # Next.js 14 App Router frontend
    └── src/
        ├── app/        # Page routes
        ├── components/ # UI components (chat, editor, layout)
        └── lib/        # API client, global state (Zustand-lite)
```

---

## Build / Lint / Test Commands

### Backend (Python)

```bash
cd backend

# Create venv and install dependencies
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate
pip install -r requirements.txt

# Run development server
uvicorn app:app --host 0.0.0.0 --port 8002 --reload

# No formal test framework configured yet
# Manual testing via frontend or curl:
curl -X POST http://127.0.0.1:8002/api/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"hello","session_id":"test","stream":false}'
```

### Frontend (TypeScript/Next.js)

```bash
cd frontend

# Install dependencies
npm install

# Development
npm run dev

# Production build
npm run build
npm run start

# Linting (uses Next.js ESLint config)
npm run lint
```

### Environment Setup

```bash
# Backend requires .env file (copy from .env.example)
cp backend/.env.example backend/.env
# Edit backend/.env and fill in required API keys:
#   ZHIPU_API_KEY=your_key        # Default LLM
#   BAILIAN_API_KEY=your_key      # Default embedding
#   TAVILY_API_KEY=your_key       # Optional web search
```

---

## Code Style Guidelines

### Python (Backend)

**Style Reference:** The codebase uses standard Python conventions with `from __future__ import annotations` for forward references.

#### Imports
- Always use `from __future__ import annotations` at the top of every Python file
- Use absolute imports within the backend package: `from api.chat import router`
- Pydantic v2 is used for models: `from pydantic import BaseModel, Field`
- LangChain imports follow official patterns: `from langchain.agents import create_agent`

```python
from __future__ import annotations

import json
from pathlib import Path
from typing import Any

from fastapi import APIRouter, HTTPException
from pydantic import BaseModel, Field
```

#### Type Annotations
- Use `Path` from `pathlib`, not strings, for file paths
- Use `dict[str, Any]` not `Dict[str, Any]` (Python 3.9+)
- Use `| None` union syntax, not `Optional[None]`
- Pydantic models use `Type[]` for `args_schema`: `args_schema: Type[BaseModel] = TerminalToolInput`

```python
def _stringify_content(content: Any) -> str:
    if isinstance(content, str):
        return content
    return str(content or "")

class Settings:
    llm_api_key: str | None  # nullable string
```

#### Naming Conventions
- Classes: `PascalCase` (e.g., `SessionManager`, `AgentManager`)
- Functions/methods: `snake_case` (e.g., `create_session`, `load_session_for_agent`)
- Private attributes: prefix with `_` (e.g., `_root_dir`, `_config_path`)
- Constants: `UPPER_SNAKE_CASE` for module-level constants (e.g., `LLM_PROVIDER_DEFAULTS`)
- Variables: `snake_case` (e.g., `session_id`, `base_dir`)

#### Error Handling
- Use typed exceptions when possible
- Bare `except Exception` allowed in async generators for stream cleanup
- JSON decode errors: catch `json.JSONDecodeError` specifically
- Config errors: use `RuntimeError` with descriptive messages

```python
try:
    response = await self._build_chat_model().ainvoke(messages)
except Exception:  # broad allowed in async generators
    return fallback_value
```

#### FastAPI Patterns
- Use `APIRouter` for route modules
- Use Pydantic `BaseModel` for request/response schemas
- Return `StreamingResponse` for SSE chat endpoints
- Use `lifespan` context manager for startup/shutdown (not `@app.on_event`)

```python
@router.post("/chat")
async def chat(payload: ChatRequest):
    ...

app = FastAPI(title="Mini-OpenClaw API", version="0.1.0", lifespan=lifespan)
```

#### LangChain Patterns
- Use `create_agent` from `langchain.agents`
- Use `ChatOpenAI` or provider-specific wrappers (e.g., `ChatDeepSeek`)
- Tool classes extend `BaseTool` with typed `args_schema`
- Async streaming via `agent.astream()` with `stream_mode=["messages", "updates"]`

---

### TypeScript / React (Frontend)

**Style Reference:** Next.js 14 App Router with strict TypeScript. ESLint extends `next/core-web-vitals`.

#### Imports
- Use `@/` path alias for local imports (configured in `tsconfig.json`)
- Client components: `"use client"` directive required at top
- Named exports preferred over default exports for components

```typescript
"use client";

import { ChatInput } from "@/components/chat/ChatInput";
import { useAppStore } from "@/lib/store";
```

#### Type Annotations
- Strict TypeScript enabled (`"strict": true` in tsconfig)
- Use explicit type annotations on function parameters
- Return types inferred for simple functions, explicit for exported APIs
- Use `type` not `interface` for object type definitions (consistent with codebase)

```typescript
type Message = {
  id: string;
  role: "user" | "assistant";
  content: string;
  toolCalls: ToolCall[];
  retrievals: RetrievalResult[];
};

export function ChatPanel() {
  // Return type inferred
}
```

#### Naming Conventions
- Components: `PascalCase` (e.g., `ChatPanel`, `InspectorPanel`)
- Functions/hooks: `camelCase` (e.g., `useAppStore`, `streamChat`)
- Types/interfaces: `PascalCase` (e.g., `SessionSummary`, `ToolCall`)
- Files: `kebab-case` for utilities (e.g., `api.ts`, `store.tsx`), `PascalCase` for components

#### React Patterns
- Use `use client` for all frontend components (no server components yet)
- Store access via `useAppStore()` hook with context
- Prefer `useEffect` with `async` IIFE pattern for initialization
- Use `useRef` for scroll-into-view and similar DOM operations
- SSE event handling via callback pattern in `streamChat`

```typescript
useEffect(() => {
  void (async () => {
    const data = await fetchData();
    setState(data);
  })();
}, []);
```

#### Tailwind CSS
- Use CSS variables for theme colors: `text-[var(--color-ink-soft)]`
- Panel styling: `rounded-[28px]` or `rounded-[32px]`
- Shadow: `shadow-panel` (custom extended in tailwind.config.ts)
- Use `flex` and `gap` for layout, avoid absolute positioning

---

## Error Handling Patterns

### Python Backend
```python
# API error responses
raise HTTPException(status_code=503, detail="Agent manager is not initialized")

# Graceful degradation
try:
    result = risky_operation()
except Exception:
    result = fallback_value  # broad catch allowed when cleanup isn't needed

# Type-safe optional chaining
value = os.getenv("KEY") or default_value
```

### TypeScript Frontend
```typescript
// API error handling in store
try {
  await apiCall();
} catch (error) {
  setErrorState(true);
  throw error; // re-throw for caller
}

// SSE error handling
if (event === "error") {
  patchAssistant((message) => ({
    ...message,
    content: message.content || `请求失败: ${String(data.error ?? "unknown error")}`
  }));
}
```

---

## File Structure Conventions

### Backend API Routes
```
backend/api/
├── __init__.py       # Empty, marks package
├── chat.py           # POST /api/chat (StreamingResponse)
├── sessions.py       # Session CRUD
├── files.py          # File read/write
├── config_api.py     # Runtime config (RAG mode)
├── tokens.py         # Token counting
└── compress.py       # Session compression
```

### Frontend Components
```
frontend/src/components/
├── chat/
│   ├── ChatPanel.tsx
│   ├── ChatMessage.tsx
│   ├── ChatInput.tsx
│   ├── ThoughtChain.tsx
│   └── RetrievalCard.tsx
├── editor/
│   └── InspectorPanel.tsx
└── layout/
    ├── Navbar.tsx
    ├── Sidebar.tsx
    └── ResizeHandle.tsx
```

---

## Important Notes

### Environment Variables
- Never commit `.env` files (already in `.gitignore`)
- API keys are resolved via cascading `_first_env()` helper in `config.py`
- Provider aliases allow flexibility: `glm`, `zhipuai`, `bigmodel` all map to `zhipu`

### Data Persistence
- Sessions saved as JSON in `backend/sessions/*.json`
- Memory stored in `backend/memory/MEMORY.md`
- Skills as `SKILL.md` files in `backend/skills/*/`
- All data directories tracked with `.gitkeep` files (empty directories otherwise invisible to git)

### Supported LLM Providers
- Zhipu (`glm-*` models)
- Bailian/Alibaba (`qwen-*` models)
- DeepSeek (`deepseek-chat`)
- OpenAI-compatible

---

## Revision Notes for Agents

When modifying this codebase:
1. **Python files**: Always add `from __future__ import annotations` at top
2. **TypeScript**: Components are client components (`"use client"`)
3. **Testing**: No formal test framework; verify manually or add pytest/playwright tests
4. **Linting**: Frontend uses `npm run lint` (Next.js ESLint); Python has no enforced linter
5. **Type safety**: Both Python and TypeScript are strictly typed; avoid `any`/`as any`
