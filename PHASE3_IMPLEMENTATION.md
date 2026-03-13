# Phase 3: Executor Service Implementation

> **Status**: ✅ **COMPLETE**  
> **Date**: March 2026  
> **Specification Reference**: obexa_navigator_spec.md Section 9.2, Section 14-16

## Executive Summary

Phase 3 implements the **Executor Service**, a Playwright-based deterministic browser automation engine designed for Cloud Run deployment. The service:

- ✅ Accepts **only structured ActionStep** requests (no improvisation)
- ✅ Executes **one action at a time** with full determinism
- ✅ Captures **before/after screenshot evidence** for every step
- ✅ Returns **structured ActionResult** responses with detailed error codes
- ✅ Enforces **risk-aware execution** with mandatory confirmation gates
- ✅ Emits **trace events** for observability and replay
- ✅ Supports **retry hooks** (orchestrator-driven, not self-retrying)
- ✅ Prepared for **private Cloud Run deployment** with service-to-service auth

The executor is a **private service** called only by the orchestrator. It never calls the orchestrator back and never makes business logic decisions.

---

## Architecture Overview

### Core Components

```
backend/executor/
├── main.py                      # Entry point
├── app.py                       # FastAPI app factory
├── config.py                    # Settings (environment-based)
├── browser_pool.py              # Playwright context pool & lifecycle
├── step_executor.py             # Core execution orchestrator
├── action_handlers.py           # 12 handlers (one per ActionType)
├── element_resolver.py          # Multi-strategy element location
├── execution_guard.py           # Pre-execution safety checks
├── screenshot_service.py        # Screenshot capture (before/after)
├── auth_middleware.py           # Cloud Run OIDC token validation
├── retry_hooks.py               # Recovery hooks (called by orchestrator)
├── routes/
│   ├── __init__.py
│   ├── execute_routes.py        # POST /execute-step (main endpoint)
│   ├── health_routes.py         # Liveness & readiness probes
│   └── context_routes.py        # Browser context management + retry hooks
├── test_executor_integration.py # Integration tests
└── deploy_notes.py              # Comprehensive deployment guide
```

### Execution Flow

```
1. Request: ExecuteStepRequest
   ├── ActionStep (structured, validated at source)
   ├── session_id (for browser context isolation)
   └── orchestrator_confirmed (required for HIGH-risk actions)

2. Guard Checks (3 layers)
   ├── Action type is in allowed set (no fabricated actions)
   ├── Risk level matches authoritative ACTION_RISK_MAP
   ├── High-risk actions have explicit orchestrator_confirmed flag
   ├── Target URLs not in blocked list
   └── Step schema well-formed (required fields present)

3. Browser Setup
   ├── Acquire page for session from BrowserPool
   └── Each session isolated in separate context

4. Before Execution
   └── Capture screenshot (for visual verification)

5. Action Execution
   ├── Resolve element (4 strategies: selector, text, ARIA role, bbox)
   ├── Call handler for ActionType (playwright interactions)
   └── Return HandlerResult (success/error_code/target_resolved)

6. After Execution
   └── Capture screenshot (for visual verification)

7. Build Result
   ├── Create ActionResult with status, error_code, latency
   ├── Include both screenshot refs
   └── Always returns ActionResult (never raises)

8. Log & Respond
   └── Return ExecuteStepResponse with ActionResult
```

### Risk Enforcement

Risk assignment is **authoritative** per `backend/policies/action_risk.py`:

```python
# LOW RISK (auto-execute allowed)
CLICK, SCROLL, WAIT, GO_BACK, OPEN_URL, HOVER, EXTRACT

# MEDIUM RISK (visual confirmation or user setting check)
TYPE, SELECT

# HIGH RISK (explicit user confirmation required)
SUBMIT, DELETE, SEND, PUBLISH
```

**Guard Check Logic**:
```python
if action_risk_level >= HIGH and not orchestrator_confirmed:
    return ActionResult(status=SKIPPED, reason="Confirmation required")
```

This prevents the executor from circumventing the planner/orchestrator's trust model.

---

## Key Features

### 1. Deterministic Action Execution

Each handler maps to exactly one Playwright operation with no side-channel decisions:

