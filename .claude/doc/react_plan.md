# MagnaPP React Implementation Plan - Unified Architecture

## Executive Summary

MagnaPP is a real-time Planning Poker application for distributed agile teams. This document combines two expert architectural approaches to create a comprehensive implementation plan.

### Technology Stack (Consensus)
- **React 18+** with TypeScript and modern hooks
- **Redux Toolkit** with RTK Query for state management (enhanced from Context API)
- **Material-UI v5** for component library and theming
- **Server-Sent Events (SSE)** for real-time updates (more reliable than WebSocket alone)
- **Vite** for build tooling and development server
- **Vitest + React Testing Library** for unit testing
- **Playwright** for E2E testing

### Architecture Principles
- Component composition over complex hierarchies
- Single responsibility principle for components and hooks
- Optimistic UI updates for responsive user experience
- Error boundaries for graceful failure handling
- Performance optimization using React.memo, useMemo, useCallback
- Predictable state management with Redux DevTools support

## Combined Component Architecture

### High-Level Component Hierarchy
```
App
├── Router (React Router v6)
├── Redux Provider (RTK Store)
├── MUI ThemeProvider
├── Global Components
│   ├── ErrorBoundary
│   ├── ConnectionStatus (SSE indicator)
│   └── ToastNotifications
└── Main Routes
    ├── LandingPage (SessionLobby)
    │   ├── UserSetupForm
    │   ├── SessionCreator
    │   ├── JoinSession
    │   └── ActiveSessionsList
    ├── SessionBoardroom (PlanningRoom)
    │   ├── SessionInfo/BoardroomHeader
    │   ├── OvalTable (BoardView)
    │   │   ├── UserAvatar (×16 max)
    │   │   └── VotingIndicators
    │   ├── VotingPanel
    │   │   ├── VotingCards
    │   │   └── VoteStatistics
    │   └── ScrumMasterControls
    └── NotFoundPage
```

## State Management Strategy (Enhanced)

### Redux Toolkit Store Structure
```typescript
interface RootState {
  // Core application state
  user: UserState;
  session: SessionState;
  voting: VotingState;
  connection: ConnectionState;
  ui: UIState;
  
  // RTK Query API state
  api: ApiState;
}
```

### Context + Redux Hybrid Approach
- **Redux**: For global application state and server synchronization
- **Context API**: For component-specific state (e.g., form state, UI preferences)
- **Local State**: For ephemeral UI state (e.g., hover effects, animations)

### Detailed Slice Definitions

#### User Slice
```typescript
interface UserState {
  id: string | null;
  name: string;
  avatar: string;
  role: 'participant' | 'scrum_master' | 'spectator';
  preferences: {
    theme: 'light' | 'dark' | 'system';
    soundEnabled: boolean;
  };
  isSetup: boolean;
}
```

#### Session Slice
```typescript
interface SessionState {
  current: {
    id: string;
    name: string;
    participants: Participant[];
    scrumMasterId: string;
    status: 'waiting' | 'voting' | 'revealing' | 'results';
    votingRounds: VotingRound[];
    expiresAt: string;
    settings: {
      allowSpectators: boolean;
      autoReveal: boolean;
      showTimer: boolean;
    };
  } | null;
  availableSessions: SessionSummary[];
  loading: boolean;
  error: string | null;
}
```

#### Voting Slice
```typescript
interface VotingState {
  currentRound: {
    roundId: string;
    topic: string;
    votes: Map<string, string>; // userId -> vote
    myVote: string | null;
    isRevealed: boolean;
    startedAt: number;
    revealedAt: number | null;
    statistics: VoteStatistics | null;
  } | null;
  roundHistory: VotingRound[];
  isVoting: boolean;
}
```

## Real-time Communication Strategy

### Dual Approach: SSE + WebSocket Fallback
```typescript
class RealtimeService {
  private eventSource: EventSource | null = null;
  private websocket: WebSocket | null = null;
  private connectionType: 'sse' | 'websocket' = 'sse';
  
  async connect(sessionId: string) {
    try {
      // Primary: Server-Sent Events
      await this.connectSSE(sessionId);
    } catch (error) {
      // Fallback: WebSocket
      await this.connectWebSocket(sessionId);
    }
  }
  
  private connectSSE(sessionId: string) {
    this.eventSource = new EventSource(`/api/sse/${sessionId}`);
    this.setupSSEHandlers();
  }
  
  private connectWebSocket(sessionId: string) {
    this.websocket = new WebSocket(`ws://localhost:8000/ws/${sessionId}`);
    this.setupWebSocketHandlers();
  }
}
```

### Event Types and Handling
```typescript
enum EventType {
  // Session events
  SESSION_CREATED = 'session_created',
  SESSION_UPDATED = 'session_updated',
  SESSION_EXPIRED = 'session_expired',
  
