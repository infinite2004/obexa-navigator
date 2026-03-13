# Phase 4: Frontend - Quick Reference Guide

**Framework**: Next.js 15 + React 19 + TypeScript + Tailwind CSS  
**Status**: ✅ Complete  
**Date**: March 12, 2026

---

## Quick Start

### Install Dependencies
```bash
cd frontend
npm install
```

### Development Server
```bash
npm run dev
# Visit http://localhost:3000
```

### Production Build
```bash
npm run build
npm start
```

### Type Check
```bash
npm run typecheck
# or
tsc --noEmit
```

### Lint
```bash
npm run lint
```

---

## Environment Setup

### .env.local (Create this file)

```bash
NEXT_PUBLIC_API_BASE=http://localhost:8000
NEXT_PUBLIC_WS_BASE=ws://localhost:8000
```

### Backend Must Be Running

```bash
# In another terminal
cd backend
python -m backend.orchestrator.main
# or
python -m backend.executor.main
```

---

## File Structure Quick Navigation

### Components

```
src/components/
├── SessionStart.tsx          # Landing page → startSession()
├── StatusBar.tsx             # Top bar → shows session + WS status
├── TranscriptPanel.tsx       # Chat → sendMessage()
├── ScreenCapture.tsx         # Input → uploadScreenshot()
├── ActionPlanPanel.tsx       # Plan → shows steps + progress
├── VerificationStatus.tsx    # Results → pass/fail
├── RecoveryState.tsx         # Recovery → strategy + status
├── InterruptionControls.tsx  # Actions → interrupt()
├── EvaluationMetrics.tsx     # Stats → metrics dashboard
├── DebugTracePanel.tsx       # Events → trace log
└── ConfirmationModal.tsx     # Gate → high-risk confirmation
```

### Hooks

```
src/hooks/
├── useSession.ts             # Central state + orchestrator API calls
└── useWebSocket.ts           # WS connection + message dispatch
```

### Library

```
src/lib/
├── api-client.ts             # REST API client methods
└── api.ts                    # API type definitions
```

### Types

```
src/types/
└── index.ts                  # All TypeScript model definitions
                              # (SessionState, ActionPlan, etc.)
```

### App

```
src/app/
├── layout.tsx                # Root layout + providers
├── page.tsx                  # Main dashboard (11-component layout)
└── globals.css               # Tailwind CSS imports
```

---

## Common Development Tasks

### Add a New Component

1. **Create file** in `src/components/MyComponent.tsx`:
```typescript
import { useSession } from "@/hooks/useSession";

export function MyComponent() {
  const { session, isLoading } = useSession();
  
  return (
    <div className="bg-zinc-900 p-2">
      {/* Component content */}
    </div>
  );
}
```

2. **Export from** `src/app/page.tsx`:
```typescript
import { MyComponent } from "@/components/MyComponent";

export default function Dashboard() {
  return (
    <div>
      <MyComponent />
    </div>
  );
}
```

3. **Use the hook** for state and methods:
```typescript
const {
  session,
  plan,
  transcript,
  isLoading,
  startSession,
  sendMessage,
  uploadScreenshot,
  interrupt,
} = useSession();
```

### Add a New API Endpoint

1. **Add method** to `src/lib/api-client.ts`:
```typescript
export async function newEndpoint(sessionId: string, data: unknown) {
  const response = await fetch(
    `${process.env.NEXT_PUBLIC_API_BASE}/sessions/${sessionId}/new-endpoint`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    }
  );
  
  if (!response.ok) {
    throw new Error(`API error: ${response.statusText}`);
  }
  
  return response.json();
}
```

2. **Add method** to `src/hooks/useSession.ts`:
```typescript
const newMethod = async (data: unknown) => {
  try {
    const result = await newEndpoint(sessionId!, data);
    // Update state
    setState(result);
  } catch (error) {
    setError(String(error));
  }
};
```

3. **Export** from hook and use in components:
```typescript
const { newMethod } = useSession();
newMethod(data);
```

### Add a New Type

1. **Add interface** to `src/types/index.ts`:
```typescript
export interface MyModel {
  id: string;
  status: "active" | "completed";
  data: Record<string, unknown>;
}
```

2. **Use in components**:
```typescript
import type { MyModel } from "@/types";

interface MyComponentProps {
  model: MyModel;
}
```

### Style a Component

Use Tailwind classes:
```typescript
<div className="
  bg-zinc-900          /* dark background */
  border border-zinc-800 /* subtle border */
  p-2                  /* padding */
  rounded              /* rounded corners */
  max-h-96             /* max height */
  overflow-y-auto      /* scroll */
  text-xs              /* small text */
  text-zinc-300        /* readable color */
">
  Content
</div>
```

Color palette:
- **Background**: `bg-zinc-950` (very dark), `bg-zinc-900` (dark), `bg-zinc-800` (panels)
- **Text**: `text-zinc-300` (readable), `text-zinc-600` (muted)
- **Status**: `cyan-500`, `violet-500`, `blue-500`, `emerald-500`, `amber-500`, `red-500`

