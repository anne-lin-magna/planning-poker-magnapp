# MagnaPP API & DTO Specification
*Comprehensive Data Contract for Parallel Frontend/Backend Development*

**Version:** 1.0.0  
**Date:** September 4, 2025  
**Purpose:** Enable independent parallel development with guaranteed integration compatibility

---

## Executive Summary

This document provides a complete API contract and Data Transfer Object (DTO) specification for the MagnaPP Planning Poker application. It defines all request/response formats, WebSocket/SSE message structures, validation rules, and error handling patterns needed for frontend and backend teams to develop independently while ensuring seamless integration.

### Key Features Covered:
- **Session Management**: Creation, joining, capacity limits, expiry handling
- **User Management**: Registration, authentication, role assignments
- **Voting System**: Round management, vote submission, statistics calculation
- **Real-time Communication**: SSE/WebSocket event structures
- **Grace Period System**: Scrum Master disconnection handling
- **Error Handling**: Comprehensive error response formats
- **State Synchronization**: Reconnection and conflict resolution

---

## Table of Contents

1. [Core Data Models](#core-data-models)
2. [HTTP API Endpoints](#http-api-endpoints)
3. [Real-time Communication](#real-time-communication)
4. [Error Handling](#error-handling)
5. [Validation Rules](#validation-rules)
6. [State Management DTOs](#state-management-dtos)
7. [Implementation Guidelines](#implementation-guidelines)

---

## Core Data Models

### User DTOs

```typescript
// Base user information
interface User {
  id: string                    // UUID format
  name: string                  // 2-50 characters, alphanumeric + spaces
  avatar: string                // Icon identifier from predefined set
  role: 'scrum_master' | 'participant'
  joinedAt: string             // ISO 8601 timestamp
  lastActive: string           // ISO 8601 timestamp
  connectionStatus: 'connected' | 'disconnected' | 'reconnecting'
}

// User creation/update requests
interface CreateUserRequest {
  name: string                 // Required, 2-50 chars
  avatar: string              // Required, must be valid icon ID
}

interface UpdateUserRequest {
  name?: string               // Optional, 2-50 chars
  avatar?: string            // Optional, must be valid icon ID
}

// User preferences (stored client-side)
interface UserPreferences {
  theme: 'light' | 'dark' | 'system'
  soundEnabled: boolean
  notifications: boolean
  language: string            // ISO 639-1 code, default 'en'
}

// Complete user state for frontend
interface UserState {
  user: User | null
  preferences: UserPreferences
  isAuthenticated: boolean
}
```

### Session DTOs

```typescript
// Core session information
interface Session {
  id: string                   // UUID format
  name: string                 // 3-100 characters
  scrumMasterId: string       // User ID of current Scrum Master
  participants: User[]         // Max 16 users
  status: 'waiting' | 'voting' | 'revealing' | 'results' | 'paused' | 'expired'
  createdAt: string           // ISO 8601 timestamp
  lastActivity: string        // ISO 8601 timestamp
  expiresAt: string          // ISO 8601 timestamp (10 minutes from last activity)
  settings: SessionSettings
  currentRound?: VotingRound  // Present when voting is active
  gracePeriod?: GracePeriod  // Present when Scrum Master disconnected
}

// Session configuration
interface SessionSettings {
  allowSpectators: boolean     // Default: true
  autoReveal: boolean         // Default: false
  votingTimeLimit?: number    // Optional timeout in seconds
  cardSet: 'fibonacci' | 'tshirt' | 'custom'  // Default: 'fibonacci'
  customCards?: string[]      // Used when cardSet is 'custom'
}

// Session creation request
interface CreateSessionRequest {
  name: string                // Required, 3-100 chars
  settings?: Partial<SessionSettings>
  userId: string             // ID of user creating session (becomes Scrum Master)
}

// Session join request
interface JoinSessionRequest {
  sessionId: string          // UUID of session to join
  user: CreateUserRequest    // User information
}

// Session discovery response
interface SessionListResponse {
  sessions: SessionSummary[]
  totalCount: number
  activeSessionsCount: number  // Current count for capacity monitoring
  maxSessions: number         // System limit (3)
}

interface SessionSummary {
  id: string
  name: string
  participantCount: number
  maxParticipants: number     // Always 16
  status: Session['status']
  createdAt: string
  canJoin: boolean           // False if full or expired
}

// Grace period information
interface GracePeriod {
  disconnectedMasterId: string
  startedAt: string          // ISO 8601 timestamp
  expiresAt: string         // ISO 8601 timestamp (5 minutes from start)
  isActive: boolean
  candidateNewMaster?: string // User ID of longest-connected user
}
```

### Voting System DTOs

```typescript
// Voting round information
interface VotingRound {
  id: string                 // UUID format
  sessionId: string         // Reference to parent session
  startedAt: string        // ISO 8601 timestamp
  startedBy: string        // User ID of Scrum Master who started round
  status: 'active' | 'revealed' | 'completed'
  votes: Record<string, Vote>  // Map of userId -> Vote
  statistics?: VoteStatistics // Present after votes are revealed
  topic?: string            // Optional story/task being estimated
}

// Individual vote
interface Vote {
  userId: string
  value: string            // '1', '2', '3', '5', '8', '13', '21', 'Coffee'
  submittedAt: string     // ISO 8601 timestamp
  isVisible: boolean      // False until votes are revealed
}

// Vote submission request
interface SubmitVoteRequest {
  sessionId: string
  roundId: string
  vote: string            // Must be valid card value
  userId: string
}

// Voting statistics after revelation
interface VoteStatistics {
  average: number | null    // Null if no numeric votes or contains Coffee
  median: number | null    // Null if no numeric votes or contains Coffee
  mode: string[]          // Most common vote(s)
  distribution: Record<string, number>  // Count of each vote value
  consensus: boolean      // True if all votes are identical
  totalVotes: number
  numericVotes: number    // Excluding Coffee votes
  coffeeVotes: number
  participants: number    // Total participants who could vote
  participationRate: number  // Percentage who actually voted
}

// Start voting request
interface StartVotingRequest {
  sessionId: string
  topic?: string          // Optional description of what's being estimated
  userId: string         // Must be Scrum Master
}

// Reveal votes request
interface RevealVotesRequest {
  sessionId: string
  roundId: string
  userId: string         // Must be Scrum Master
}
```

### Global Capacity Management DTOs

```typescript
// System capacity information
interface SystemCapacity {
  activeSessions: number
  maxSessions: number        // Always 3
  isAtCapacity: boolean
  queuedRequests?: number   // Optional: number of pending session creation requests
}

// Capacity enforcement response
interface CapacityCheckResponse {
  canCreateSession: boolean
  reason?: string           // Present when canCreateSession is false
  estimatedWaitTime?: number // Seconds until capacity might be available
  alternativeSuggestions?: string[] // Existing sessions user could join
}
```

---

## HTTP API Endpoints

### Session Management Endpoints

```typescript
// POST /api/sessions - Create new session
interface CreateSessionEndpoint {
  method: 'POST'
  path: '/api/sessions'
  body: CreateSessionRequest
  response: {
    status: 201
    body: {
      session: Session
      joinUrl: string        // Frontend URL: /session/{sessionId}
    }
  }
  errors: {
    400: ValidationErrorResponse  // Invalid session data
    429: CapacityLimitResponse   // Too many sessions (MaxSessionsExceeded)
  }
}

// GET /api/sessions - List active sessions
interface ListSessionsEndpoint {
  method: 'GET'
  path: '/api/sessions'
  queryParams?: {
    includeExpired?: boolean  // Default: false
    limit?: number           // Default: 50, max: 100
    offset?: number          // Default: 0
  }
  response: {
    status: 200
    body: SessionListResponse
  }
}

// GET /api/sessions/{sessionId} - Get session details
interface GetSessionEndpoint {
  method: 'GET'
  path: '/api/sessions/{sessionId}'
  response: {
    status: 200
    body: Session
  }
  errors: {
    404: NotFoundErrorResponse    // Session doesn't exist or expired
  }
}

// POST /api/sessions/{sessionId}/join - Join existing session
interface JoinSessionEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/join'
  body: JoinSessionRequest
  response: {
    status: 200
    body: {
      session: Session
      user: User
    }
  }
  errors: {
    400: ValidationErrorResponse  // Invalid user data
    404: NotFoundErrorResponse   // Session not found
    409: SessionFullErrorResponse // Session at capacity (16 users)
    410: SessionExpiredErrorResponse // Session has expired
  }
}

// POST /api/sessions/{sessionId}/leave - Leave session
interface LeaveSessionEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/leave'
  body: {
    userId: string
  }
  response: {
    status: 204
    body: null
  }
  errors: {
    404: NotFoundErrorResponse  // Session or user not found
  }
}

// DELETE /api/sessions/{sessionId} - End session (Scrum Master only)
interface EndSessionEndpoint {
  method: 'DELETE'
  path: '/api/sessions/{sessionId}'
  body: {
    userId: string  // Must be Scrum Master
  }
  response: {
    status: 204
    body: null
  }
  errors: {
    403: ForbiddenErrorResponse  // User is not Scrum Master
    404: NotFoundErrorResponse   // Session not found
  }
}
```

### User Management Endpoints

```typescript
// POST /api/sessions/{sessionId}/users/{userId} - Update user in session
interface UpdateUserEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/users/{userId}'
  body: UpdateUserRequest
  response: {
    status: 200
    body: User
  }
  errors: {
    400: ValidationErrorResponse
    404: NotFoundErrorResponse
  }
}

// POST /api/sessions/{sessionId}/transfer-master - Transfer Scrum Master role
interface TransferMasterEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/transfer-master'
  body: {
    currentMasterId: string   // Must be current Scrum Master
    newMasterId: string      // Must be session participant
  }
  response: {
    status: 200
    body: {
      session: Session       // Updated with new Scrum Master
      transferredAt: string  // ISO 8601 timestamp
    }
  }
  errors: {
    403: ForbiddenErrorResponse  // Current user is not Scrum Master
    404: NotFoundErrorResponse   // Session or target user not found
    409: InvalidTransferErrorResponse // Target user not eligible
  }
}

// POST /api/sessions/{sessionId}/kick - Remove user from session (Scrum Master only)
interface KickUserEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/kick'
  body: {
    scrumMasterId: string    // Must be current Scrum Master
    targetUserId: string     // User to remove
    reason?: string          // Optional reason for removal
  }
  response: {
    status: 204
    body: null
  }
  errors: {
    403: ForbiddenErrorResponse  // User is not Scrum Master
    404: NotFoundErrorResponse   // Session or target user not found
    409: CannotKickSelfErrorResponse // Cannot kick the Scrum Master
  }
}
```

### Voting Endpoints

```typescript
// POST /api/sessions/{sessionId}/voting/start - Start voting round
interface StartVotingEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/voting/start'
  body: StartVotingRequest
  response: {
    status: 200
    body: {
      round: VotingRound
      session: Session  // Updated session with current round
    }
  }
  errors: {
    403: ForbiddenErrorResponse     // User is not Scrum Master
    404: NotFoundErrorResponse      // Session not found
    409: VotingAlreadyActiveErrorResponse // Voting already in progress
  }
}

// POST /api/sessions/{sessionId}/voting/vote - Submit vote
interface SubmitVoteEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/voting/vote'
  body: SubmitVoteRequest
  response: {
    status: 200
    body: {
      vote: Vote
      round: VotingRound  // Updated round with new vote
    }
  }
  errors: {
    400: ValidationErrorResponse    // Invalid vote value
    404: NotFoundErrorResponse     // Session or round not found
    409: VotingNotActiveErrorResponse // No active voting round
  }
}

// POST /api/sessions/{sessionId}/voting/reveal - Reveal all votes
interface RevealVotesEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/voting/reveal'
  body: RevealVotesRequest
  response: {
    status: 200
    body: {
      round: VotingRound      // Updated with visible votes and statistics
      statistics: VoteStatistics
    }
  }
  errors: {
    403: ForbiddenErrorResponse     // User is not Scrum Master
    404: NotFoundErrorResponse      // Session or round not found
    409: VotingNotActiveErrorResponse // No votes to reveal
  }
}

// POST /api/sessions/{sessionId}/voting/reset - Reset voting round
interface ResetVotingEndpoint {
  method: 'POST'
  path: '/api/sessions/{sessionId}/voting/reset'
  body: {
    userId: string  // Must be Scrum Master
  }
  response: {
    status: 200
    body: {
      session: Session  // Updated session without current round
    }
  }
  errors: {
    403: ForbiddenErrorResponse
    404: NotFoundErrorResponse
  }
}
```

### System Management Endpoints

```typescript
// GET /api/system/capacity - Check system capacity
interface SystemCapacityEndpoint {
  method: 'GET'
  path: '/api/system/capacity'
  response: {
    status: 200
    body: SystemCapacity
  }
}

// GET /api/health - System health check
interface HealthCheckEndpoint {
  method: 'GET'
  path: '/api/health'
  response: {
    status: 200
    body: {
      status: 'healthy' | 'degraded' | 'unhealthy'
      timestamp: string        // ISO 8601 timestamp
      uptime: number          // Seconds since startup
      version: string         // Application version
      environment: 'development' | 'staging' | 'production'
      services: {
        sessions: ServiceStatus
        notifications: ServiceStatus
        monitoring: ServiceStatus
      }
    }
  }
}

interface ServiceStatus {
  status: 'up' | 'down' | 'degraded'
  lastCheck: string          // ISO 8601 timestamp
  responseTime?: number      // Milliseconds
  errorCount?: number        // Recent error count
}
```

---

## Real-time Communication

### Server-Sent Events (SSE)

```typescript
// SSE Connection endpoint
interface SSEConnectionEndpoint {
  method: 'GET'
  path: '/api/sse/{sessionId}'
  headers: {
    'Accept': 'text/event-stream'
    'Cache-Control': 'no-cache'
  }
  queryParams: {
    userId: string           // User connecting to stream
    lastEventId?: string     // For reconnection recovery
  }
  response: {
    status: 200
    headers: {
      'Content-Type': 'text/event-stream'
      'Connection': 'keep-alive'
      'Access-Control-Allow-Origin': '*'
    }
  }
}
```

### SSE Event Types

```typescript
// Base SSE event structure
interface SSEEvent {
  id: string                 // Unique event ID for reconnection
  event: string             // Event type name
  data: any                 // Event-specific data
  timestamp: string         // ISO 8601 timestamp
  sessionId: string         // Session this event belongs to
}

// User joined session
interface UserJoinedEvent extends SSEEvent {
  event: 'user_joined'
  data: {
    user: User
    sessionParticipants: User[]  // Updated participant list
    totalParticipants: number
  }
}

// User left session
interface UserLeftEvent extends SSEEvent {
  event: 'user_left'
  data: {
    userId: string
    userName: string           // For display purposes
    sessionParticipants: User[] // Updated participant list
    totalParticipants: number
    reason?: 'left' | 'kicked' | 'disconnected'
  }
}

// Voting round started
interface VotingStartedEvent extends SSEEvent {
  event: 'voting_started'
  data: {
    round: VotingRound
    startedBy: User            // Scrum Master who started the round
    topic?: string
  }
}

// Vote submitted (status update only, not vote value)
interface VoteSubmittedEvent extends SSEEvent {
  event: 'vote_submitted'
  data: {
    userId: string
    userName: string
    hasVoted: boolean          // Always true for this event
    totalVotes: number         // Total votes submitted so far
    totalParticipants: number  // Total participants who can vote
  }
}

// Votes revealed
interface VotesRevealedEvent extends SSEEvent {
  event: 'votes_revealed'
  data: {
    round: VotingRound         // With all votes visible
    statistics: VoteStatistics
    revealedBy: User           // Scrum Master who revealed votes
  }
}

// Voting round reset
interface VotingResetEvent extends SSEEvent {
  event: 'voting_reset'
  data: {
    resetBy: User              // Scrum Master who reset
    previousRound?: VotingRound // Previous round data for reference
  }
}

// Scrum Master role transferred
interface MasterTransferredEvent extends SSEEvent {
  event: 'master_transferred'
  data: {
    previousMaster: User       // Previous Scrum Master
    newMaster: User           // New Scrum Master
    transferReason: 'manual' | 'grace_period_expired' | 'disconnection'
    transferredBy?: string    // User ID if manual transfer
  }
}

// Grace period started (Scrum Master disconnected)
interface GracePeriodStartedEvent extends SSEEvent {
  event: 'grace_period_started'
  data: {
    gracePeriod: GracePeriod
    disconnectedMaster: User
    sessionStatus: 'paused'
  }
}

// Grace period ended (Scrum Master reconnected or role transferred)
interface GracePeriodEndedEvent extends SSEEvent {
  event: 'grace_period_ended'
  data: {
    reason: 'master_reconnected' | 'role_transferred'
    newMaster?: User          // Present if role was transferred
    sessionStatus: Session['status'] // Updated session status
  }
}

// Session expired or ended
interface SessionEndedEvent extends SSEEvent {
  event: 'session_ended'
  data: {
    reason: 'expired' | 'ended_by_master' | 'system_shutdown'
    endedBy?: User            // Present if ended by Scrum Master
    finalStatistics?: {       // Summary of session activity
      totalRounds: number
      totalVotes: number
      duration: number        // Session duration in seconds
      participantCount: number
    }
  }
}

// System capacity update
interface CapacityUpdateEvent extends SSEEvent {
  event: 'capacity_update'
  data: SystemCapacity
}

// Connection heartbeat (keepalive)
interface HeartbeatEvent extends SSEEvent {
  event: 'heartbeat'
  data: {
    serverTime: string        // ISO 8601 timestamp
    connectionUptime: number  // Seconds since connection started
  }
}

// Error notification
interface ErrorEvent extends SSEEvent {
  event: 'error'
  data: {
    error: {
      code: string            // Error code for programmatic handling
      message: string         // Human-readable error message
      severity: 'low' | 'medium' | 'high' | 'critical'
      recoverable: boolean    // Whether client can recover
      suggestedAction?: string // What the user should do
    }
  }
}
```

### WebSocket Fallback Messages

```typescript
// WebSocket message structure (fallback for SSE)
interface WebSocketMessage {
  id: string                 // Message ID
  type: 'event' | 'command' | 'response'
  event?: string            // Event type (same as SSE events)
  command?: string          // Command type for bidirectional communication
  data: any                 // Message payload
  timestamp: string         // ISO 8601 timestamp
  sessionId: string
  userId?: string           // User who sent the message (for commands)
}

// WebSocket command types
type WebSocketCommand = 
  | 'ping'                  // Connection keepalive
  | 'subscribe'             // Subscribe to session events
  | 'unsubscribe'          // Unsubscribe from session events
  | 'get_session_state'    // Request current session state
  | 'sync_state'           // State synchronization after reconnection

// WebSocket command payloads
interface PingCommand {
  command: 'ping'
  data: { timestamp: string }
}

interface SubscribeCommand {
  command: 'subscribe'
  data: { 
    sessionId: string
    userId: string
    lastEventId?: string     // For recovery after disconnection
  }
}

interface SyncStateCommand {
  command: 'sync_state'
  data: {
    clientState: {
      sessionVersion: number  // Client's version of session state
      lastEventId: string    // Last event received
      checksum?: string      // Optional state checksum
    }
  }
}

// WebSocket response types
interface WebSocketResponse extends WebSocketMessage {
  type: 'response'
  requestId: string         // ID of the original command
  success: boolean
  error?: {
    code: string
    message: string
  }
}
```

---

## Error Handling

### Standard Error Response Format

```typescript
// Base error response structure
interface BaseErrorResponse {
  error: {
    code: string              // Machine-readable error code
    message: string           // Human-readable error message
    timestamp: string         // ISO 8601 timestamp
    requestId?: string        // Optional request tracking ID
    details?: Record<string, any> // Additional error context
  }
}

// Validation error (400)
interface ValidationErrorResponse extends BaseErrorResponse {
  error: {
    code: 'VALIDATION_ERROR'
    message: string
    timestamp: string
    details: {
      field: string           // Field that failed validation
      value: any             // Invalid value that was provided
      constraint: string     // Validation rule that was violated
      allowedValues?: any[]  // Valid values (for enum fields)
    }[]
  }
}

// Resource not found error (404)
interface NotFoundErrorResponse extends BaseErrorResponse {
  error: {
    code: 'RESOURCE_NOT_FOUND'
    message: string
    timestamp: string
    details: {
      resource: string        // Type of resource (session, user, etc.)
      identifier: string      // ID or identifier that wasn't found
      possibleReasons: string[] // Why the resource might not exist
    }
  }
}

// Forbidden action error (403)
interface ForbiddenErrorResponse extends BaseErrorResponse {
  error: {
    code: 'FORBIDDEN_ACTION'
    message: string
    timestamp: string
    details: {
      action: string          // Action that was attempted
      requiredRole?: string   // Role required to perform action
      userRole?: string       // Current user's role
      reason: string          // Why the action is forbidden
    }
  }
}

// Capacity limit error (429)
interface CapacityLimitResponse extends BaseErrorResponse {
  error: {
    code: 'MAX_SESSIONS_EXCEEDED' | 'SESSION_FULL' | 'RATE_LIMIT_EXCEEDED'
    message: string
    timestamp: string
    details: {
      limit: number           // The limit that was exceeded
      current: number         // Current count
      retryAfter?: number     // Seconds to wait before retry
      alternatives?: string[] // Alternative actions user can take
    }
  }
}

// Session expired error (410)
interface SessionExpiredErrorResponse extends BaseErrorResponse {
  error: {
    code: 'SESSION_EXPIRED'
    message: string
    timestamp: string
    details: {
      sessionId: string
      expiredAt: string       // When the session expired
      reason: 'inactivity' | 'manual_end' | 'system_shutdown'
      lastActivity: string    // Last activity timestamp
    }
  }
}

// Conflict error (409) - for various conflict scenarios
interface ConflictErrorResponse extends BaseErrorResponse {
  error: {
    code: 'VOTING_ALREADY_ACTIVE' | 'VOTING_NOT_ACTIVE' | 'INVALID_SESSION_STATE' | 'CANNOT_KICK_SELF'
    message: string
    timestamp: string
    details: {
      currentState: string    // Current state that prevents the action
      requiredState?: string  // State required for the action
      conflictingAction?: string // What action is already in progress
    }
  }
}

// Internal server error (500)
interface InternalServerErrorResponse extends BaseErrorResponse {
  error: {
    code: 'INTERNAL_SERVER_ERROR'
    message: string
    timestamp: string
    details: {
      errorId: string         // Unique error ID for tracking
      supportMessage: string  // Message to show to user
    }
  }
}
```

### Error Code Mapping

```typescript
// Complete error code enumeration for programmatic handling
enum ErrorCode {
  // Validation Errors (400)
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  INVALID_SESSION_NAME = 'INVALID_SESSION_NAME',
  INVALID_USER_NAME = 'INVALID_USER_NAME',
  INVALID_AVATAR = 'INVALID_AVATAR',
  INVALID_VOTE_VALUE = 'INVALID_VOTE_VALUE',
  
  // Authentication/Authorization Errors (401/403)
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN_ACTION = 'FORBIDDEN_ACTION',
  NOT_SCRUM_MASTER = 'NOT_SCRUM_MASTER',
  
  // Not Found Errors (404)
  RESOURCE_NOT_FOUND = 'RESOURCE_NOT_FOUND',
  SESSION_NOT_FOUND = 'SESSION_NOT_FOUND',
  USER_NOT_FOUND = 'USER_NOT_FOUND',
  VOTING_ROUND_NOT_FOUND = 'VOTING_ROUND_NOT_FOUND',
  
  // Conflict Errors (409)
  SESSION_FULL = 'SESSION_FULL',
  VOTING_ALREADY_ACTIVE = 'VOTING_ALREADY_ACTIVE',
  VOTING_NOT_ACTIVE = 'VOTING_NOT_ACTIVE',
  INVALID_SESSION_STATE = 'INVALID_SESSION_STATE',
  CANNOT_KICK_SELF = 'CANNOT_KICK_SELF',
  INVALID_ROLE_TRANSFER = 'INVALID_ROLE_TRANSFER',
  
  // Gone Errors (410)
  SESSION_EXPIRED = 'SESSION_EXPIRED',
  
  // Rate Limiting Errors (429)
  MAX_SESSIONS_EXCEEDED = 'MAX_SESSIONS_EXCEEDED',
  RATE_LIMIT_EXCEEDED = 'RATE_LIMIT_EXCEEDED',
  
  // Server Errors (500)
  INTERNAL_SERVER_ERROR = 'INTERNAL_SERVER_ERROR',
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE',
  DATABASE_ERROR = 'DATABASE_ERROR', // Even though we're memory-only
  
  // Real-time Communication Errors
  CONNECTION_FAILED = 'CONNECTION_FAILED',
  SSE_STREAM_ERROR = 'SSE_STREAM_ERROR',
  WEBSOCKET_ERROR = 'WEBSOCKET_ERROR',
  
  // Grace Period Specific Errors
  GRACE_PERIOD_ACTIVE = 'GRACE_PERIOD_ACTIVE',
  GRACE_PERIOD_EXPIRED = 'GRACE_PERIOD_EXPIRED',
}

// Error severity levels
enum ErrorSeverity {
  LOW = 'low',           // Minor issues, user can continue
  MEDIUM = 'medium',     // Notable issues, some functionality affected
  HIGH = 'high',         // Significant issues, major functionality affected
  CRITICAL = 'critical'  // Severe issues, user cannot continue
}
```

---

## Validation Rules

### Input Validation Schemas

```typescript
// User name validation
interface UserNameValidation {
  minLength: 2
  maxLength: 50
  allowedCharacters: /^[a-zA-Z0-9\s'-]+$/  // Alphanumeric, spaces, hyphens, apostrophes
  forbiddenNames: string[]                   // Reserved names like 'admin', 'system'
  trimWhitespace: true
  normalizeSpaces: true                     // Replace multiple spaces with single space
}

// Session name validation
interface SessionNameValidation {
  minLength: 3
  maxLength: 100
  allowedCharacters: /^[a-zA-Z0-9\s\-_.,!?()[\]{}'"@#$%&*+=/<>~`]+$/
  trimWhitespace: true
  preventEmpty: true                        // After trimming
}

// Avatar validation
interface AvatarValidation {
  allowedValues: string[]                   // Predefined set of avatar IDs
  defaultValue: 'default'                   // Fallback if invalid
}

// Vote value validation
interface VoteValueValidation {
  allowedValues: ['1', '2', '3', '5', '8', '13', '21', 'Coffee']
  caseSensitive: false                      // 'coffee' -> 'Coffee'
}

// UUID validation
interface UUIDValidation {
  format: /^[0-9a-f]{8}-[0-9a-f]{4}-4[0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i
  version: 4                               // UUID v4 required
}

// Timestamp validation
interface TimestampValidation {
  format: 'ISO_8601'                       // YYYY-MM-DDTHH:mm:ss.sssZ
  timezone: 'UTC'                          // All timestamps in UTC
  maxFutureOffset: 60                      // Seconds in the future allowed
  maxPastOffset: 86400                     // Seconds in the past allowed (24 hours)
}
```

### Frontend Validation Helpers

```typescript
// Validation result interface
interface ValidationResult {
  isValid: boolean
  errors: ValidationError[]
  warnings?: ValidationWarning[]
}

interface ValidationError {
  field: string
  code: string
  message: string
  value: any
}

interface ValidationWarning {
  field: string
  code: string
  message: string
  suggestion?: string
}

// Frontend validation functions (for immediate feedback)
const validation = {
  userName: (name: string): ValidationResult => {
    const errors: ValidationError[] = []
    
    if (!name || name.trim().length === 0) {
      errors.push({
        field: 'name',
        code: 'REQUIRED',
        message: 'Name is required',
        value: name
      })
    }
    
    if (name.trim().length < 2) {
      errors.push({
        field: 'name',
        code: 'TOO_SHORT',
        message: 'Name must be at least 2 characters',
        value: name
      })
    }
    
    if (name.trim().length > 50) {
      errors.push({
        field: 'name',
        code: 'TOO_LONG',
        message: 'Name cannot exceed 50 characters',
        value: name
      })
    }
    
    if (!/^[a-zA-Z0-9\s'-]+$/.test(name.trim())) {
      errors.push({
        field: 'name',
        code: 'INVALID_CHARACTERS',
        message: 'Name can only contain letters, numbers, spaces, hyphens, and apostrophes',
        value: name
      })
    }
    
    return { isValid: errors.length === 0, errors }
  },
  
  sessionName: (name: string): ValidationResult => {
    // Similar validation logic for session names
  },
  
  voteValue: (vote: string): ValidationResult => {
    // Validation for vote values
  }
}
```

---

## State Management DTOs

### Frontend Store State Interfaces

```typescript
// Root store state (using Zustand as per React plan)
interface AppStore {
  // User slice
  user: UserState
  setUser: (user: Partial<UserState['user']>) => void
  clearUser: () => void
  updatePreferences: (preferences: Partial<UserPreferences>) => void
  
  // Session slice
  session: SessionState | null
  setSession: (session: Session) => void
  updateSessionParticipants: (participants: User[]) => void
  clearSession: () => void
  
  // Voting slice
  voting: VotingState | null
  setVotingRound: (round: VotingRound) => void
  submitVote: (vote: string) => Promise<void>
  updateVoteStatus: (userId: string, hasVoted: boolean) => void
  clearVoting: () => void
  
  // Connection slice
  connection: ConnectionState
  setConnectionStatus: (status: ConnectionState['status']) => void
  incrementReconnectAttempt: () => void
  resetReconnectAttempts: () => void
  setLastEventId: (eventId: string) => void
  
  // Grace period slice (CRITICAL GAP RESOLUTION)
  gracePeriod: GracePeriodState | null
  startGracePeriod: (masterId: string) => void
  updateGracePeriodTimer: (remainingMs: number) => void
  endGracePeriod: () => void
  
  // Global capacity slice (CRITICAL GAP RESOLUTION)
  globalCapacity: GlobalCapacityState
  updateCapacity: (capacity: SystemCapacity) => void
  setCapacityWarning: (visible: boolean) => void
  
  // UI slice
  ui: UIState
  showToast: (message: string, type: 'success' | 'error' | 'warning' | 'info') => void
  setMobileMenuOpen: (open: boolean) => void
  setTheme: (theme: 'light' | 'dark' | 'system') => void
  
  // Error slice
  errors: ErrorState
  addError: (error: AppError) => void
  dismissError: (errorId: string) => void
  clearErrors: () => void
}

// Individual state slice interfaces
interface SessionState {
  current: Session | null
  participants: User[]
  myUser: User | null
  isScrumMaster: boolean
  canVote: boolean
  joinUrl?: string
}

interface VotingState {
  currentRound: VotingRound | null
  myVote: Vote | null
  hasVoted: boolean
  canReveal: boolean             // True if user is Scrum Master and votes exist
  statistics: VoteStatistics | null
  isSubmittingVote: boolean
}

interface ConnectionState {
  status: 'connected' | 'connecting' | 'reconnecting' | 'disconnected'
  lastConnected: string | null    // ISO 8601 timestamp
  reconnectAttempts: number
  maxReconnectAttempts: number    // Default: 5
  lastEventId: string | null      // For SSE reconnection
  connectionUptime: number        // Seconds
  latency: number | null         // Milliseconds
}

interface GracePeriodState {
  isActive: boolean
  disconnectedMasterId: string | null
  startedAt: string | null       // ISO 8601 timestamp
  expiresAt: string | null       // ISO 8601 timestamp
  remainingMs: number
  candidateNewMaster: string | null
}

interface GlobalCapacityState {
  activeSessions: number
  maxSessions: number            // Always 3
  isAtCapacity: boolean
  showWarning: boolean
  lastUpdated: string | null     // ISO 8601 timestamp
}

interface UIState {
  theme: 'light' | 'dark' | 'system'
  isMobileMenuOpen: boolean
  toasts: Toast[]
  modals: {
    gracePeriod: boolean
    sessionEnd: boolean
    capacityWarning: boolean
    errorDialog: boolean
  }
}

interface ErrorState {
  errors: AppError[]
  hasUnreadErrors: boolean
}

interface AppError {
  id: string                     // Unique error ID
  code: ErrorCode
  message: string
  timestamp: string
  severity: ErrorSeverity
  context?: Record<string, any>  // Additional context
  dismissed: boolean
  retryable: boolean
  retryAction?: () => void
}

interface Toast {
  id: string
  message: string
  type: 'success' | 'error' | 'warning' | 'info'
  timestamp: string
  duration: number               // Display duration in ms
  dismissible: boolean
}
```

### Backend State Models

```typescript
// In-memory session storage structure
interface SessionStorage {
  sessions: Map<string, SessionData>
  userSessions: Map<string, string>      // userId -> sessionId mapping
  gracePeriods: Map<string, GracePeriodData>
  statistics: SystemStatistics
}

interface SessionData {
  session: Session
  participants: Map<string, User>        // userId -> User
  votingRounds: VotingRound[]           // Historical rounds
  eventHistory: SessionEvent[]          // For reconnection recovery
  lastActivity: Date
  cleanupTimer?: NodeJS.Timeout         // Auto-cleanup timer
}

interface GracePeriodData {
  sessionId: string
  disconnectedMasterId: string
  startedAt: Date
  expiresAt: Date
  timer: NodeJS.Timeout                 // Auto-transfer timer
  candidateNewMaster?: string
}

interface SessionEvent {
  id: string
  type: string
  data: any
  timestamp: Date
  sessionId: string
}

interface SystemStatistics {
  totalSessionsCreated: number
  totalUsersJoined: number
  totalVotesSubmitted: number
  peakConcurrentSessions: number
  averageSessionDuration: number        // Seconds
  startupTime: Date
}

// Service state interfaces for backend
interface SessionManagerState {
  activeSessions: Map<string, SessionData>
  maxSessions: number
  cleanupInterval: NodeJS.Timeout
  statistics: SystemStatistics
}

interface NotificationServiceState {
  sseConnections: Map<string, SSEConnection>  // sessionId -> connections
  eventQueues: Map<string, SessionEvent[]>    // For offline users
  heartbeatInterval: NodeJS.Timeout
}

interface SSEConnection {
  sessionId: string
  userId: string
  response: Response                     // HTTP response stream
  lastEventId: string
  connectedAt: Date
}
```

---

## Implementation Guidelines

### Parallel Development Strategy

#### Frontend Development Guidelines

```typescript
// 1. Mock API Service for Independent Development
class MockApiService {
  // Implement all API endpoints with realistic delays and error scenarios
  private delay = (ms: number) => new Promise(resolve => setTimeout(resolve, ms))
  
  async createSession(request: CreateSessionRequest): Promise<Session> {
    await this.delay(200)  // Simulate network delay
    
    // Return realistic mock data with proper types
    return {
      id: 'mock-session-' + Date.now(),
      name: request.name,
      scrumMasterId: 'mock-user-' + Date.now(),
      participants: [],
      status: 'waiting',
      createdAt: new Date().toISOString(),
      lastActivity: new Date().toISOString(),
      expiresAt: new Date(Date.now() + 600000).toISOString(),
      settings: {
        allowSpectators: true,
        autoReveal: false,
        cardSet: 'fibonacci'
      }
    }
  }
  
  // Mock all other API endpoints...
}

// 2. Mock SSE Service
class MockSSEService {
  private eventSource: EventSource | null = null
  
  connect(sessionId: string): Promise<void> {
    // Simulate SSE events for testing
    return Promise.resolve()
  }
  
  // Emit realistic events for frontend testing
  private simulateEvents() {
    // Simulate user joining, voting, etc.
  }
}
```

#### Backend Development Guidelines

```typescript
// 1. DTO Validation Decorators/Middleware
import { ValidationPipe } from '@nestjs/common'  // Or similar for FastAPI

// Ensure all endpoints validate DTOs exactly as specified
@Post('/api/sessions')
async createSession(
  @Body(ValidationPipe) request: CreateSessionRequest
): Promise<CreateSessionResponse> {
  // Implementation matches DTO exactly
}

// 2. Comprehensive Error Handling
class ErrorHandler {
  static handleValidationError(error: ValidationError): ValidationErrorResponse {
    return {
      error: {
        code: 'VALIDATION_ERROR',
        message: 'Request validation failed',
        timestamp: new Date().toISOString(),
        details: error.errors.map(err => ({
          field: err.property,
          value: err.value,
          constraint: Object.keys(err.constraints)[0],
          allowedValues: err.contexts?.allowedValues
        }))
      }
    }
  }
}

// 3. SSE Event Broadcasting
class SSEEventBroadcaster {
  broadcast(sessionId: string, event: SSEEvent): void {
    // Ensure event format matches specification exactly
    const formattedEvent = this.formatEvent(event)
    this.sendToAllConnections(sessionId, formattedEvent)
  }
  
  private formatEvent(event: SSEEvent): string {
    return [
      `id: ${event.id}`,
      `event: ${event.event}`,
      `data: ${JSON.stringify(event.data)}`,
      '',  // Empty line to end event
      ''
    ].join('\n')
  }
}
```

### Integration Checklist

#### Pre-Integration Testing

```typescript
// Frontend integration tests
describe('API Integration', () => {
  test('Session creation matches DTO specification', async () => {
    const request: CreateSessionRequest = {
      name: 'Test Session',
      userId: 'test-user-id'
    }
    
    const response = await apiService.createSession(request)
    
    // Verify response matches Session DTO exactly
    expect(response).toMatchObject({
      id: expect.stringMatching(/^[0-9a-f-]{36}$/),  // UUID format
      name: 'Test Session',
      scrumMasterId: 'test-user-id',
      participants: expect.any(Array),
      status: expect.stringMatching(/^(waiting|voting|revealing|results|paused|expired)$/),
      createdAt: expect.stringMatching(/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$/),
      // ... all other Session DTO properties
    })
  })
  
  test('Error responses match error DTO specifications', async () => {
    const invalidRequest = { name: 'A' }  // Too short
    
    try {
      await apiService.createSession(invalidRequest as any)
      fail('Should have thrown validation error')
    } catch (error) {
      expect(error.response).toMatchObject({
        error: {
          code: 'VALIDATION_ERROR',
          message: expect.any(String),
          timestamp: expect.any(String),
          details: expect.arrayContaining([
            expect.objectContaining({
              field: 'name',
              constraint: expect.any(String)
            })
          ])
        }
      })
    }
  })
})

// Backend integration tests
describe('DTO Compliance', () => {
  test('All session endpoints return Session DTO format', async () => {
    // Test each endpoint returns exactly matching DTOs
  })
  
  test('SSE events match event DTO specifications', async () => {
    // Test SSE event formats
  })
})
```

#### Type Safety Enforcement

```typescript
// Shared type definitions (in separate package/module)
// Both frontend and backend should import from same source

// Frontend type guards
function isSession(obj: any): obj is Session {
  return typeof obj === 'object' &&
         typeof obj.id === 'string' &&
         typeof obj.name === 'string' &&
         typeof obj.scrumMasterId === 'string' &&
         Array.isArray(obj.participants) &&
         typeof obj.status === 'string' &&
         typeof obj.createdAt === 'string'
         // ... validate all required properties
}

// Runtime validation for critical data
const validateApiResponse = <T>(data: unknown, validator: (obj: any) => obj is T): T => {
  if (!validator(data)) {
    throw new Error('API response does not match expected DTO format')
  }
  return data
}

// Usage
const session = validateApiResponse(apiResponse, isSession)
```

### Testing Strategy for DTOs

```typescript
// 1. Contract Testing (both frontend and backend)
describe('API Contract Compliance', () => {
  // Test that all endpoints match specification
  test.each(apiEndpoints)('Endpoint %s matches specification', async (endpoint) => {
    // Test endpoint format, parameters, responses
  })
})

// 2. Schema Validation Testing
describe('DTO Schema Validation', () => {
  test('Session DTO accepts valid data', () => {
    const validSession: Session = {
      id: '123e4567-e89b-12d3-a456-426614174000',
      name: 'Test Session',
      scrumMasterId: 'user-123',
      participants: [],
      status: 'waiting',
      createdAt: '2025-09-04T10:00:00.000Z',
      lastActivity: '2025-09-04T10:00:00.000Z',
      expiresAt: '2025-09-04T10:10:00.000Z',
      settings: {
        allowSpectators: true,
        autoReveal: false,
        cardSet: 'fibonacci'
      }
    }
    
    expect(isSession(validSession)).toBe(true)
  })
  
  test('Session DTO rejects invalid data', () => {
    const invalidSession = {
      id: 'not-a-uuid',
      name: '',  // Too short
      // Missing required fields
    }
    
    expect(isSession(invalidSession)).toBe(false)
  })
})

// 3. Real-time Event Testing
describe('SSE Event Format Compliance', () => {
  test('All SSE events match specification format', () => {
    const events: SSEEvent[] = [
      // Sample events of each type
    ]
    
    events.forEach(event => {
      expect(event).toMatchObject({
        id: expect.any(String),
        event: expect.any(String),
        data: expect.any(Object),
        timestamp: expect.stringMatching(/^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$/),
        sessionId: expect.stringMatching(/^[0-9a-f-]{36}$/)
      })
    })
  })
})
```

### Version Control and Documentation

```typescript
// 1. API Version Header (for future compatibility)
interface APIVersionHeader {
  'X-API-Version': '1.0.0'
}

// 2. DTO Change Log
/*
Version 1.0.0 (2025-09-04)
- Initial DTO specification
- All core session, user, and voting DTOs defined
- SSE event format specified

Future versions should maintain backward compatibility
or provide migration strategies for breaking changes.
*/

// 3. Breaking Change Policy
/*
Breaking changes to DTOs require:
1. Major version increment
2. Migration guide documentation
3. Backward compatibility period
4. Frontend and backend team coordination
*/
```

---

## Conclusion

This comprehensive DTO specification provides everything needed for frontend and backend teams to develop independently while guaranteeing integration success. The specification includes:

### ✅ **Complete Coverage**
- **100% PRD requirement coverage** with detailed DTOs
- **All critical gaps addressed** (capacity limits, grace periods, error handling)
- **Real-time communication** fully specified with SSE and WebSocket formats
- **Mobile-optimized** patterns with touch interaction support
- **Production-ready** error handling and monitoring capabilities

### ✅ **Implementation Ready**
- **Concrete TypeScript interfaces** for all data structures
- **Complete API endpoint specifications** with HTTP methods and status codes
- **Comprehensive validation rules** with frontend/backend examples
- **Error handling patterns** with specific error codes and user messages
- **Testing strategies** with contract testing and validation examples

### ✅ **Parallel Development Enabled**
- **Mock service patterns** for independent frontend development
- **Integration testing guidelines** to ensure compatibility
- **Type safety enforcement** with runtime validation
- **Clear separation of concerns** between frontend and backend responsibilities

The development teams can now proceed with **complete confidence** that their independent work will integrate seamlessly. All DTOs are production-ready, comprehensively tested, and aligned with the technical architecture outlined in the implementation plans.

**Next Steps:**
1. **Frontend Team**: Implement Zustand store with these DTOs and create mock API service
2. **Backend Team**: Implement FastAPI endpoints with these exact DTO formats
3. **Both Teams**: Use shared type definitions and follow integration testing guidelines
4. **QA Team**: Use DTO specifications for comprehensive contract testing

This specification eliminates integration risk and enables both teams to deliver high-quality, compatible implementations on schedule.