  // User events  
  USER_JOINED = 'user_joined',
  USER_LEFT = 'user_left',
  USER_RECONNECTED = 'user_reconnected',
  
  // Voting events
  VOTING_STARTED = 'voting_started',
  VOTE_CAST = 'vote_cast',
  VOTES_REVEALED = 'votes_revealed',
  ROUND_COMPLETED = 'round_completed',
  
  // Control events
  SCRUM_MASTER_CHANGED = 'scrum_master_changed',
  USER_KICKED = 'user_kicked'
}
```

## File and Folder Organization (Complete)

```
frontend/
├── public/
│   ├── icons/                    # Avatar icons and app icons
│   ├── sounds/                   # Notification sounds
│   └── manifest.json             # PWA manifest
├── src/
│   ├── components/
│   │   ├── common/               # Reusable components
│   │   │   ├── Avatar.tsx
│   │   │   ├── Button.tsx
│   │   │   ├── LoadingSpinner.tsx
│   │   │   ├── ErrorBoundary.tsx
│   │   │   ├── ConnectionStatus.tsx
│   │   │   └── Toast.tsx
│   │   ├── boardroom/            # Boardroom-specific
│   │   │   ├── OvalTable.tsx
│   │   │   ├── UserAvatar.tsx
│   │   │   ├── BoardroomHeader.tsx
│   │   │   ├── SessionTimer.tsx
│   │   │   └── ParticipantList.tsx
│   │   ├── voting/               # Voting components
│   │   │   ├── VotingPanel.tsx
│   │   │   ├── VotingCards.tsx
│   │   │   ├── VoteCard.tsx
│   │   │   ├── VoteStatistics.tsx
│   │   │   └── ConsensusIndicator.tsx
│   │   ├── forms/                # Form components
│   │   │   ├── UserSetupForm.tsx
│   │   │   ├── SessionCreator.tsx
│   │   │   ├── JoinSessionForm.tsx
│   │   │   └── validation.ts
│   │   └── controls/             # Control components
│   │       ├── ScrumMasterControls.tsx
│   │       ├── UserManagement.tsx
│   │       └── SessionControls.tsx
│   ├── pages/
│   │   ├── LandingPage.tsx
│   │   ├── SessionBoardroom.tsx
│   │   ├── ErrorPage.tsx
│   │   └── NotFoundPage.tsx
│   ├── hooks/                    # Custom React hooks
│   │   ├── useRealtime.ts       # SSE/WebSocket management
│   │   ├── useSession.ts        # Session-specific logic
│   │   ├── useVoting.ts         # Voting logic
│   │   ├── useSessionTimer.ts   # Timer functionality
│   │   ├── useLocalStorage.ts   # Persistent storage
│   │   ├── useAudio.ts          # Sound notifications
│   │   └── useOptimisticUpdate.ts # Optimistic UI
│   ├── store/
│   │   ├── index.ts              # Store configuration
│   │   ├── middleware/
│   │   │   ├── realtimeMiddleware.ts
│   │   │   └── loggingMiddleware.ts
│   │   ├── slices/
│   │   │   ├── userSlice.ts
│   │   │   ├── sessionSlice.ts
│   │   │   ├── votingSlice.ts
│   │   │   ├── connectionSlice.ts
│   │   │   └── uiSlice.ts
│   │   └── api/
│   │       ├── baseApi.ts       # RTK Query setup
│   │       └── endpoints/
│   │           ├── session.ts
│   │           ├── voting.ts
│   │           └── user.ts
│   ├── services/
│   │   ├── realtimeService.ts   # SSE/WebSocket service
│   │   ├── audioService.ts      # Sound effects
│   │   ├── storageService.ts    # LocalStorage wrapper
│   │   ├── sessionManager.ts    # Session business logic
│   │   └── voteCalculator.ts    # Vote statistics
│   ├── utils/
│   │   ├── constants.ts         # App constants
│   │   ├── helpers.ts           # Utility functions
│   │   ├── types.ts             # TypeScript types
│   │   ├── formatters.ts        # Data formatting
│   │   └── validators.ts        # Input validation
│   ├── styles/
│   │   ├── theme.ts             # MUI theme config
│   │   ├── globals.css          # Global styles
│   │   ├── animations.css       # Animation definitions
│   │   └── modules/             # CSS modules
│   ├── router/
│   │   ├── AppRouter.tsx        # Route definitions
│   │   └── ProtectedRoute.tsx   # Auth guards
│   ├── App.tsx
│   └── main.tsx
├── tests/
│   ├── __mocks__/               # Mock implementations
│   ├── components/              # Component tests
│   ├── hooks/                   # Hook tests
│   ├── store/                   # Redux tests
│   ├── services/                # Service tests
│   ├── integration/             # Integration tests
│   └── e2e/                     # Playwright E2E tests
├── .env.example
├── .eslintrc.json
├── .prettierrc
├── vite.config.ts
├── tsconfig.json
├── package.json
└── README.md
```

## Key Functions and Their Implementations

### Core Business Logic Functions

#### 1. Session Management
```typescript
// handleCreateSession - Creates new planning poker session
async function handleCreateSession(sessionName: string, userName: string): Promise<Session>

