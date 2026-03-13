# Obexa Navigator — Phase 1 Implementation Plan

**Date:** March 12, 2026  
**Status:** Ready for implementation

---

## REVISION SUMMARY (8 Adjustments Applied)

1. ✅ **Frontend deployment decoupled** — Can use Vercel, Netlify, or standalone. Only orchestrator + executor are core judged services on Cloud Run.

2. ✅ **Clear architecture choice** — FastAPI as main app layer, ADK integrated inside orchestrator (not ambiguous). Executor never calls Gemini (deterministic only).

3. ✅ **Schema versioning added** — All JSON schemas and Pydantic models include `schema_version: string` field for safe evolution.

4. ✅ **Interrupt events schema created** — Separate `interrupt_event.schema.json` (v1) with `reason`, `invalidated_step_ids`, `new_constraints`, `replanned_from_ui_state_id`.

5. ✅ **Firestore collections documented** — High-level document shapes for 7 top-level collections (sessions/, events/, plans/, ui_states/, interruptions/, recoveries/, evaluations/).

6. ✅ **Shared contracts layer** — `contracts/` folder with `schemas.py` (Pydantic models), `enums.py` (shared constants), `version.py` (versioning utility). Frontend and backend both import from here.

7. ✅ **Phase 1 includes sample JSON + tests** — Deliverables now include sample valid JSON examples for each schema and comprehensive schema validation tests (`test_schemas.py`).

8. ✅ **Executor constraint explicit** — Documented: Executor never calls Gemini, only executes structured ActionSteps deterministically with Playwright.

---

## PHASE 1: FOUNDATION LAYER (Week 1)

### Goal
Establish the shared contracts and schema foundation that both frontend and backend depend on. Zero business logic yet—just wiring, types, and validation.

### Deliverables

#### A. Contracts Layer (contracts/)
**New folder containing shared source of truth:**

- `contracts/__init__.py`
- `contracts/version.py` — Schema versioning utilities
- `contracts/enums.py` — Shared enums (ActionType, RiskLevel, ErrorCode, etc.)
- `contracts/schemas.py` — **7 Pydantic models** (v1):
  - `UIState` — visible elements, page type, entities
  - `ActionPlan` — ordered steps with risk/reason/success_condition
  - `ActionResult` — execution result with screenshots
  - `VerificationResult` — pass/fail with failure class
  - `InterruptEvent` — interruption with new constraints
  - `RecoveryEvent` — recovery attempt with strategy
  - `SessionState` — session metadata and flags

#### B. JSON Schemas (backend/schemas/)
**Seven reference schemas with sample data:**

- `ui_state.schema.json` (v1) + valid example
- `action_plan.schema.json` (v1) + valid example
- `action_result.schema.json` (v1) + valid example
- `verification_result.schema.json` (v1) + valid example
- `interrupt_event.schema.json` (v1) + valid example **[NEW]**
- `recovery_event.schema.json` (v1) + valid example
- `session_state.schema.json` (v1) + valid example

#### C. Python Models (backend/models/)
**Internal trace/audit models:**

- `trace_event.py` — TraceEvent dataclass
- `tool_call.py` — ToolCall dataclass
- `session_snapshot.py` — SessionSnapshot dataclass
- `action_trace.py` — ActionTrace dataclass
- `recovery_trace.py` — RecoveryTrace dataclass

#### D. FastAPI Skeletons
**Minimal services ready to accept routes:**

