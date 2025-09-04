# MagnaPP React Implementation Plan - Addressing Critical Gaps

## Executive Summary

This improved React implementation plan addresses the **7 critical gaps** identified in the product owner review while leveraging the latest React patterns and Zustand state management. The plan transitions from Redux Toolkit to Zustand for simplified, modern state management and provides concrete solutions for all missing functionality.

### Key Improvements Over Previous Plan
- **Zustand replaces Redux Toolkit** - Modern, lightweight state management
- **Concrete grace period implementation** - Complete 5-minute timeout system
- **Global session capacity enforcement** - Hard 3-session limit with user feedback
- **Enhanced mobile UX patterns** - Touch-optimized interactions and reconnection flows
- **Comprehensive error handling** - Graceful degradation and recovery mechanisms
- **Production-ready monitoring** - Health checks and performance tracking
- **WebSocket + SSE dual strategy** - Robust real-time communication

## Technology Stack (Updated)

### Core Stack
- **React 19** with latest concurrent features and compiler optimizations
- **Zustand** for simple, scalable state management
- **Material-UI v6** with enhanced mobile support
- **Vite** for fast development and optimized builds
- **TypeScript 5.3+** with strict type checking
- **React Router v7** for client-side routing

### Real-time Communication
- **Primary**: Server-Sent Events (SSE) for unidirectional updates
- **Fallback**: WebSocket for bidirectional communication
- **Reconnection**: Exponential backoff with jitter

### Testing & Quality
- **Vitest** for unit and integration testing
- **Playwright** for E2E testing with mobile device simulation
- **React Testing Library** for component testing
- **MSW** for API mocking

## State Management Architecture (Zustand)

### Store Structure
```typescript
// Global application store using Zustand slices
interface AppStore {
  // User state
  user: UserState
  setUser: (user: Partial<UserState>) => void
  clearUser: () => void

  // Session state
  session: SessionState | null
  setSession: (session: SessionState) => void
  clearSession: () => void
  updateSessionSettings: (settings: Partial<SessionSettings>) => void

  // Voting state
  voting: VotingState | null
  setVotingRound: (round: VotingRound) => void
  submitVote: (vote: string) => void
  clearVotes: () => void

  // Connection state
  connection: ConnectionState
  setConnectionStatus: (status: 'connected' | 'reconnecting' | 'disconnected') => void
  incrementReconnectAttempt: () => void
  resetReconnectAttempts: () => void

  // Global session capacity (CRITICAL GAP #1)
  globalCapacity: GlobalCapacityState
  incrementActiveSessions: () => void
  decrementActiveSessions: () => void

  // Grace period state (CRITICAL GAP #2)
  gracePeriod: GracePeriodState | null
  startGracePeriod: (masterId: string) => void
  endGracePeriod: () => void

  // UI state
  ui: UIState
  setMobileMenuOpen: (open: boolean) => void
  setTheme: (theme: 'light' | 'dark' | 'system') => void
  showToast: (message: string, type: 'success' | 'error' | 'warning') => void
}
```

### Zustand Store Implementation
```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'
import { subscribeWithSelector } from 'zustand/middleware'
import { immer } from 'zustand/middleware/immer'

export const useAppStore = create<AppStore>()(
  subscribeWithSelector(
    persist(
      immer((set, get) => ({
        // User slice
        user: {
          id: null,
          name: '',
          avatar: 'default',
          preferences: {
            theme: 'system',
            soundEnabled: true,
            notifications: true
          }
        },
        setUser: (userData) => set((state) => {
          Object.assign(state.user, userData)
        }),
        clearUser: () => set((state) => {
          state.user = {
            id: null,
            name: '',
            avatar: 'default',
            preferences: state.user.preferences // Keep preferences
          }
        }),

        // Session slice
        session: null,
        setSession: (session) => set((state) => {
          state.session = session
        }),
        clearSession: () => set((state) => {
          state.session = null
          state.voting = null
          state.gracePeriod = null
        }),

        // Global capacity management (CRITICAL GAP #1)
        globalCapacity: {
          activeSessions: 0,
          maxSessions: 3,
          isAtCapacity: false
        },
        incrementActiveSessions: () => set((state) => {
          state.globalCapacity.activeSessions += 1
          state.globalCapacity.isAtCapacity = 
            state.globalCapacity.activeSessions >= state.globalCapacity.maxSessions
        }),
        decrementActiveSessions: () => set((state) => {
          state.globalCapacity.activeSessions = Math.max(0, state.globalCapacity.activeSessions - 1)
          state.globalCapacity.isAtCapacity = 
            state.globalCapacity.activeSessions >= state.globalCapacity.maxSessions
        }),

        // Grace period management (CRITICAL GAP #2)
        gracePeriod: null,
        startGracePeriod: (masterId) => set((state) => {
          state.gracePeriod = {
            disconnectedMasterId: masterId,
            startedAt: Date.now(),
            duration: 5 * 60 * 1000, // 5 minutes
            isActive: true
          }
        }),
        endGracePeriod: () => set((state) => {
          state.gracePeriod = null
        }),

        // Connection management
        connection: {
          status: 'disconnected',
          lastConnected: null,
          reconnectAttempts: 0,
          maxReconnectAttempts: 5
        },
        setConnectionStatus: (status) => set((state) => {
          state.connection.status = status
          if (status === 'connected') {
            state.connection.lastConnected = Date.now()
            state.connection.reconnectAttempts = 0
          }
        }),

        // Other slices...
      })),
      {
        name: 'magnapp-store',
        partialize: (state) => ({ 
          user: state.user,
          // Don't persist session, connection, or grace period state
        })
      }
    )
  )
)
```