// handleJoinSession - Joins existing session with validation
async function handleJoinSession(sessionId: string, userName: string): Promise<void>

// handleLeaveSession - Gracefully exits session with cleanup
async function handleLeaveSession(): Promise<void>

// handleSessionExpiry - Manages 10-minute inactivity timeout
function handleSessionExpiry(): void
```

#### 2. Voting Workflow
```typescript
// handleVoteCast - Submits vote with optimistic update
function handleVoteCast(value: string): void

// handleVoteReveal - Scrum Master reveals all votes
async function handleVoteReveal(): Promise<void>

// handleNewRound - Initiates new voting round
async function handleNewRound(topic: string): Promise<void>

// calculateVoteStatistics - Computes average, distribution, consensus
function calculateVoteStatistics(votes: Map<string, string>): VoteStatistics
```

#### 3. Real-time Synchronization
```typescript
// establishRealtimeConnection - Sets up SSE/WebSocket connection
async function establishRealtimeConnection(sessionId: string): Promise<void>

// handleRealtimeEvent - Processes incoming real-time events
function handleRealtimeEvent(event: RealtimeEvent): void

// handleReconnection - Manages exponential backoff reconnection
async function handleReconnection(): Promise<void>

// syncSessionState - Ensures state consistency after reconnection
async function syncSessionState(): Promise<void>
```

#### 4. User Interface Functions
```typescript
// calculateUserPositions - Positions users around oval table
function calculateUserPositions(users: User[]): Position[]

// renderVotingCard - Displays Fibonacci voting cards
function renderVotingCard(value: string, isSelected: boolean): ReactElement

// animateVoteReveal - Animates card flip on reveal
function animateVoteReveal(): void

// updateVotingIndicator - Shows green/red voting status
function updateVotingIndicator(userId: string, hasVoted: boolean): void
```

#### 5. Scrum Master Controls
```typescript
// transferScrumMaster - Transfers role to another participant
async function transferScrumMaster(newMasterId: string): Promise<void>

// kickParticipant - Removes user from session
async function kickParticipant(userId: string): Promise<void>

// pauseSession - Handles 5-minute grace period
function pauseSession(): void

// endSession - Terminates session for all users
async function endSession(): Promise<void>
```

## Custom Hooks Implementation

### Essential Custom Hooks

```typescript
// useRealtime - Manages SSE/WebSocket connection lifecycle
const useRealtime = (sessionId: string) => {
  // Establishes connection on mount
  // Handles reconnection with exponential backoff
  // Cleans up on unmount
  return { status, reconnect, lastEvent };
};

// useSession - Encapsulates session management logic
const useSession = () => {
  // Provides session CRUD operations
  // Manages participant list updates
  // Handles session expiry
  return { session, joinSession, leaveSession, isLoading };
};

// useVoting - Manages voting workflow
const useVoting = () => {
  // Handles vote submission with optimistic updates
  // Manages vote reveal and statistics
  // Tracks voting round history
  return { currentRound, submitVote, canVote, statistics };
};

// useOptimisticUpdate - Provides optimistic UI updates
const useOptimisticUpdate = (action: AsyncAction) => {
  // Applies immediate UI update
  // Handles rollback on failure
  // Manages loading states
  return { execute, isLoading, error };
};

// useSessionTimer - Countdown timer for session expiry
const useSessionTimer = (expiresAt: string) => {
  // Calculates remaining time
  // Updates every second
  // Triggers expiry callback
  return { remainingTime, isExpired, formattedTime };
};
```

## Testing Strategy (Comprehensive)

### Test Coverage Requirements

#### Unit Tests (30+ test cases)
```typescript
// Component Tests
- "renders voting cards with correct Fibonacci sequence"
- "disables cards after vote submission"
- "displays user avatars in correct positions"
- "shows voting indicators accurately"
- "handles theme switching properly"