**backend/orchestrator/**
- `main.py` — Entry point
- `app.py` — FastAPI app instance
- `config.py` — Configuration, environment vars
- `dependencies.py` — FastAPI injection dependencies
- `routes/health_routes.py` — `/health`, `/ready` endpoints

**backend/executor/**
- `main.py` — Entry point
- `app.py` — FastAPI app instance
- `config.py` — Configuration
- `routes/health_routes.py` — `/health`, `/ready` endpoints

#### E. Schema Validation Tests (backend/tests/unit/)
**Comprehensive validation suite:**

- `test_schemas.py`:
  - Validate each JSON schema against sample data
  - Test Pydantic models serialize/deserialize correctly
  - Test required fields enforced
  - Test enum constraints
  - Test schema versioning (version field present)
  - Test invalid data rejected with clear errors
  
- `fixtures/sample_data.py` — Valid JSON examples for all 7 schemas

#### F. Documentation
**Architecture and setup clarity:**

- `PHASE1_README.md` — What was built, why, and next steps
- Docstrings in contracts/ explaining each schema
- `backend/.env.example` — Local dev environment template

### Success Criteria

✅ All 7 JSON schemas validate against sample data  
✅ All 7 Pydantic models match schemas exactly  
✅ Frontend can import types: `from contracts.schemas import UIState, ActionPlan, ...`  
✅ Backend can import types: `from contracts.schemas import UIState, ActionPlan, ...`  
✅ All enums shared and consistent  
✅ Services start without errors  
✅ `/health` and `/ready` endpoints respond  
✅ `test_schemas.py` passes (100% schema validation)  
✅ Schema versioning field present on all types  
✅ No executor/orchestrator business logic yet (skeleton only)  

---

## ARCHITECTURE (FastAPI + ADK Integration)

### Clear Service Design

```
Frontend (HTTP/WS client)
    ↓ REST API + WebSocket
Orchestrator (FastAPI + ADK integration)
    ├─ Routes (HTTP/WS endpoints)
    ├─ ADK Session (session lifecycle + memory)
    ├─ Gemini tools layer (will implement Phase 3+)
    └─ Executor client (will implement Phase 3+)
           ↓ service-to-service auth
Executor (FastAPI, deterministic only)
    ├─ Routes (HTTP endpoints)
    ├─ Playwright browser pool (will implement Phase 4)
    └─ Action handlers (will implement Phase 4)
```

### Why This Stack

- **FastAPI** handles HTTP and async elegantly
- **ADK** provides sessions, memory, structured events—integrated inside orchestrator
- **Gemini** called only by orchestrator tools (never executor)
- **Executor** stays deterministic (Playwright + structured steps, no MLC)
- **Pydantic** models enforce schema contracts at the language level
- **Firestore** for persistence, simple querying, built-in versioning support

### Executor Constraint (CRITICAL)

**Executor never calls Gemini or any LLM:**
- Receives ONE structured `ActionStep`
- Executes it deterministically withPlaywright
- Captures before/after screenshots
- Returns `ActionResult`
- NO reasoning, NO fallbacks, NO LLM

Only orchestrator calls Gemini (via tools). This enforces separation and maintains verifiability.

---

## FIRESTORE DOCUMENT DESIGN (Reference)

Collections created during Phase 3+, but schema defined now:

```
sessions/{session_id}
  ├─ goal: string
  ├─ current_status: string (PLANNING, EXECUTING, VERIFYING, RECOVERING, COMPLETE)
  ├─ current_plan_id: string
  ├─ latest_ui_state_id: string
  ├─ last_completed_step_id: string
  ├─ interruption_active: boolean
  ├─ recovery_active: boolean
  ├─ created_at: timestamp
  ├─ updated_at: timestamp
  └─ schema_version: string (v1)

sessions/{session_id}/events/{event_id}
  ├─ event_type: string
  ├─ timestamp: timestamp
  ├─ service_name: string (orchestrator, executor)
  ├─ status: string
  ├─ latency_ms: number
  ├─ payload: object (varies by type)
  └─ schema_version: string (v1)

sessions/{session_id}/plans/{plan_id}
sessions/{session_id}/ui_states/{ui_state_id}
sessions/{session_id}/interruptions/{interruption_id}
sessions/{session_id}/recoveries/{recovery_id}
sessions/{session_id}/evaluations/{evaluation_id}
```

---

## SHARED ENUMS (contracts/enums.py)

Both frontend and backend import and use the same enums:

```python
class ActionType(str, Enum):
    CLICK = "CLICK"
    TYPE = "TYPE"
    SCROLL = "SCROLL"
    WAIT = "WAIT"
    GO_BACK = "GO_BACK"
    OPEN_URL = "OPEN_URL"
    EXTRACT = "EXTRACT"
    SELECT = "SELECT"
    SUBMIT = "SUBMIT"
    # ... others as needed

class RiskLevel(str, Enum):
    LOW = "LOW"
    MEDIUM = "MEDIUM"
    HIGH = "HIGH"

class ErrorCode(str, Enum):
    ELEMENT_NOT_FOUND = "ELEMENT_NOT_FOUND"
    PAGE_NOT_LOADED = "PAGE_NOT_LOADED"
    STATE_MISMATCH = "STATE_MISMATCH"
    EXECUTOR_TIMEOUT = "EXECUTOR_TIMEOUT"
    VERIFICATION_FAILED = "VERIFICATION_FAILED"

class VerificationStatus(str, Enum):
    PASSED = "PASSED"
    FAILED = "FAILED"

class ExecutionStatus(str, Enum):
    SUCCESS = "SUCCESS"
    FAILED = "FAILED"
    SKIPPED = "SKIPPED"

class RecoveryStrategy(str, Enum):
    FUZZY_MATCH = "FUZZY_MATCH"
    RESCAN_UI = "RESCAN_UI"
    ALTERNATE_ELEMENT = "ALTERNATE_ELEMENT"
    WAIT_AND_RETRY = "WAIT_AND_RETRY"
    REFRESH_PAGE = "REFRESH_PAGE"
    REPLAN = "REPLAN"
```

---

## WHAT PHASE 1 DOES NOT INCLUDE

- ❌ Gemini integration
- ❌ Action planning or execution
- ❌ Browser automation (Playwright)
- ❌ Verification logic
- ❌ Recovery logic
- ❌ Frontend components
- ❌ Firestore actual implementation
- ❌ ADK integration code

These come in Phases 2-11. Phase 1 is **just contracts, models, and skeleton services**.

---

## HOW FRONTEND AND BACKEND SHARE TYPES

**Single source of truth:**

```
contracts/
├── schemas.py       ← Pydantic models (Python)
├── enums.py         ← Shared constants
└── version.py       ← Versioning utilities

backend/
└── __init__.py      (imports from contracts/)

frontend/
├── app/
├── components/
└── lib/contracts.ts (TypeScript equivalents of contracts/)
```

**Frontend setup** (after contracts layer is done):
```bash
# In frontend/lib/index.ts
export type { UIState, ActionPlan, ActionResult, ... } from '../../contracts/schemas'
export { ActionType, RiskLevel, ErrorCode, ... } from '../../contracts/enums'
```

This ensures frontend and backend stay in sync.

---

## PHASE 1 → PHASE 2 DECISION POINT

After Phase 1 is complete and tests pass, verify:
- ✅ All schemas validate  
- ✅ Frontend can import contracts/  
- ✅ Backend can import contracts/  
- ✅ Services start and respond to health checks  

Then: **Get approval to proceed to Phase 2 (Policies)**.

---

## NEXT STEP: GENERATE PHASE 1 CODE

Ready to implement:
1. `contracts/` folder with schemas.py, enums.py, version.py
2. `backend/schemas/` with 7 JSON schemas + sample data
3. `backend/models/` with 5 dataclasses
4. `backend/orchestrator/` and `backend/executor/` skeletons
5. `backend/tests/unit/test_schemas.py` with comprehensive validation
6. `.env.example` template

**Approval:** Generate Phase 1 code now? (yes/no)