```python
# CLICK handler: just clicks, never adjusts or retries
async def handle_click(page: Page, step: ActionStep, settings):
    locator, resolved = await resolve_element(page, step.target, settings)
    if locator is None:
        return _not_found(step, resolved)  # Fail immediately
    await locator.click(timeout=settings.default_action_timeout_ms)
    return _success(resolved, f"Clicked {step.target.element_id}")
```

**No Improvisation Rules**:
- If element not found → return ELEMENT_NOT_FOUND, don't search nearby
- If timeout → return timeout error, don't retry
- If guard blocks → return SKIPPED, don't execute anyway
- If handler crashes → return PLAYWRIGHT_ERROR, don't invent a recovery

### 2. Multi-Strategy Element Resolution

The executor tries to locate elements using multiple strategies in priority order:

```python
# Strategy 1: CSS/XPath selector (most reliable)
selector: "button.submit"

# Strategy 2: Text content match (if selector unavailable)
text_match: "Submit"

# Strategy 3: ARIA role + name (accessibility standard)
role="button" name="Submit"

# Strategy 4: Bounding box coordinates (last resort)
bbox: [100, 200, 50, 30]
```

Each strategy supports **retries** (configurable delays) to handle timing-sensitive pages.

### 3. Execution Guard

Pre-execution validation prevents unsafe actions from reaching Playwright:

```python
check_execution_guard(step, settings, orchestrator_confirmed):
  1. Is action_type in known set? (no CLICK_FASTER or typos)
  2. Is risk_level >= authoritative level? (planner cannot under-risk)
  3. For HIGH-risk, is orchestrator_confirmed=True? (no bypass)
  4. Is target URL not blocked? (no payment/checkout sites)
  5. Is step schema valid? (required fields present)
```

If any check fails → `ActionResult(status=SKIPPED)` (not FAILED). The orchestrator handles SKIPPED as a policy decision, not an execution error.

### 4. Screenshot Evidence Audit Trail

Every step produces **before and after screenshots** for verification and replay:

```python
# Before screenshot
before_ref = "local://session_123/before_click_btn_submit-a1b2c3d4.png"

# Execute action

# After screenshot
after_ref = "local://session_123/after_click_btn_submit-a1b2c3d4.png"

# Result includes both
ActionResult(
    before_screenshot_ref=before_ref,
    after_screenshot_ref=after_ref,
    ...
)
```

This enables:
- **Visual verification**: Did the screen actually change?
- **Replay**: Re-run saved traces with stored artifacts
- **Debugging**: Screenshots show what the executor saw
- **Audit**: Complete visual record of every action

### 5. Error Codes Drive Recovery

The executor categorizes failures into named error codes (not generic "error"):

```python
# Each error code implies specific recovery strategies
ErrorCode.ELEMENT_NOT_FOUND
  → Strategies: FUZZY_ELEMENT_MATCH, SCROLL_TO_ELEMENT, RESCAN_VISIBLE_UI

ErrorCode.PAGE_NOT_LOADED
  → Strategies: WAIT_AND_RETRY, REFRESH_PAGE, CHECK_LOADING_INDICATORS

ErrorCode.NAVIGATION_TIMEOUT
  → Strategies: WAIT_AND_RETRY, REFRESH_PAGE

ErrorCode.ELEMENT_NOT_INTERACTABLE
  → Strategies: SCROLL_TO_ELEMENT, DISMISS_MODAL, WAIT_AND_RETRY
```

The **orchestrator's recovery manager** consults a `recovery_policy` table to decide the next action. The executor has no recovery logic.

### 6. Retry Hooks (Not Auto-Retry)

The executor exposes recovery hooks that the orchestrator can call:

```python
# Retry hook endpoints
POST  /context/refresh              → Refresh current page
POST  /context/go-back              → Navigate back
POST  /context/dismiss-modal        → Try to dismiss blocking modal
POST  /context/scroll-to-element    → Scroll to make element visible
POST  /context/capture-screenshot   → Get fresh screenshot for replanning
```

These are **not called automatically**. The orchestrator decides if/when to use them.

### 7. Cloud Run Service-to-Service Authentication

Production deployments use **OIDC identity tokens** for Cloud Run services:

```python
# 1. Orchestrator calls executor with OIDC token (auto by Cloud Run)
GET /execute-step
  Authorization: Bearer <google.oidc.token>

# 2. Executor validates token
from google.oauth2 import id_token
payload = id_token.verify_oauth2_token(
    token,
    request=Request(),
    audience="https://us-central1-PROJECT.run.app/obexa-executor"
)

# 3. Token includes: sub, email, aud, exp (not user-provided)
```

