# Big Picture Architecture Plan - MagnaPP Planning Poker

## System Overview

MagnaPP is a real-time collaborative Planning Poker application built with a modern tech stack designed for corporate environments. The system supports up to 3 concurrent sessions with 16 users each, focusing on seamless user experience and reliable real-time synchronization.

## Technology Stack Integration

### Frontend: React 18 + TypeScript
- **Framework**: React 18 with concurrent features
- **State Management**: Redux Toolkit with RTK Query
- **UI Library**: Material-UI (MUI) v5
- **Build Tool**: Vite for fast development and optimized builds
- **Styling**: MUI theming system with CSS-in-JS
- **Testing**: Vitest (unit) + Playwright (E2E)

### Backend: Python FastAPI
- **Framework**: FastAPI with async/await patterns
- **Real-time**: Server-Sent Events (SSE) with EventSource API
- **Data Validation**: Pydantic models
- **Session Storage**: In-memory with TTL management
- **API Documentation**: Automatic OpenAPI generation
- **Testing**: Pytest with async support

### Communication Protocol
- **Real-time Updates**: Server-Sent Events (unidirectional, server → client)
- **User Actions**: HTTP POST requests (client → server)
- **Session Management**: RESTful API endpoints
- **Connection Handling**: EventSource automatic reconnection

## Architecture Integration Points

### 1. Real-time Data Flow

```
User Action (React) 
    ↓ HTTP POST
FastAPI Endpoint 
    ↓ Update In-Memory State
Session State Manager
    ↓ Broadcast Event
SSE Event Stream
    ↓ EventSource
React Redux Store
    ↓ State Update
UI Components Re-render
```

### 2. Session Lifecycle Management

**Frontend Responsibilities:**
- User authentication (localStorage preferences)
- Session discovery and joining interface
- Real-time UI updates via Redux
- Connection status management
- Optimistic UI updates for better UX

**Backend Responsibilities:**
- Session creation and GUID generation
- User capacity enforcement (16 per session, 3 total sessions)
- Session expiry management (10-minute inactivity timeout)
- Vote calculation and statistics
- SSE event broadcasting

### 3. State Synchronization Strategy

**Redux Store (Frontend):**
```typescript
interface RootState {
  user: UserState;           // Local user preferences
  session: SessionState;     // Current session data
  voting: VotingState;       // Real-time voting state
  connection: ConnectionState; // SSE connection status
  ui: UIState;              // UI-specific state
}
```

**In-Memory Store (Backend):**
```python
class SessionManager:
    sessions: Dict[str, Session]  # GUID → Session
    user_sessions: Dict[str, str] # UserID → SessionID
    max_sessions: int = 3
    session_timeout: int = 600    # 10 minutes
```

## Component Integration Architecture

### Frontend Component Hierarchy
```
App (Redux Provider + MUI Theme)
├── Router (React Router v6)
│   ├── LandingPage
│   │   ├── UserSetupForm (localStorage)
│   │   ├── SessionCreator (API calls)
│   │   └── ActiveSessionsList (SSE updates)
│   └── SessionBoardroom
│       ├── OvalTable (SVG visualization)
│       ├── VotingPanel (optimistic updates)
│       └── ScrumMasterControls (role-based UI)
└── Global Components
    ├── ConnectionStatus (SSE monitoring)
    └── ToastNotifications (user feedback)
```

### Backend Service Architecture
```
backend/
├── src/
│   ├── main.py                    # FastAPI application entry point
│   ├── models/                    # Pydantic data models
│   │   ├── session.py             # Session and user models
│   │   ├── voting.py              # Voting and statistics models
│   │   └── events.py              # SSE event models
│   ├── api/                       # API route handlers
│   │   ├── sessions.py            # Session CRUD endpoints
│   │   ├── voting.py              # Voting flow endpoints
│   │   ├── users.py               # User management endpoints
│   │   └── sse.py                 # Server-Sent Events endpoint
│   ├── services/                  # Business logic layer
│   │   ├── session_manager.py     # Core session management
│   │   ├── voting_manager.py      # Voting logic and statistics
│   │   ├── sse_manager.py         # Real-time event broadcasting
│   │   └── cleanup_service.py     # Background cleanup tasks
│   └── core/                      # Configuration and utilities
│       ├── config.py              # Environment configuration
│       ├── exceptions.py          # Custom exception hierarchy
│       └── constants.py           # Application constants
```