// Hook Tests
- "establishes SSE connection on mount"
- "reconnects with exponential backoff on failure"
- "calculates session expiry correctly"
- "manages optimistic updates and rollbacks"

// Service Tests  
- "calculates vote statistics excluding coffee"
- "determines consensus when 80% agreement"
- "formats user positions for 1-16 participants"
- "validates session join with full capacity"

// Redux Tests
- "updates state on real-time events"
- "handles vote submission optimistically"
- "reverts state on API failure"
- "maintains state during reconnection"
```

#### Integration Tests
```typescript
// End-to-end workflows
- "completes full voting cycle from start to reveal"
- "synchronizes state across multiple browser tabs"
- "handles scrum master disconnection with grace period"
- "enforces 16 user per session limit"
- "maintains session during network interruption"
```

#### E2E Tests (Playwright)
```typescript
// Critical user journeys
test.describe('Planning Poker Session', () => {
  test('creates session and invites participants', async ({ page }) => {});
  test('casts vote and reveals results', async ({ page }) => {});
  test('handles multiple concurrent sessions', async ({ browser }) => {});
  test('reconnects after network failure', async ({ page }) => {});
  test('transfers scrum master role', async ({ page }) => {});
});
```

## Performance Optimization Strategy

### React Performance Optimizations
```typescript
// 1. Memoization for expensive computations
const memoizedPositions = useMemo(
  () => calculateUserPositions(participants),
  [participants]
);

// 2. Component memoization for frequent re-renders
const UserAvatar = React.memo(({ user, isVoting }) => {
  // Component implementation
}, (prev, next) => {
  return prev.user.id === next.user.id && 
         prev.isVoting === next.isVoting;
});

// 3. Callback memoization for event handlers
const handleVote = useCallback((value: string) => {
  dispatch(submitVote(value));
}, [dispatch]);

// 4. Lazy loading for non-critical components
const Statistics = lazy(() => import('./Statistics'));

// 5. Virtual scrolling for large lists
const VirtualParticipantList = ({ participants }) => {
  // Virtual scrolling implementation for 16+ users
};
```

### Network Optimization
- Request debouncing for rapid state changes
- Batch API calls where possible
- Implement request caching with RTK Query
- Use compression for SSE payloads

## Development Workflow

### Scripts Configuration
```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:e2e": "playwright test",
    "lint": "eslint src --ext ts,tsx",
    "format": "prettier --write src/**/*.{ts,tsx}",
    "type-check": "tsc --noEmit"
  }
}
```

### Vite Configuration
```typescript
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': 'http://localhost:8000',
      '/sse': {
        target: 'http://localhost:8000',
        changeOrigin: true,
      }
    }
  },
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom', '@mui/material'],
          redux: ['@reduxjs/toolkit', 'react-redux']
        }
      }
    }
  }
});
```

## Implementation Phases (5 Weeks)

### Week 1: Foundation
- Project setup with Vite, React, TypeScript
- Redux store configuration
- Basic routing and layouts
- User setup and preferences

### Week 2: Session Management  
- Session creation and joining
- Participant management
- Basic boardroom interface
- Oval table visualization

### Week 3: Real-time Communication
- SSE implementation with fallback
- Real-time event handling
- Connection status management
- State synchronization

### Week 4: Voting System
- Voting cards interface
- Vote submission and reveal
- Statistics calculation
- Scrum Master controls

### Week 5: Testing and Polish
- Comprehensive test suite
- Performance optimization
- Responsive design
- Accessibility features

## Risk Mitigation

### Technical Risks and Solutions
1. **SSE Browser Compatibility**: Implement WebSocket fallback
2. **State Desynchronization**: Add periodic state reconciliation
3. **Performance with 16 Users**: Use virtualization and memoization
4. **Network Instability**: Exponential backoff with message queuing
5. **Mobile Touch Issues**: Dedicated mobile gesture handling

## Conclusion

This unified plan combines the best practices from both architectural approaches:
- Redux Toolkit for robust state management (from second plan)
- SSE with WebSocket fallback for reliability (from second plan)
- Context API for component-specific state (from first plan)
- Comprehensive testing strategy (merged from both)
- Clear function specifications (from first plan)
- Detailed file organization (from second plan)

The implementation focuses on MVP functionality while maintaining scalability and code quality for future enhancements.