**Local development**: Set `EXECUTOR_REQUIRE_AUTH=false` to skip.

---

## Implemented Handlers

All 12 `ActionType` values are fully implemented:

| Action | Risk | Purpose | Notes |
|--------|------|---------|-------|
| CLICK | LOW | Click element | Waits for actionability |
| TYPE | MEDIUM | Type text | Clears field first, timeout per char |
| SCROLL | LOW | Scroll page | Up/down/left/right, pixel amount |
| WAIT | LOW | Wait duration | 1-30 seconds, capped |
| GO_BACK | LOW | Navigate back | Browser history |
| OPEN_URL | LOW | Navigate to URL | Blocked URLs rejected |
| SELECT | MEDIUM | Dropdown option | By label/value |
| HOVER | LOW | Hover element | Mouseover (no click) |
| EXTRACT | LOW | Read text | Returns inner_text |
| SUBMIT | HIGH | Submit form | Click + wait for navigation |
| DELETE | HIGH | Delete action | Usually click delete button |
| SEND | HIGH | Send message | Click send button |
| PUBLISH | HIGH | Publish content | Click publish button |

---

## Configuration

All settings are environment-based (12-factor app):

```bash
# Service
EXECUTOR_HOST=0.0.0.0
EXECUTOR_PORT=8010
EXECUTOR_LOG_LEVEL=INFO

# Browser
EXECUTOR_BROWSER_TYPE=chromium          # or firefox, webkit
EXECUTOR_HEADLESS=true
EXECUTOR_BROWSER_POOL_SIZE=2
EXECUTOR_DEFAULT_VIEWPORT_WIDTH=1280
EXECUTOR_DEFAULT_VIEWPORT_HEIGHT=720
EXECUTOR_DEFAULT_NAVIGATION_TIMEOUT_MS=30000
EXECUTOR_DEFAULT_ACTION_TIMEOUT_MS=10000

# Screenshots
EXECUTOR_SCREENSHOT_FORMAT=png           # or jpeg
EXECUTOR_SCREENSHOT_QUALITY=null         # JPEG only
EXECUTOR_SCREENSHOT_FULL_PAGE=false

# Auth (Cloud Run)
EXECUTOR_REQUIRE_AUTH=true
EXECUTOR_EXPECTED_AUDIENCE=https://us-central1-PROJECT.run.app/obexa-executor

# Element retry
EXECUTOR_MAX_ELEMENT_RETRIES=2
EXECUTOR_RETRY_DELAY_MS=500

# Safety
EXECUTOR_BLOCKED_URL_PATTERNS=*/payment*,*/checkout*,*/purchase*
```

**Note**: `EXECUTOR_` prefix (set in `ExecutorSettings.model_config`).

---

## API Endpoints

### Execute Step (Main Endpoint)

```http
POST /execute-step
Content-Type: application/json
Authorization: Bearer <oidc_token>  [production only]

{
  "step": {
    "step_id": "step_1",
    "action": "CLICK",
    "target": {
      "element_id": "btn_submit",
      "selector": "button.submit",
      "text_match": "Submit",
      "bbox": null
    },
    "risk_level": "low",
    "reason": "Click the submit button to proceed",
    "success_condition": "Form submitted and confirmation page visible"
  },
  "session_id": "session_abc123",
  "orchestrator_confirmed": false
}
```

**Response**: `ActionResult`
```json
{
  "step_id": "step_1",
  "status": "success",
  "error_code": null,
  "error_message": null,
  "before_screenshot_ref": "local://session_abc123/before_click_btn_submit-a1b2.png",
  "after_screenshot_ref": "local://session_abc123/after_click_btn_submit-a1b2.png",
  "executed_at": "2026-03-12T15:30:45.123Z",
  "executor_latency_ms": 1234.5,
  "action_performed": "Clicked btn_submit",
  "target_resolved": "selector:button.submit"
}
```

### Health Checks

```http
GET /health
→ {"status": "ok", "service": "executor"}

GET /health/ready
→ {
    "status": "ready",
    "service": "executor",
    "browser": "running",
    "active_contexts": 2
  }
```

### Context Management