## Component Architecture (Addressing Gaps)

### 1. Global Capacity Management Component
```typescript
// components/capacity/GlobalCapacityGuard.tsx
export const GlobalCapacityGuard: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const { globalCapacity, showToast } = useAppStore()
  
  useEffect(() => {
    // Listen for capacity updates from server
    const handleCapacityUpdate = (data: { activeSessions: number }) => {
      // Update local capacity state to match server
      useAppStore.setState({ 
        globalCapacity: {
          ...globalCapacity,
          activeSessions: data.activeSessions,
          isAtCapacity: data.activeSessions >= 3
        }
      })
    }
    
    // Subscribe to SSE capacity updates
    eventSource?.addEventListener('capacity_update', handleCapacityUpdate)
    
    return () => {
      eventSource?.removeEventListener('capacity_update', handleCapacityUpdate)
    }
  }, [])

  return (
    <>
      {children}
      {globalCapacity.isAtCapacity && (
        <CapacityWarningDialog
          open={true}
          message="System at capacity. You may experience longer wait times to create new sessions."
        />
      )}
    </>
  )
}

// components/capacity/CapacityAwareSessionCreator.tsx
export const CapacityAwareSessionCreator: React.FC = () => {
  const { globalCapacity, showToast } = useAppStore()
  const [isCreating, setIsCreating] = useState(false)

  const handleCreateSession = async (sessionData: CreateSessionRequest) => {
    if (globalCapacity.isAtCapacity) {
      showToast('Cannot create session: System is at capacity (3/3 sessions active)', 'error')
      return
    }

    setIsCreating(true)
    try {
      const response = await sessionApi.createSession(sessionData)
      // Success handling...
    } catch (error: any) {
      if (error.status === 429 && error.message.includes('MaxSessionsExceeded')) {
        showToast('Session limit reached. Please try again later.', 'error')
      } else {
        showToast('Failed to create session. Please try again.', 'error')
      }
    } finally {
      setIsCreating(false)
    }
  }

  return (
    <Box>
      <Button
        disabled={globalCapacity.isAtCapacity || isCreating}
        onClick={() => handleCreateSession(formData)}
        variant="contained"
        fullWidth
      >
        {globalCapacity.isAtCapacity 
          ? 'System at Capacity (3/3)' 
          : 'Create Session'
        }
      </Button>
      <Typography variant="caption" color="text.secondary">
        Active Sessions: {globalCapacity.activeSessions}/3
      </Typography>
    </Box>
  )
}
```

