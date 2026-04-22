# Stack Specifications

Este documento describe las decisiones tecnológicas y la estructura del proyecto.

## Stack Tecnológico

| Capa | Tecnología | Razón |
|------|-----------|-------|
| Frontend | Next.js 15 + TypeScript | Routing multi-board nativo (`/[boardId]`), SSR opcional, React sin lock-in |
| Backend | FastAPI + Python 3.12+ | Ecosistema AI maduro, LiteLLM, RAG, compatible con LangGraph futuro |
| LLM | LiteLLM | Abstracción sobre 100+ proveedores. Cambio de modelo = cambio de variable de entorno |
| Styling | Tailwind CSS + shadcn/ui | Sin vendor lock-in — componentes copiados al repo. Dark mode trivial con Tailwind |
| Tests | Playwright (E2E) + pytest (Python unitarios) | E2E cubre el happy path completo; pytest cubre lógica pura de backend |
| Deploy | Docker + docker-compose | Servidor propio, dos contenedores: frontend y backend |

**Principio:** un lenguaje por capa. TypeScript en frontend, Python en backend.

---

## Estructura de Carpetas

```
advisory-board/
├── frontend/                        # Next.js app
│   ├── app/
│   │   ├── [boardId]/page.tsx       # vista principal de cada board
│   │   ├── layout.tsx
│   │   └── page.tsx                 # redirect al primer board disponible
│   ├── components/
│   │   ├── ui/                      # shadcn/ui (componentes propios)
│   │   └── board/
│   │       ├── ConsensusBar.tsx
│   │       ├── ExpertCard.tsx
│   │       ├── ExpertGrid.tsx
│   │       ├── SynthesisPanel.tsx
│   │       └── ExpertChatModal.tsx
│   ├── hooks/
│   ├── lib/api.ts                   # cliente API y helpers de SSE
│   ├── types/
│   └── tests/e2e/                   # tests Playwright
│
├── backend/
│   ├── app/
│   │   ├── main.py
│   │   ├── orchestrator.py
│   │   ├── agent.py                 # BaseAgent + DirectLLMAgent
│   │   ├── retriever.py             # BaseRetriever + TFIDFRetriever
│   │   ├── llm.py                   # wrapper LiteLLM
│   │   ├── config.py                # carga y validación de board.yaml
│   │   └── routers/
│   │       ├── board.py
│   │       └── auth.py
│   └── tests/unit/
│
├── boards/                          # configuración externa (ver domain.md)
│   ├── veterinary/
│   │   ├── board.yaml
│   │   ├── agents/
│   │   └── knowledge/
│   └── [otros boards]/
│
├── specs/
├── docker-compose.yml
├── Dockerfile.frontend
├── Dockerfile.backend
├── README.md
└── CONTRIBUTING.md
```

---

## Docker

```yaml
services:
  frontend:
    build: ./Dockerfile.frontend
    ports: ["3000:3000"]
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8000

  backend:
    build: ./Dockerfile.backend
    ports: ["8000:8000"]
    volumes:
      - ./boards:/app/boards
    environment:
      - APP_PASSWORD
      - ANTHROPIC_API_KEY
      - LLM_MODEL
      - AGENT_BATCH_SIZE
      - AGENT_BATCH_DELAY
```

Los boards se montan como volumen — se pueden editar sin reconstruir la imagen.

---

## Variables de Entorno

```bash
# Backend
APP_PASSWORD=           # contraseña de acceso al demo
ANTHROPIC_API_KEY=      # clave de API de Anthropic
LLM_MODEL=anthropic/claude-haiku-4-5-20251001   # modelo por defecto
AGENT_BATCH_SIZE=3      # agentes por batch
AGENT_BATCH_DELAY=0     # segundos entre batches

# Frontend
NEXT_PUBLIC_API_URL=http://localhost:8000
```