```http
POST /context/create
{"session_id": "session_abc", "start_url": "https://example.com"}
→ {"session_id": "...", "page_url": "...", "page_title": "..."}

GET /context/{session_id}/status
→ {"session_id": "...", "active": true, "page_url": "...", "page_title": "..."}

POST /context/destroy
{"session_id": "session_abc"}
→ {"status": "destroyed", "session_id": "..."}
```

### Retry Hooks

```http
POST /context/refresh
{"session_id": "session_abc"}
→ {"status": "refreshed", "url": "...", "title": "..."}

POST /context/go-back
{"session_id": "session_abc"}
→ {"status": "navigated_back", "url": "...", "title": "..."}

POST /context/dismiss-modal
{"session_id": "session_abc"}
→ {"status": "dismissed", "method": "[aria-label='Close']"}

POST /context/scroll-to-element
{"session_id": "session_abc", "selector": "button.submit"}
→ {"status": "scrolled", "selector": "..."}

POST /context/capture-screenshot
{"session_id": "session_abc"}
→ {"status": "captured", "screenshot_ref": "local://..."}
```

---

## Error Handling

### Execution Status Values

```python
ExecutionStatus.SUCCESS      # Action was physically performed
ExecutionStatus.FAILED       # Action could not be performed (error_code set)
ExecutionStatus.SKIPPED      # Guard check prevented execution
```

### Decision Tree

```
Did guard check pass?
  NO  → status=SKIPPED, error_code=guard_error, don't execute
  YES → Try to execute

Did handler succeed?
  YES → status=SUCCESS, no error_code
  NO  → status=FAILED, error_code set, error_message set

Example flow:
  Request → Guard: HIGH-risk, no confirmation
    → Return ActionResult(status=SKIPPED, reason="Confirmation required")

  Request → Guard: OK
    → Execute → Element not found
    → Return ActionResult(status=FAILED, error_code=ELEMENT_NOT_FOUND)

  Request → Guard: OK
    → Execute → Click succeeded
    → Return ActionResult(status=SUCCESS)
```

---

## Testing

### Test Coverage

- ✅ **Execution guard**: Valid/invalid actions, risk levels, confirmation
- ✅ **Action handlers**: Each of 12 handlers tested with mocked page
- ✅ **Screenshot capture**: Before/after captured for all paths
- ✅ **Error handling**: Failures produce correct error codes
- ✅ **Risk level mapping**: LOW/MEDIUM/HIGH correctly assigned

### Run Tests

```bash
# All tests
pytest backend/executor/test_executor_integration.py -v

# Specific test class
pytest backend/executor/test_executor_integration.py::TestExecutionGuard -v

# With real browser (requires Playwright)
pytest backend/executor/test_executor_integration.py -v --playwright
```

### Manual Testing

```bash
# Local dev
export EXECUTOR_REQUIRE_AUTH=false
python -m backend.executor.main

# Send request (Terminal 2)
curl -X POST http://localhost:8010/execute-step \
  -H "Content-Type: application/json" \
  -d '{
    "step": {
      "step_id": "step_1",
      "action": "CLICK",
      "target": {"element_id": "btn", "selector": "button"},
      "risk_level": "low",
      "reason": "Click button",
      "success_condition": "Button clicked"
    },
    "session_id": "test_session",
    "orchestrator_confirmed": false
  }'
```

---

## Deployment

### Local Docker

```bash
docker build -f Dockerfile.executor -t obexa-executor .
docker run -p 8010:8010 -e EXECUTOR_REQUIRE_AUTH=false obexa-executor
curl http://localhost:8010/health
```

### Cloud Run

```bash
gcloud run deploy obexa-executor \
  --source . \
  --dockerfile Dockerfile.executor \
  --platform managed \
  --region us-central1 \
  --port 8010 \
  --no-allow-unauthenticated \
  --ingress internal \
  --memory 2Gi \
  --cpu 2 \
  --min-instances 1 \
  --max-instances 10 \
  --update-env-vars EXECUTOR_REQUIRE_AUTH=true

# Grant orchestrator permission
gcloud run services add-iam-policy-binding obexa-executor \
  --region us-central1 \
  --member serviceAccount:orchestrator@PROJECT.iam.gserviceaccount.com \
  --role roles/run.invoker
```

### Monitoring

- **Logs**: `gcloud run logs read obexa-executor`
- **Health checks**: `/health` (liveness), `/health/ready` (readiness)
- **Metrics**: Cloud Monitoring dashboard (executor_latency_ms, execution_failures)
- **Alerts**: High error rates (>10%), slow executions (p95 > 5s)