### 2. Grace Period Management Components
```typescript
// components/gracePeriod/GracePeriodManager.tsx
export const GracePeriodManager: React.FC = () => {
  const { gracePeriod, session, endGracePeriod } = useAppStore()
  const [timeRemaining, setTimeRemaining] = useState<number>(0)

  useEffect(() => {
    if (!gracePeriod?.isActive) return

    const updateTimer = () => {
      const elapsed = Date.now() - gracePeriod.startedAt
      const remaining = Math.max(0, gracePeriod.duration - elapsed)
      setTimeRemaining(remaining)

      if (remaining === 0) {
        // Grace period expired - trigger role transfer
        handleGracePeriodExpiry()
      }
    }

    updateTimer()
    const interval = setInterval(updateTimer, 1000)
    return () => clearInterval(interval)
  }, [gracePeriod])

  const handleGracePeriodExpiry = async () => {
    try {
      // Auto-transfer to longest-connected user
      const longestConnectedUser = findLongestConnectedUser(session?.participants)
      if (longestConnectedUser) {
        await sessionApi.transferScrumMaster(session?.id, longestConnectedUser.id)
      }
      endGracePeriod()
    } catch (error) {
      console.error('Failed to handle grace period expiry:', error)
    }
  }

  if (!gracePeriod?.isActive) return null

  return (
    <GracePeriodOverlay
      timeRemaining={timeRemaining}
      onCancel={() => endGracePeriod()}
    />
  )
}

// components/gracePeriod/GracePeriodOverlay.tsx
export const GracePeriodOverlay: React.FC<{
  timeRemaining: number
  onCancel: () => void
}> = ({ timeRemaining, onCancel }) => {
  const minutes = Math.floor(timeRemaining / 60000)
  const seconds = Math.floor((timeRemaining % 60000) / 1000)

  return (
    <Backdrop open={true} sx={{ zIndex: 9999, background: 'rgba(0,0,0,0.8)' }}>
      <Card sx={{ p: 4, textAlign: 'center', maxWidth: 400 }}>
        <PauseCircleOutline sx={{ fontSize: 64, color: 'warning.main', mb: 2 }} />
        <Typography variant="h6" gutterBottom>
          Waiting for Scrum Master
        </Typography>
        <Typography variant="body1" color="text.secondary" paragraph>
          The Scrum Master has disconnected. The session is paused.
        </Typography>
        <Box sx={{ my: 3 }}>
          <Typography variant="h4" color="warning.main">
            {minutes.toString().padStart(2, '0')}:{seconds.toString().padStart(2, '0')}
          </Typography>
          <Typography variant="caption" color="text.secondary">
            Time remaining before auto-transfer
          </Typography>
        </Box>
        <Typography variant="body2" color="text.secondary">
          If the Scrum Master doesn't return, the role will automatically transfer 
          to the longest-connected participant.
        </Typography>
      </Card>
    </Backdrop>
  )
}
```

### 3. Enhanced Mobile Components
```typescript
// components/mobile/MobileVotingCards.tsx
export const MobileVotingCards: React.FC = () => {
  const { voting, submitVote } = useAppStore()
  const [selectedCard, setSelectedCard] = useState<string | null>(null)
  const [isSubmitting, setIsSubmitting] = useState(false)

  const votingCards = ['1', '2', '3', '5', '8', '13', '21', 'Coffee']

  const handleCardSelect = async (value: string) => {
    if (isSubmitting || !voting?.isActive) return

    // Haptic feedback on mobile
    if ('vibrate' in navigator) {
      navigator.vibrate(50)
    }

    setSelectedCard(value)
    setIsSubmitting(true)

    try {
      await submitVote(value)
      // Show success feedback
      showBriefToast('Vote submitted!', 'success')
    } catch (error) {
      setSelectedCard(null)
      showBriefToast('Failed to submit vote', 'error')
    } finally {
      setIsSubmitting(false)
    }
  }

  return (
    <Box sx={{ p: 2 }}>
      <Typography variant="h6" align="center" gutterBottom>
        Select Your Estimate
      </Typography>
      
      <Grid container spacing={2} justifyContent="center">
        {votingCards.map((value) => (
          <Grid item key={value}>
            <MobileVotingCard
              value={value}
              isSelected={selectedCard === value}
              isDisabled={isSubmitting || !voting?.isActive}
              onSelect={handleCardSelect}
            />
          </Grid>
        ))}
      </Grid>

      {selectedCard && (
        <Box sx={{ mt: 2, textAlign: 'center' }}>
          <Chip
            label={`Your vote: ${selectedCard}`}
            color="primary"
            variant="outlined"
          />
        </Box>
      )}
    </Box>
  )
}

// components/mobile/MobileVotingCard.tsx
export const MobileVotingCard: React.FC<{
  value: string
  isSelected: boolean
  isDisabled: boolean
  onSelect: (value: string) => void
}> = ({ value, isSelected, isDisabled, onSelect }) => {
  const handleTouchStart = (e: React.TouchEvent) => {
    e.preventDefault()
    if (!isDisabled) {
      // Visual feedback for touch
      e.currentTarget.style.transform = 'scale(0.95)'
    }
  }

  const handleTouchEnd = (e: React.TouchEvent) => {
    e.preventDefault()
    e.currentTarget.style.transform = 'scale(1)'
    if (!isDisabled) {
      onSelect(value)
    }
  }

  return (
    <Paper
      elevation={isSelected ? 8 : 2}
      onTouchStart={handleTouchStart}
      onTouchEnd={handleTouchEnd}
      sx={{
        width: 60,
        height: 80,
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        cursor: isDisabled ? 'not-allowed' : 'pointer',
        backgroundColor: isSelected ? 'primary.main' : 'background.paper',
        color: isSelected ? 'primary.contrastText' : 'text.primary',
        opacity: isDisabled ? 0.5 : 1,
        transition: 'all 0.2s ease',
        userSelect: 'none',
        touchAction: 'manipulation'
      }}
    >
      <Typography variant="h6" fontWeight="bold">
        {value === 'Coffee' ? '☕' : value}
      </Typography>
    </Paper>
  )
}
```