### Debug State

Add console logging to see hook state:
```typescript
const session = useSession();
console.log("Session state:", session);
console.log("Current plan:", session.plan);
console.log("Transcript:", session.transcript);
console.log("Metrics:", session.metrics);
```

Or check in React DevTools:
1. Open DevTools (F12)
2. Components tab
3. Find component
4. Inspect hook state in sidebar

### Test a Component

```typescript
// Create a mock useSession
jest.mock("@/hooks/useSession", () => ({
  useSession: () => ({
    sessionId: "test-123",
    session: { status: "executing" },
    plan: { goal: "test", steps: [] },
    isLoading: false,
    error: null,
    sendMessage: jest.fn(),
  }),
}));

// Render component
render(<MyComponent />);
expect(screen.getByText("test")).toBeInTheDocument();
```

---

## WebSocket Message Handling

### Server → Client Messages

The `useWebSocket` hook automatically dispatches these to `useSession`:

```typescript
// Message types
type WSServerMessage =
  | { type: "session_updated"; data: SessionState }
  | { type: "plan_created"; data: ActionPlan }
  | { type: "step_executed"; data: ActionResult }
  | { type: "step_verified"; data: VerificationResult }
  | { type: "recovery_triggered"; data: RecoveryEvent }
  | { type: "interruption_received"; data: InterruptionEvent }
  | { type: "trace_event"; data: TraceEvent }
  | { type: "confirmation_required"; step_id: string; reason: string }
  | { type: "session_completed"; result: "success" | "failed" };
```

### Client → Server Messages

Send via API client (which converts to WebSocket):

```typescript
// Send message
await sendMessage("text", frameBase64?);
// → WebSocket: { type: "send_message", message, frame_base64 }

// Execute step
await executeStep(stepId, confirmed?);
// → WebSocket: { type: "execute_step", step_id, confirmed }

// Confirm risky step
await confirmStep(stepId);
// → WebSocket: { type: "confirm_step", step_id }

// Interrupt
await interrupt(newGoal);
// → WebSocket: { type: "interrupt", new_goal }
```

---

## Performance Tips

### 1. Avoid Unnecessary Re-renders

Use `React.memo` for expensive components:
```typescript
export const MyComponent = React.memo(function MyComponent({ data }) {
  return <div>{data.value}</div>;
});
```

### 2. Memoize Calculations

Use `useMemo` in hooks:
```typescript
const metrics = useMemo(() => {
  return calculateMetrics(transcript, plan);
}, [transcript, plan]);
```

### 3. Lazy Load Route Components

```typescript
// In layout.tsx
const DebugPanel = dynamic(() => import("@/components/DebugTracePanel"), {
  loading: () => <div>Loading trace panel...</div>,
});
```

### 4. Optimize Images

Convert screenshots to small thumbnails:
```typescript
const thumbnail = await shrinkImage(base64, 200);
```

---

## Debugging

### TypeScript Errors

```bash
npm run typecheck
# Shows all type errors with line numbers
```

### Runtime Errors

1. **Check browser console** (F12)
2. **Check Network tab** for failed API calls
3. **Check Components tab** (React DevTools) for hook state
4. **Add console.log()** in hooks/components

### WebSocket Connection Issues

```typescript
// Check useWebSocket status in StatusBar
// Should show "connected" in green

// Or log in hook:
console.log("WebSocket status:", wsStatus);
```

### API Client Issues

```typescript
// Add logging to api-client.ts:
console.log("Request:", url, data);
console.log("Response:", await response.json());
```

---

## Common Errors & Fixes

### Error: "Cannot find module '@/types'"

**Fix**: Check path alias in `tsconfig.json`:
```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

### Error: "useSession is not a function"

**Fix**: Import hook at top of component:
```typescript
import { useSession } from "@/hooks/useSession";
```

### Error: WebSocket connection refused

**Fix**: Start backend on port 8000:
```bash
cd backend
python -m backend.orchestrator.main
```

And set env var:
```bash
NEXT_PUBLIC_WS_BASE=ws://localhost:8000
```

### Error: "sessionId is null"

**Fix**: Don't call methods until session is created:
```typescript
if (!sessionId) {
  return <div>Start a session first</div>;
}
```

Or check in hook:
```typescript
if (!useSession().sessionId) {
  return null;
}
```

### Error: Tailwind classes not applying

**Fix**: Check `globals.css` has Tailwind imports:
```css
@tailwind base;
@tailwind components;
@tailwind utilities;
```

And dark mode is enabled in `tailwind.config.ts`:
```typescript
{
  darkMode: "class",
  theme: {
    extend: {},
  },
}
```

---

## Deployment

### Local Testing

```bash
npm run build
npm start
# Visit http://localhost:3000
```

### Docker

```dockerfile
FROM node:18-alpine
WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy built app
COPY .next ./
COPY public ./public