### Backend API Structure
```
/api/
├── sessions/
│   ├── POST /                     # Create session
│   ├── GET /                      # List active sessions
│   ├── GET /{id}                  # Get session details
│   ├── POST /{id}/join            # Join session
│   ├── POST /{id}/leave           # Leave session
│   └── DELETE /{id}               # End session (Scrum Master)
├── voting/
│   ├── POST /{id}/start           # Start voting round (Scrum Master)
│   ├── POST /{id}/vote            # Submit vote
│   ├── POST /{id}/reveal          # Reveal votes (Scrum Master)
│   └── GET /{id}/round            # Get current round status
├── users/
│   ├── POST /{session_id}/kick/{user_id}    # Kick user (Scrum Master)
│   ├── POST /{session_id}/transfer          # Transfer Scrum Master role
│   └── POST /{session_id}/reconnect         # Reconnect after disconnection
└── sse/
    └── GET /{session_id}          # EventSource streaming endpoint
```

## Real-time Communication Flow

### 1. Connection Establishment
```typescript
// Frontend: EventSource connection
const eventSource = new EventSource(`/api/sse/${sessionId}`);
eventSource.onmessage = (event) => {
  const data = JSON.parse(event.data);
  dispatch(handleSSEEvent(data));
};
```

```python
# Backend: SSE endpoint
@app.get("/api/sse/{session_id}")
async def stream_session_events(session_id: str):
    async def event_generator():
        while True:
            if session_has_updates(session_id):
                yield f"data: {json.dumps(get_session_update(session_id))}\n\n"
            await asyncio.sleep(0.1)
    
    return StreamingResponse(event_generator(), media_type="text/plain")
```

### 2. Event Types and Handling
```typescript
// Frontend: Event type definitions
type SSEEvent = 
  | { type: 'user_joined'; data: { user: User } }
  | { type: 'user_left'; data: { userId: string } }
  | { type: 'voting_started'; data: { roundId: string } }
  | { type: 'vote_submitted'; data: { userId: string } }
  | { type: 'votes_revealed'; data: { votes: Vote[]; statistics: Stats } }
  | { type: 'session_expired'; data: {} }
  | { type: 'scrum_master_changed'; data: { newMasterId: string } };
```

### 3. Optimistic Updates Pattern
```typescript
// Frontend: Optimistic voting
const submitVote = async (vote: string) => {
  // Immediate UI update (optimistic)
  dispatch(setMyVote(vote));
  
  try {
    await api.submitVote({ sessionId, vote });
    // Server confirms - no additional action needed
  } catch (error) {
    // Server rejects - revert optimistic update
    dispatch(revertMyVote());
    showError('Vote submission failed');
  }
};
```

## Data Models and Types

### Shared TypeScript/Python Types
```typescript
// User Model
interface User {
  id: string;
  name: string;
  avatar: string;
  isOnline: boolean;
  joinedAt: string;
}

// Session Model  
interface Session {
  id: string;
  name: string;
  participants: User[];
  scrumMasterId: string;
  status: 'waiting' | 'voting' | 'revealing' | 'results';
  currentRound: VotingRound | null;
  expiresAt: string;
  createdAt: string;
}

// Voting Model
interface VotingRound {
  id: string;
  votes: Record<string, string>; // userId → vote
  isRevealed: boolean;
  statistics: {
    average: number;
    distribution: Record<string, number>;
    consensus: boolean;
    coffeeVotes: number;
  } | null;
}
```

## Development Workflow Integration

### 1. Development Environment Setup
```bash
# Start both services in development
npm run dev:frontend  # Vite dev server on :5173
python -m uvicorn backend.main:app --reload --port 8000

# Vite proxies API calls to :8000
# SSE connections go directly to backend
```

### 2. Build and Deployment Pipeline
```yaml
# Frontend Build (Vite)
build:
  - npm run build
  - outputs: dist/ (static files)
  - deployment: nginx or CDN

# Backend Build (Python)
build:
  - pip install -r requirements.txt
  - deployment: uvicorn with gunicorn
```

### 3. Testing Strategy Integration
```bash
# Unit Tests
npm run test              # Vitest (frontend components/hooks)
pytest tests/             # Python (backend logic/API)

# Integration Tests  
npm run test:integration  # Frontend + mocked SSE
pytest tests/integration/ # Backend + real sessions

# E2E Tests
npm run test:e2e          # Playwright (full user flows)
```