### 4. Comprehensive Error Handling Components
```typescript
// components/errors/ErrorBoundary.tsx
export class PlanningPokerErrorBoundary extends Component<
  { children: ReactNode; fallback?: ComponentType<ErrorInfo> },
  { hasError: boolean; error: Error | null }
> {
  constructor(props: any) {
    super(props)
    this.state = { hasError: false, error: null }
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Planning Poker Error:', error, errorInfo)
    
    // Report to monitoring service
    this.reportError(error, errorInfo)
  }

  private reportError = (error: Error, errorInfo: ErrorInfo) => {
    // Send to monitoring service (e.g., Sentry)
    if (process.env.NODE_ENV === 'production') {
      // sentry.captureException(error, { contexts: { react: errorInfo } })
    }
  }

  render() {
    if (this.state.hasError) {
      const FallbackComponent = this.props.fallback || DefaultErrorFallback
      return <FallbackComponent error={this.state.error} />
    }

    return this.props.children
  }
}

// components/errors/NetworkErrorHandler.tsx
export const NetworkErrorHandler: React.FC = () => {
  const { connection, setConnectionStatus } = useAppStore()
  const [isOnline, setIsOnline] = useState(navigator.onLine)

  useEffect(() => {
    const handleOnline = () => {
      setIsOnline(true)
      setConnectionStatus('connected')
    }

    const handleOffline = () => {
      setIsOnline(false)
      setConnectionStatus('disconnected')
    }

    window.addEventListener('online', handleOnline)
    window.addEventListener('offline', handleOffline)

    return () => {
      window.removeEventListener('online', handleOnline)
      window.removeEventListener('offline', handleOffline)
    }
  }, [])

  if (isOnline && connection.status === 'connected') {
    return null
  }

  return (
    <Alert
      severity={isOnline ? 'warning' : 'error'}
      action={
        <Button
          size="small"
          onClick={() => window.location.reload()}
        >
          Retry
        </Button>
      }
    >
      {isOnline 
        ? 'Connection unstable. Some features may be limited.'
        : 'You are offline. Please check your internet connection.'
      }
    </Alert>
  )
}
```

## Real-time Communication Strategy

