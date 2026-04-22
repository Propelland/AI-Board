# API Specifications

Este documento detalla los contratos de la API del backend.

## Resumen
- **Base URL**: `/api`
- **Protocolo**: HTTP (REST) + SSE para streaming
- **Formato**: JSON (entrada/salida), `text/event-stream` (streaming)
- **Autenticación**: Bearer token en header `Authorization: Bearer {token}`

---

## 1. Autenticación (`/api/auth`)

### POST `/api/auth/login`
Autentica al usuario con contraseña.
- **Request Body**: `{ "password": string }`
- **Response**: `{ "token": "UUID" }`
- **Error**: `401` si la contraseña es incorrecta.

---

## 2. Boards (`/api/[boardId]`)

### GET `/api/[boardId]/agents`
Devuelve la lista de agentes del board.
- **Response**:
```json
[
  {
    "id": "gp_companion",
    "name": "Dr. Sarah Chen",
    "credentials": "DVM, DACVIM",
    "seat": "Companion Animals",
    "specialty": "Dogs & cats clinical practice",
    "avatar": "/avatars/gp_companion.png"
  }
]
```
- **Error**: `404` si el `boardId` no existe.

---

### POST `/api/[boardId]/query`
Lanza una consulta al board. Devuelve un stream SSE.
- **Request Body**: `{ "question": string, "agent_ids": string[] | null }`
  - `agent_ids`: si es `null`, se usan todos los agentes del board.
- **Response**: `text/event-stream`

**Eventos SSE:**

```
data: { "type": "query_type", "value": "evaluation" }

data: { "type": "agent_chunk", "agent_id": "gp_companion", "text": "...", "done": false }

data: { "type": "agent_done", "agent_id": "gp_companion", "verdict": {
  "recommendation": "ENDORSE_WITH_CONDITIONS",
  "primary_concern": "...",
  "path_forward": "...",
  "blocking_issue": null,
  "condition": "...",
  "strength": "...",
  "gap": "..."
}}

data: { "type": "consensus_update", "score": { "ENDORSE": 2, "ENDORSE_WITH_CONDITIONS": 3, "DEFER": 1 } }

data: { "type": "orchestrator", "action": "reconsider", "agents": ["gp_companion"], "reason": "..." }

data: { "type": "synthesis_chunk", "text": "...", "done": false }

data: { "type": "synthesis_done", "consensus_score": { "ENDORSE": 2, "ENDORSE_WITH_CONDITIONS": 3, "DEFER": 1 } }

data: { "type": "round_done" }
```

---

## 3. Chat con Experto (`/api/[boardId]/expert`)

### POST `/api/[boardId]/expert/[agentId]/chat`
Inicia o continúa un chat 1:1 con un agente. Devuelve un stream SSE.
- **Request Body**:
```json
{
  "message": string,
  "conversation_id": "UUID",
  "board_context": string | null
}
```
- **Response**: `text/event-stream`

**Eventos SSE:**
```
data: { "type": "chunk", "text": "...", "done": false }
data: { "type": "done" }
```

---

### DELETE `/api/[boardId]/expert/[agentId]/conversation/[conversationId]`
Borra el historial de una conversación 1:1.
- **Response**: `{ "status": "ok" }`

---

## 4. Sistema (`/api`)

### GET `/api/health`
Healthcheck. No requiere autenticación.
- **Response**: `{ "status": "ok" }`