EXPOSE 3000
CMD ["npm", "start"]
```

Build and run:
```bash
docker build -f Dockerfile.frontend -t obexa-navigator-frontend .
docker run -e NEXT_PUBLIC_API_BASE=http://backend:8000 -p 3000:3000 obexa-navigator-frontend
```

### Vercel (Recommended for Next.js)

1. Push to GitHub
2. Connect repo to Vercel
3. Set environment variables in Vercel dashboard
4. Deploy automatically

Environment variables to set:
- `NEXT_PUBLIC_API_BASE` - Backend API URL (e.g., `https://api.example.com`)
- `NEXT_PUBLIC_WS_BASE` - WebSocket URL (optional, defaults to origin)

---

## Testing Checklist

### Manual Testing

- [ ] App loads on http://localhost:3000
- [ ] SessionStart component shows landing page
- [ ] Can enter goal and start session
- [ ] Dashboard appears with all 11 panels
- [ ] StatusBar shows session ID and "connected"
- [ ] Can upload screenshot
- [ ] Can paste screenshot from clipboard
- [ ] Can capture screen (Chrome only)
- [ ] Screenshot preview appears
- [ ] Can send message in transcript
- [ ] Message appears in transcript
- [ ] Plan generates and displays
- [ ] Steps show risk level colors
- [ ] Can execute step
- [ ] Step result appears in transcript
- [ ] Verification status updates
- [ ] Can interrupt with new goal
- [ ] Recovery state displays if error
- [ ] Metrics update
- [ ] Debug panel shows trace events
- [ ] Can expand trace events
- [ ] Can filter trace events
- [ ] High-risk action shows modal
- [ ] Can confirm/cancel in modal

### Automated Testing

```bash
npm test
# Run Jest tests (configure as needed)
```

---

## Code Style Guidelines

### Component Structure

```typescript
import { useSession } from "@/hooks/useSession";
import type { SessionState } from "@/types";

interface MyComponentProps {
  data?: string;
}

export function MyComponent({ data }: MyComponentProps) {
  const { session, isLoading } = useSession();

  if (isLoading) {
    return <div>Loading...</div>;
  }

  if (!session) {
    return <div>No session</div>;
  }

  return (
    <div className="bg-zinc-900 p-2">
      <h3 className="text-xs font-semibold">{session.status}</h3>
      <p className="text-zinc-600">{data}</p>
    </div>
  );
}
```

### Naming Conventions

- **Components**: PascalCase (e.g., `ActionPlanPanel`)
- **Hooks**: camelCase with `use` prefix (e.g., `useSession`)
- **Functions**: camelCase (e.g., `calculateMetrics`)
- **Constants**: UPPER_SNAKE_CASE (e.g., `MAX_RETRIES`)
- **Classes**: PascalCase (e.g., `APIClient`)

### Tailwind Class Order

```typescript
className="
  /* Layout */
  flex flex-col
  /* Sizing */
  w-full h-96
  /* Spacing */
  p-2 gap-2
  /* Appearance */
  bg-zinc-900 border border-zinc-800 rounded
  /* Text */
  text-xs text-zinc-300
  /* State */
  hover:bg-zinc-800
"
```

---

## Useful Commands

```bash
# Install & run
npm install
npm run dev

# Build & start
npm run build
npm start

# Check types
npm run typecheck

# Lint code
npm run lint

# Format (if prettier configured)
npm run format

# Stop dev server
Ctrl+C

# Clear node_modules & reinstall
rm -rf node_modules && npm install

# Update package.json versions
npm update

# Install specific package version
npm install package@1.2.3

# Uninstall package
npm uninstall package
```

---

## Resources

### Documentation
- [Next.js Docs](https://nextjs.org/docs)
- [React Docs](https://react.dev)
- [TypeScript Docs](https://www.typescriptlang.org/docs)
- [Tailwind CSS Docs](https://tailwindcss.com/docs)

### Tools
- [React DevTools](https://react-devtools-tutorial.vercel.app/) - Inspect components
- [Network Tab](https://developer.chrome.com/docs/devtools/network/) - Inspect API calls
- [VS Code Extensions](#vs-code-extensions)

### VS Code Extensions
- **ES7+ React/Redux/React-Native snippets** - Code snippets
- **Tailwind CSS IntelliSense** - Tailwind class completion
- **TypeScript Vue Plugin (Volar)** - TypeScript support
- **Prettier** - Code formatting
- **ESLint** - Linting

---

## Project Statistics

| Metric | Value |
|--------|-------|
| React Components | 11 |
| Custom Hooks | 2 |
| TypeScript Files | 18 |
| Total Lines of Code | ~2,000 |
| Framework | Next.js 15 |
| UI Library | Tailwind CSS |
| Language | TypeScript |
| Node Version | 18+ |

---

## Contacts & Support

For issues or questions:
1. Check the [Specification](./obexa_navigator_spec.md)
2. Review [Phase 4 Implementation Guide](./PHASE4_IMPLEMENTATION.md)
3. Check [Verification Checklist](./PHASE4_VERIFICATION_CHECKLIST.md)
4. Look in code comments for inline documentation

---

**Last Updated**: March 12, 2026  
**Status**: ✅ Phase 4 Frontend Complete