---

## Key Design Decisions

### 1. No Self-Retry

The executor **never retries** on its own. Each step is attempted exactly once:

**Why**: Retries are policy decisions that require orchestrator context (user confirmation, circuit breakers, plan changes). The executor must remain stateless and deterministic.

**Implementation**: GuardResult + HandlerResult → ActionResult (one shot).

### 2. Screenshot for Every Action

Even failed actions capture after-screenshots:

**Why**: Failure context matters. Screenshots show whether the page partially changed, a modal appeared, etc. This is critical for the verifier.

**Cost**: ~200KB per screenshot (PNG, viewport size). Stored locally in `/tmp/obexa-screenshots` with reference strings.

### 3. Error Codes, Not Exceptions

The executor never raises to the caller. All outcomes are ActionResult:

**Why**: The caller (orchestrator) is async and may wait for multiple concurrent actions. Exceptions complicate coordination.

**Implementation**: Catch all exceptions at handler level, return HandlerResult. Catch at step_executor level, return ActionResult.

### 4. Risk Gates Before Execution

High-risk actions are gated at guard-check time, not handler time:

**Why**: Fail fast. Better to reject a dangerous step upfront than after the browser page is already mutated.

**Implementation**: ExecutionGuard runs before any page interaction. If it rejects → ActionResult(status=SKIPPED).

### 5. Stateless Handlers

Action handlers are pure functions, not class methods:

```python
# Pure function (no state, no side effects except Playwright)
async def handle_click(page: Page, step: ActionStep, settings: ExecutorSettings) -> HandlerResult

# NOT a method
class ClickHandler:
    def execute(self, ...): ...
```

**Why**: Simpler testing, easier to reason about, no handler-level state to manage.

### 6. Session Isolation

Each session gets its own browser context with separate cookies/storage:

```python
# Different sessions never share context
page1 = await pool.get_or_create_page("session_123")  # Gets context A
page2 = await pool.get_or_create_page("session_456")  # Gets context B (different)

# Same session reuses context
page3 = await pool.get_or_create_page("session_123")  # Reuses context A
```

**Why**: Prevent cross-contamination. User A's session should not see User B's cookies.

---

## Conformance to Specification

This implementation satisfies all executor requirements from spec Section 9.2:

| Requirement | Implementation | Verified |
|-------------|-----------------|----------|
| Receive structured steps only | ActionStep pydantic model | ✅ |
| Execute action deterministically | One-shot handlers, no retry logic | ✅ |
| Capture before/after screenshots | StepExecutor.execute() calls capture_screenshot twice | ✅ |
| Emit deterministic results | ActionResult always returned | ✅ |
| Never invent new steps | action_handlers has fixed set of handlers | ✅ |
| Risk checks | ExecutionGuard pre-execution validation | ✅ |
| Confirmation gates | orchestrator_confirmed flag required for HIGH-risk | ✅ |
| Retry hooks (not auto-retry) | retry_hooks.py + context_routes.py endpoints | ✅ |
| Private Cloud Run deployment | Dockerfile.executor, OIDC auth, internal ingress | ✅ |
| Service-to-service auth | CloudRunAuthMiddleware validates OIDC tokens | ✅ |

---

## What Comes Next (Phase 4+)

The executor service is **complete and deployable**. The orchestrator (Phase 2) will call it. Phases 4+ add:

- **Phase 4**: Deploy to Cloud Run, final integration
- **Frontend**: Voice capture, debug panel UI, confirmation prompts
- **Verifier** (Phase 3b): Post-execution state verification
- **Recovery Manager** (Phase 3b): Invoke retry hooks on failure

The executor service itself **requires no changes** for these phases.

---

## References

- **Specification**: obexa_navigator_spec.md (Sections 9.2, 14-16)
- **Models**: `backend/models/action_*.py`
- **Risk Policy**: `backend/policies/action_risk.py`
- **Deployment**: `backend/executor/deploy_notes.py`
- **Tests**: `backend/executor/test_executor_integration.py`
- **Docker**: `Dockerfile.executor`
- **Configuration**: `backend/executor/config.py`

---

**Implementation Date**: March 12, 2026  
**Status**: ✅ Complete and ready for integration with Phase 2 (Orchestrator) and Phase 3b (Verifier/Recovery).