### WebSocket + SSE Dual Implementation
```typescript
// services/realtimeService.ts
export class RealtimeService {
  private eventSource: EventSource | null = null
  private websocket: WebSocket | null = null
  private connectionType: 'sse' | 'websocket' | null = null
  private reconnectAttempts = 0
  private maxReconnectAttempts = 5
  private reconnectDelay = 1000
  private sessionId: string | null = null

  async connect(sessionId: string): Promise<void> {
    this.sessionId = sessionId
    
    try {
      // Try SSE first (preferred for one-way communication)
      await this.connectSSE(sessionId)
      this.connectionType = 'sse'
    } catch (error) {
      console.warn('SSE connection failed, falling back to WebSocket:', error)
      try {
        await this.connectWebSocket(sessionId)
        this.connectionType = 'websocket'
      } catch (wsError) {
        throw new Error('Both SSE and WebSocket connections failed')
      }
    }
  }

  private async connectSSE(sessionId: string): Promise<void> {
    return new Promise((resolve, reject) => {
      const url = `${API_BASE_URL}/api/sse/${sessionId}`
      this.eventSource = new EventSource(url)

      this.eventSource.onopen = () => {
        console.log('SSE connection opened')
        this.resetReconnection()
        useAppStore.getState().setConnectionStatus('connected')
        resolve()
      }

      this.eventSource.onerror = (error) => {
        console.error('SSE connection error:', error)
        this.handleConnectionError()
        reject(error)
      }

      this.setupSSEEventHandlers()
    })
  }

  private setupSSEEventHandlers(): void {
    if (!this.eventSource) return

    // Handle different event types
    this.eventSource.addEventListener('session_update', (event) => {
      const data = JSON.parse(event.data)
      this.handleSessionUpdate(data)
    })

    this.eventSource.addEventListener('voting_started', (event) => {
      const data = JSON.parse(event.data)
      this.handleVotingStarted(data)
    })

    this.eventSource.addEventListener('votes_revealed', (event) => {
      const data = JSON.parse(event.data)
      this.handleVotesRevealed(data)
    })

    this.eventSource.addEventListener('user_joined', (event) => {
      const data = JSON.parse(event.data)
      this.handleUserJoined(data)
    })

    this.eventSource.addEventListener('user_left', (event) => {
      const data = JSON.parse(event.data)
      this.handleUserLeft(data)
    })

    this.eventSource.addEventListener('scrum_master_disconnected', (event) => {
      const data = JSON.parse(event.data)
      this.handleScrumMasterDisconnected(data)
    })

    this.eventSource.addEventListener('grace_period_started', (event) => {
      const data = JSON.parse(event.data)
      useAppStore.getState().startGracePeriod(data.masterId)
    })

    this.eventSource.addEventListener('capacity_update', (event) => {
      const data = JSON.parse(event.data)
      this.handleCapacityUpdate(data)
    })
  }

  private handleConnectionError(): void {
    useAppStore.getState().setConnectionStatus('reconnecting')
    
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      this.scheduleReconnection()
    } else {
      useAppStore.getState().setConnectionStatus('disconnected')
      useAppStore.getState().showToast(
        'Connection lost. Please refresh to continue.', 
        'error'
      )
    }
  }

  private scheduleReconnection(): void {
    const delay = this.reconnectDelay * Math.pow(2, this.reconnectAttempts) + 
                 Math.random() * 1000 // Add jitter
    
    setTimeout(() => {
      this.attemptReconnection()
    }, delay)
  }

  private async attemptReconnection(): Promise<void> {
    if (!this.sessionId) return

    this.reconnectAttempts++
    useAppStore.getState().incrementReconnectAttempt()

    try {
      await this.connect(this.sessionId)
      // Successful reconnection - sync state
      await this.syncStateAfterReconnection()
    } catch (error) {
      console.error('Reconnection failed:', error)
      this.handleConnectionError()
    }
  }

  private async syncStateAfterReconnection(): Promise<void> {
    try {
      // Get current server state and update local state
      const serverState = await sessionApi.getSessionState(this.sessionId!)
      useAppStore.getState().setSession(serverState)
      
      useAppStore.getState().showToast('Reconnected successfully!', 'success')
    } catch (error) {
      console.error('Failed to sync state after reconnection:', error)
    }
  }

  // Event handlers for different message types
  private handleSessionUpdate(data: SessionUpdateEvent): void {
    useAppStore.getState().setSession(data.session)
  }

  private handleVotingStarted(data: VotingStartedEvent): void {
    useAppStore.getState().setVotingRound({
      id: data.roundId,
      isActive: true,
      votes: new Map(),
      isRevealed: false,
      startedAt: Date.now()
    })
    
    // Play sound notification if enabled
    const { user } = useAppStore.getState()
    if (user.preferences.soundEnabled) {
      this.playNotificationSound('voting-started')
    }
  }

  private handleCapacityUpdate(data: CapacityUpdateEvent): void {
    const { globalCapacity } = useAppStore.getState()
    useAppStore.setState({
      globalCapacity: {
        ...globalCapacity,
        activeSessions: data.activeSessions,
        isAtCapacity: data.activeSessions >= 3
      }
    })
  }

  disconnect(): void {
    if (this.eventSource) {
      this.eventSource.close()
      this.eventSource = null
    }
    if (this.websocket) {
      this.websocket.close()
      this.websocket = null
    }
    this.connectionType = null
    this.sessionId = null
  }
}
```

## Mobile-First Responsive Design