## Deployment Architecture

### Production Environment
```
[Users] → [Load Balancer] → [Nginx] → [Static Frontend Files]
                                  ↓
                              [FastAPI Backend] → [In-Memory Sessions]
                                  ↑
                              [SSE Connections]
```

### Development Environment
```
[Developer] → [Vite Dev Server :5173] → [Proxy] → [FastAPI :8000]
                                                     ↓
                                              [In-Memory Sessions]
```

## Performance Considerations

### Frontend Optimizations
- **React.memo** for user avatar components (prevent unnecessary re-renders)
- **Lazy loading** for non-critical components (statistics, admin panels)
- **Redux selector optimization** to minimize subscription updates
- **EventSource reconnection** with exponential backoff
- **Bundle splitting** by feature (voting, session management, etc.)

### Backend Optimizations
- **Async FastAPI** for concurrent request handling
- **In-memory session storage** for fast state access
- **Background tasks** for session cleanup
- **SSE connection pooling** to handle multiple simultaneous streams
- **Event batching** to reduce SSE message frequency

### Network Optimizations
- **SSE over HTTP/2** for multiplexed connections
- **Gzip compression** for API responses
- **Static asset caching** with proper cache headers
- **CDN deployment** for global availability

## Security Considerations

### Frontend Security
- **Input validation** on all user-provided data
- **XSS protection** through proper data sanitization
- **localStorage security** for user preferences
- **HTTPS enforcement** for all communications
- **Content Security Policy** headers

### Backend Security
- **Request validation** with Pydantic models
- **Rate limiting** on API endpoints
- **Session isolation** between different sessions
- **Memory cleanup** to prevent information leakage
- **CORS configuration** for allowed origins

## Monitoring and Observability

### Application Metrics
- **Session Creation Rate**: Sessions created per hour
- **User Engagement**: Average session duration, voting participation
- **Connection Health**: SSE connection success rate, reconnection frequency
- **Performance**: API response times, frontend rendering performance
- **Error Rates**: Failed votes, connection drops, session timeouts

### Technical Monitoring
- **Memory Usage**: Backend session storage consumption
- **Connection Count**: Concurrent SSE connections per session
- **API Latency**: Request/response times for voting actions
- **Frontend Performance**: Core Web Vitals, bundle size impact
- **Browser Compatibility**: Error rates by browser type

## Risk Mitigation Strategies

### Real-time Communication Risks
- **SSE Connection Drops**: Automatic reconnection with exponential backoff
- **State Desynchronization**: Server-side state checkpoints and reconciliation
- **High Latency Networks**: Optimistic updates with rollback capability
- **Firewall Issues**: SSE over standard HTTP (port 80/443)

### Scalability Risks
- **Memory Leaks**: Automatic session cleanup and memory monitoring
- **Concurrent User Limits**: Graceful degradation when limits exceeded
- **Performance Degradation**: Performance budgets and monitoring alerts
- **Browser Compatibility**: Comprehensive cross-browser testing

### User Experience Risks
- **Poor Mobile Experience**: Mobile-first responsive design
- **Accessibility Issues**: WCAG 2.1 AA compliance testing
- **Complex UI**: User testing and iterative improvements
- **Network Interruptions**: Clear connection status and recovery UX

## Success Metrics and KPIs

### User Experience Metrics
- **Session Completion Rate**: >95% of started sessions complete voting
- **User Retention**: >90% of users stay for full session duration
- **Reconnection Success**: >98% of dropped connections recover automatically
- **Cross-device Functionality**: Consistent experience across desktop/mobile

### Technical Performance Metrics
- **Real-time Latency**: <1 second for all state synchronization
- **Page Load Time**: <2 seconds for initial application load
- **API Response Time**: <200ms for voting actions
- **Connection Success Rate**: >99% SSE connection establishment

### Business Impact Metrics
- **Adoption Rate**: Number of teams using MagnaPP regularly
- **Session Frequency**: Average sessions per team per week
- **Estimation Efficiency**: Reduced time per estimation compared to alternatives
- **User Satisfaction**: User feedback scores and feature requests

This big picture plan provides a comprehensive view of how all system components work together to deliver the MagnaPP Planning Poker application. The architecture prioritizes real-time collaboration, user experience, and maintainable code while meeting all technical and business requirements.