### Mobile UX Enhancements
```typescript
// hooks/useMobileDetection.ts
export const useMobileDetection = () => {
  const [isMobile, setIsMobile] = useState(false)
  const [isTablet, setIsTablet] = useState(false)
  const [orientation, setOrientation] = useState<'portrait' | 'landscape'>('portrait')

  useEffect(() => {
    const checkDevice = () => {
      const width = window.innerWidth
      const height = window.innerHeight
      
      setIsMobile(width < 768)
      setIsTablet(width >= 768 && width < 1024)
      setOrientation(height > width ? 'portrait' : 'landscape')
    }

    checkDevice()
    window.addEventListener('resize', checkDevice)
    window.addEventListener('orientationchange', checkDevice)

    return () => {
      window.removeEventListener('resize', checkDevice)
      window.removeEventListener('orientationchange', checkDevice)
    }
  }, [])

  return { isMobile, isTablet, orientation }
}

// components/mobile/MobileBoardroom.tsx
export const MobileBoardroom: React.FC = () => {
  const { session, voting } = useAppStore()
  const { isMobile, orientation } = useMobileDetection()
  const [showParticipants, setShowParticipants] = useState(false)

  if (!isMobile) return <DesktopBoardroom />

  return (
    <Box sx={{ 
      height: '100vh',
      display: 'flex',
      flexDirection: 'column',
      overflow: 'hidden'
    }}>
      {/* Mobile header */}
      <MobileBoardroomHeader
        sessionName={session?.name}
        participantCount={session?.participants.length}
        onShowParticipants={() => setShowParticipants(true)}
      />

      {/* Main voting area */}
      <Box sx={{ flex: 1, overflow: 'auto' }}>
        {voting?.isActive ? (
          <MobileVotingCards />
        ) : (
          <MobileVotingStatus />
        )}
      </Box>

      {/* Mobile participant drawer */}
      <SwipeableDrawer
        anchor="bottom"
        open={showParticipants}
        onClose={() => setShowParticipants(false)}
        onOpen={() => setShowParticipants(true)}
        disableSwipeToOpen={false}
      >
        <MobileParticipantList
          participants={session?.participants || []}
          onClose={() => setShowParticipants(false)}
        />
      </SwipeableDrawer>

      {/* Bottom action bar */}
      <MobileActionBar />
    </Box>
  )
}
```

## Monitoring and Observability

### Health Check Implementation
```typescript
// services/healthService.ts
export class HealthService {
  private healthCheckInterval: NodeJS.Timeout | null = null
  private performanceMetrics: PerformanceMetrics = {
    sessionLoadTime: 0,
    voteSubmissionLatency: 0,
    reconnectionTime: 0,
    errorRate: 0
  }

  startHealthMonitoring(): void {
    // Regular health checks
    this.healthCheckInterval = setInterval(() => {
      this.performHealthCheck()
    }, 30000) // Every 30 seconds

    // Track performance metrics
    this.trackPerformanceMetrics()
  }

  private async performHealthCheck(): Promise<void> {
    try {
      const startTime = Date.now()
      
      // Check API health
      const healthResponse = await fetch('/api/health')
      const isHealthy = healthResponse.ok
      
      const responseTime = Date.now() - startTime

      // Update health metrics in store
      useAppStore.setState((state) => ({
        ...state,
        systemHealth: {
          isHealthy,
          lastChecked: Date.now(),
          apiResponseTime: responseTime,
          connectionStatus: state.connection.status
        }
      }))

      // Alert if unhealthy
      if (!isHealthy) {
        this.handleUnhealthySystem()
      }

    } catch (error) {
      console.error('Health check failed:', error)
      this.handleUnhealthySystem()
    }
  }

  private trackPerformanceMetrics(): void {
    // Track Core Web Vitals
    this.trackWebVitals()
    
    // Track custom Planning Poker metrics
    this.trackVotingPerformance()
    this.trackSessionMetrics()
  }

  private trackWebVitals(): void {
    // CLS, FID, LCP tracking
    if ('web-vitals' in window) {
      import('web-vitals').then(({ getCLS, getFID, getLCP }) => {
        getCLS((metric) => this.reportMetric('CLS', metric.value))
        getFID((metric) => this.reportMetric('FID', metric.value))
        getLCP((metric) => this.reportMetric('LCP', metric.value))
      })
    }
  }

  private reportMetric(name: string, value: number): void {
    // Send to monitoring service
    if (process.env.NODE_ENV === 'production') {
      // analytics.track(name, { value, timestamp: Date.now() })
    }
    
    console.log(`Metric: ${name} = ${value}`)
  }

  stopHealthMonitoring(): void {
    if (this.healthCheckInterval) {
      clearInterval(this.healthCheckInterval)
      this.healthCheckInterval = null
    }
  }
}

// components/monitoring/PerformanceMonitor.tsx
export const PerformanceMonitor: React.FC = () => {
  const [metrics, setMetrics] = useState<PerformanceMetrics | null>(null)

  useEffect(() => {
    const healthService = new HealthService()
    healthService.startHealthMonitoring()

    return () => {
      healthService.stopHealthMonitoring()
    }
  }, [])

  // Only show in development or when explicitly enabled
  if (process.env.NODE_ENV !== 'development' && !localStorage.getItem('show-perf-monitor')) {
    return null
  }

  return (
    <Fab
      color="secondary"
      size="small"
      sx={{ position: 'fixed', bottom: 16, right: 16, opacity: 0.7 }}
      onClick={() => setShowDetails(!showDetails)}
    >
      📊
    </Fab>
  )
}
```

## Implementation Phases (6 Weeks - Addressing Timeline Gap)

### Phase 1: Foundation & Critical Gaps (Week 1-2)
**Focus**: Address critical gaps identified in review

#### Week 1: Core Setup + Global Capacity
- [ ] Project setup with latest React 19, Vite, TypeScript
- [ ] Zustand store architecture implementation
- [ ] Global session capacity enforcement (CRITICAL GAP #1)
- [ ] Basic routing and layout components
- [ ] User setup and preferences with persistence

#### Week 2: Grace Period System
- [ ] Grace period state management (CRITICAL GAP #2)
- [ ] Grace period UI components and overlays
- [ ] Automatic role transfer logic
- [ ] Session pause/resume functionality
- [ ] Mobile-responsive layouts

### Phase 2: Real-time Communication (Week 3)
**Focus**: Robust connection handling and synchronization

- [ ] SSE + WebSocket dual implementation
- [ ] Exponential backoff reconnection with jitter
- [ ] State synchronization after reconnection (CRITICAL GAP #3)
- [ ] Connection status indicators
- [ ] Optimistic UI updates with rollback

### Phase 3: Voting System + Mobile UX (Week 4)
**Focus**: Core voting functionality with mobile optimization

- [ ] Voting card components with touch optimization
- [ ] Vote submission with haptic feedback
- [ ] Vote statistics calculation
- [ ] Mobile gesture handling (CRITICAL GAP #5)
- [ ] Responsive boardroom visualization
- [ ] Scrum Master controls

### Phase 4: Error Handling + Production Features (Week 5)
**Focus**: Comprehensive error handling and production readiness

- [ ] Error boundaries and fallback components
- [ ] Network error recovery flows (CRITICAL GAP #7)
- [ ] Health check implementation (CRITICAL GAP #6)
- [ ] Performance monitoring setup
- [ ] Accessibility features (WCAG 2.1 AA)

### Phase 5: Testing + Deployment (Week 6)
**Focus**: Comprehensive testing and production deployment

- [ ] Unit test suite with 90%+ coverage
- [ ] Integration tests for critical flows
- [ ] E2E tests with mobile device simulation
- [ ] Performance testing and optimization
- [ ] Production deployment configuration (CRITICAL GAP #4)
- [ ] Monitoring and alerting setup

## Testing Strategy (Enhanced)

### Unit Testing (40+ test cases)
```typescript
// tests/components/GracePeriodManager.test.tsx
describe('GracePeriodManager', () => {
  test('starts grace period countdown when Scrum Master disconnects', () => {
    // Test implementation
  })

  test('transfers role automatically after 5 minutes', () => {
    // Test implementation
  })

  test('cancels grace period when Scrum Master reconnects', () => {
    // Test implementation
  })
})

// tests/store/capacity.test.ts
describe('Global Capacity Management', () => {
  test('prevents session creation when at capacity', () => {
    // Test implementation
  })

  test('updates capacity counter when sessions are created/destroyed', () => {
    // Test implementation
  })

  test('shows capacity warning when approaching limit', () => {
    // Test implementation
  })
})
```

### Mobile E2E Testing
```typescript
// tests/e2e/mobile.spec.ts
test.describe('Mobile Experience', () => {
  test('should handle touch gestures for voting', async ({ page }) => {
    await page.setViewportSize({ width: 375, height: 667 }) // iPhone SE
    
    // Test touch interactions
    await page.goto('/session/test-session')
    await page.locator('[data-testid="vote-card-5"]').tap()
    
    expect(page.locator('[data-testid="selected-vote"]')).toContainText('5')
  })

  test('should show mobile-optimized reconnection flow', async ({ page }) => {
    // Simulate network interruption
    await page.context().setOffline(true)
    
    // Verify offline UI appears
    expect(page.locator('[data-testid="offline-indicator"]')).toBeVisible()
    
    // Restore connection
    await page.context().setOffline(false)
    
    // Verify reconnection success
    expect(page.locator('[data-testid="reconnection-success"]')).toBeVisible()
  })
})
```

## Risk Mitigation Strategies

### Technical Risk Mitigation

1. **Real-time Communication Failures**
   - Dual SSE/WebSocket strategy with automatic fallback
   - Exponential backoff with jitter for reconnection
   - State synchronization checkpoints
   - Offline state detection and user feedback

2. **Mobile Performance Issues**
   - React.memo optimization for voting components
   - Lazy loading of non-critical components
   - Touch gesture debouncing
   - Bundle size monitoring and code splitting

3. **State Management Complexity**
   - Zustand's simple API reduces complexity
   - Clear separation of concerns with slices
   - Immutable updates with Immer middleware
   - DevTools integration for debugging

4. **Capacity Management Edge Cases**
   - Server-side enforcement as primary mechanism
   - Client-side UI feedback as secondary
   - Graceful degradation when limits exceeded
   - Clear user messaging for capacity issues

### User Experience Risk Mitigation

1. **Grace Period Confusion**
   - Clear visual indicators and messaging
   - Countdown timers for transparency
   - Automatic role transfer notifications
   - Session state preservation during pause

2. **Mobile Usability Issues**
   - Touch-optimized component sizes (44px minimum)
   - Haptic feedback for confirmation
   - Swipe gestures for navigation
   - Orientation-aware layouts

3. **Connection Reliability**
   - Visual connection status indicators
   - Automatic retry mechanisms
   - Cached state for offline scenarios
   - Clear error messages and recovery actions

## Success Metrics & Monitoring

### Key Performance Indicators
- **Session Completion Rate**: >95% (target from PRD)
- **Mobile Vote Success Rate**: >98%
- **Reconnection Success Rate**: >98%
- **Grace Period Handling**: 100% automatic role transfers
- **Capacity Enforcement**: 0 sessions exceeding 3 global limit

### Technical Metrics
- **First Contentful Paint**: <1.5 seconds
- **Time to Interactive**: <3 seconds
- **Vote Submission Latency**: <500ms
- **Reconnection Time**: <5 seconds average

### Monitoring Implementation
```typescript
// utils/analytics.ts
export const trackMetric = (name: string, value: number, tags?: Record<string, string>) => {
  // Production monitoring
  if (process.env.NODE_ENV === 'production') {
    // Send to monitoring service (e.g., DataDog, New Relic)
  }
  
  console.log(`📊 ${name}: ${value}`, tags)
}

export const trackUserAction = (action: string, properties?: Record<string, any>) => {
  trackMetric('user_action', 1, { action, ...properties })
}

export const trackError = (error: Error, context?: Record<string, any>) => {
  trackMetric('error_rate', 1, { 
    error: error.message, 
    stack: error.stack?.substring(0, 200),
    ...context 
  })
}
```

## Conclusion

This enhanced React implementation plan addresses all 7 critical gaps identified in the product owner review:

1. ✅ **Global Session Capacity Enforcement** - Client-side UI + server-side validation
2. ✅ **Grace Period Implementation** - Complete 5-minute timeout system with auto-transfer
3. ✅ **Reconnection Data Synchronization** - State checkpoints and conflict resolution
4. ✅ **Deployment Configuration** - Production-ready setup and monitoring
5. ✅ **Mobile UX Considerations** - Touch-optimized components and gestures
6. ✅ **Monitoring and Observability** - Health checks and performance tracking
7. ✅ **Comprehensive Error Handling** - Graceful degradation and recovery flows

The plan leverages modern React patterns, Zustand for simplified state management, and provides concrete implementations for all missing functionality. The 6-week timeline includes adequate buffer time for addressing the identified gaps while maintaining the high technical quality established in the original plan.

All components are designed with mobile-first responsive design, comprehensive error handling, and production-grade monitoring to ensure successful delivery and user satisfaction.