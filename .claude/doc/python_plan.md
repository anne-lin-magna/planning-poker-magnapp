# Python/FastAPI Implementation Plan - MagnaPP Planning Poker Backend

## Executive Summary

This document provides a comprehensive implementation plan for the MagnaPP Planning Poker backend using Python 3.11+ and FastAPI. The backend serves as the real-time coordination engine for collaborative planning poker sessions, supporting up to 3 concurrent sessions with 16 users each through Server-Sent Events (SSE) and RESTful API endpoints.

The architecture prioritizes simplicity, performance, and real-time synchronization while maintaining strict session constraints and automatic cleanup mechanisms. All data is stored in-memory with TTL management, and the system provides sub-second real-time updates across all connected clients.

## File Structure and Organization

```
backend/
├── src/
│   ├── main.py                    # FastAPI application entry point
│   ├── models/                    # Pydantic data models
│   │   ├── __init__.py
│   │   ├── session.py             # Session-related models
│   │   ├── user.py                # User-related models
│   │   ├── voting.py              # Voting-related models
│   │   └── events.py              # SSE event models
│   ├── api/                       # API route handlers
│   │   ├── __init__.py
│   │   ├── sessions.py            # Session management endpoints
│   │   ├── voting.py              # Voting endpoints
│   │   ├── users.py               # User management endpoints
│   │   └── sse.py                 # Server-Sent Events endpoint
│   ├── services/                  # Business logic services
│   │   ├── __init__.py
│   │   ├── session_manager.py     # Core session management
│   │   ├── voting_manager.py      # Voting logic and statistics
│   │   ├── user_manager.py        # User management within sessions
│   │   ├── sse_manager.py         # SSE connection and broadcasting
│   │   └── cleanup_service.py     # Background cleanup tasks
│   ├── core/                      # Core utilities and configuration
│   │   ├── __init__.py
│   │   ├── config.py              # Application configuration
│   │   ├── exceptions.py          # Custom exception classes
│   │   ├── dependencies.py        # Dependency injection
│   │   └── constants.py           # Application constants
│   └── utils/                     # Utility functions
│       ├── __init__.py
│       ├── guid_generator.py      # GUID generation utilities
│       ├── statistics.py          # Voting statistics calculations
│       └── validators.py          # Input validation helpers
├── tests/                         # Test files
│   ├── __init__.py
│   ├── conftest.py                # Pytest fixtures
│   ├── test_models/               # Model tests
│   ├── test_api/                  # API endpoint tests
│   ├── test_services/             # Service layer tests
│   └── test_integration/          # Integration tests
├── requirements.txt               # Production dependencies
├── requirements-dev.txt           # Development dependencies
├── pyproject.toml                # Project configuration
├── .env.example                  # Environment variables example
└── README.md                     # Backend documentation
```

## Data Models (Pydantic)

### Core Models Definition

#### User Models (`src/models/user.py`)
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Optional

class User(BaseModel):
    """Core user model for session participants"""
    id: str = Field(..., description="Unique user identifier (GUID)")
    name: str = Field(..., min_length=1, max_length=50, description="Display name")
    avatar: str = Field(..., description="Avatar identifier or URL")
    is_online: bool = Field(default=True, description="Connection status")
    joined_at: datetime = Field(default_factory=datetime.utcnow, description="Join timestamp")
    
class CreateUserRequest(BaseModel):
    """Request model for user creation"""
    name: str = Field(..., min_length=1, max_length=50)
    avatar: str = Field(...)
    
class UserResponse(BaseModel):
    """Response model for user data"""
    id: str
    name: str
    avatar: str
    is_online: bool
    joined_at: str
```

#### Session Models (`src/models/session.py`)
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import List, Optional, Literal
from .user import User
from .voting import VotingRound

SessionStatus = Literal['waiting', 'voting', 'revealing', 'results', 'paused']

class Session(BaseModel):
    """Core session model"""
    id: str = Field(..., description="Session GUID")
    name: str = Field(..., min_length=1, max_length=100)
    participants: List[User] = Field(default_factory=list)
    scrum_master_id: str = Field(..., description="User ID of session leader")
    status: SessionStatus = Field(default='waiting')
    current_round: Optional[VotingRound] = Field(default=None)
    expires_at: datetime = Field(..., description="Session expiry timestamp")
    created_at: datetime = Field(default_factory=datetime.utcnow)
    last_activity: datetime = Field(default_factory=datetime.utcnow)
    
class CreateSessionRequest(BaseModel):
    """Request model for session creation"""
    name: str = Field(..., min_length=1, max_length=100)
    creator_name: str = Field(..., min_length=1, max_length=50)
    creator_avatar: str = Field(...)
    
class SessionSummary(BaseModel):
    """Lightweight session info for listing"""
    id: str
    name: str
    participant_count: int
    scrum_master_name: str
    status: SessionStatus
    created_at: str
    
class JoinSessionRequest(BaseModel):
    """Request model for joining session"""
    user_name: str = Field(..., min_length=1, max_length=50)
    user_avatar: str = Field(...)
```

#### Voting Models (`src/models/voting.py`)
```python
from pydantic import BaseModel, Field
from datetime import datetime
from typing import Dict, Optional, List
from enum import Enum

class VoteValue(str, Enum):
    """Valid voting values"""
    ONE = "1"
    TWO = "2"
    THREE = "3"
    FIVE = "5"
    EIGHT = "8"
    THIRTEEN = "13"
    TWENTY_ONE = "21"
    COFFEE = "coffee"

class Vote(BaseModel):
    """Individual vote model"""
    user_id: str
    value: VoteValue
    submitted_at: datetime = Field(default_factory=datetime.utcnow)

class VotingStatistics(BaseModel):
    """Voting round statistics"""
    average: Optional[float] = Field(None, description="Average of numeric votes")
    distribution: Dict[str, int] = Field(..., description="Vote value counts")
    consensus: bool = Field(..., description="All numeric votes are identical")
    coffee_votes: int = Field(default=0, description="Number of coffee votes")
    total_votes: int = Field(..., description="Total number of votes")

class VotingRound(BaseModel):
    """Complete voting round model"""
    id: str = Field(..., description="Round identifier")
    votes: Dict[str, Vote] = Field(default_factory=dict)  # user_id -> Vote
    is_revealed: bool = Field(default=False)
    statistics: Optional[VotingStatistics] = Field(default=None)
    started_at: datetime = Field(default_factory=datetime.utcnow)
    
class SubmitVoteRequest(BaseModel):
    """Request model for vote submission"""
    vote: VoteValue
    
class VoteResponse(BaseModel):
    """Response model for vote operations"""
    success: bool
    message: str
```

#### SSE Event Models (`src/models/events.py`)
```python
from pydantic import BaseModel, Field
from typing import Union, Any, Literal
from datetime import datetime

EventType = Literal[
    'session_updated', 'user_joined', 'user_left', 'user_disconnected',
    'voting_started', 'vote_submitted', 'votes_revealed', 'round_completed',
    'session_expired', 'scrum_master_changed', 'session_paused'
]

class SSEEvent(BaseModel):
    """Base SSE event model"""
    type: EventType
    session_id: str
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    data: Any = Field(...)

class UserJoinedEvent(SSEEvent):
    """User joined session event"""
    type: Literal['user_joined'] = 'user_joined'
    data: dict  # User data

class VoteSubmittedEvent(SSEEvent):
    """Vote submitted event (without revealing vote value)"""
    type: Literal['vote_submitted'] = 'vote_submitted'
    data: dict  # user_id only

class VotesRevealedEvent(SSEEvent):
    """Votes revealed event"""
    type: Literal['votes_revealed'] = 'votes_revealed'
    data: dict  # Full voting results and statistics
```

## API Endpoints Design

### Session Management Endpoints (`src/api/sessions.py`)

```python
from fastapi import APIRouter, HTTPException, Depends
from typing import List
from ..services.session_manager import SessionManager
from ..models.session import *
from ..core.dependencies import get_session_manager

router = APIRouter(prefix="/api/sessions", tags=["sessions"])

@router.post("/", response_model=dict)
async def create_session(
    request: CreateSessionRequest,
    session_manager: SessionManager = Depends(get_session_manager)
) -> dict:
    """Create new planning poker session
    
    Returns: {"session_id": str, "join_url": str}
    Raises: 409 if maximum sessions reached
    """

@router.get("/", response_model=List[SessionSummary])
async def list_sessions(
    session_manager: SessionManager = Depends(get_session_manager)
) -> List[SessionSummary]:
    """List all active sessions available for joining"""

@router.get("/{session_id}", response_model=Session)
async def get_session(
    session_id: str,
    session_manager: SessionManager = Depends(get_session_manager)
) -> Session:
    """Get detailed session information
    
    Raises: 404 if session not found or expired
    """

@router.post("/{session_id}/join", response_model=dict)
async def join_session(
    session_id: str,
    request: JoinSessionRequest,
    session_manager: SessionManager = Depends(get_session_manager)
) -> dict:
    """Join existing session as participant
    
    Returns: {"user_id": str, "session": Session}
    Raises: 404 if session not found, 409 if session full
    """

@router.post("/{session_id}/leave", response_model=dict)
async def leave_session(
    session_id: str,
    user_id: str,
    session_manager: SessionManager = Depends(get_session_manager)
) -> dict:
    """Leave session (remove user)
    
    Returns: {"success": bool, "message": str}
    """

@router.delete("/{session_id}", response_model=dict)
async def end_session(
    session_id: str,
    user_id: str,
    session_manager: SessionManager = Depends(get_session_manager)
) -> dict:
    """End session (Scrum Master only)
    
    Returns: {"success": bool, "message": str}
    Raises: 403 if not Scrum Master
    """
```

### Voting Endpoints (`src/api/voting.py`)

```python
@router.post("/{session_id}/start", response_model=VoteResponse)
async def start_voting_round(
    session_id: str,
    user_id: str,
    voting_manager: VotingManager = Depends(get_voting_manager)
) -> VoteResponse:
    """Start new voting round (Scrum Master only)
    
    Creates new round, resets all votes, broadcasts start event
    """

@router.post("/{session_id}/vote", response_model=VoteResponse)
async def submit_vote(
    session_id: str,
    user_id: str,
    request: SubmitVoteRequest,
    voting_manager: VotingManager = Depends(get_voting_manager)
) -> VoteResponse:
    """Submit vote for current round
    
    Accepts vote, updates session state, broadcasts vote submitted event
    """

@router.post("/{session_id}/reveal", response_model=dict)
async def reveal_votes(
    session_id: str,
    user_id: str,
    voting_manager: VotingManager = Depends(get_voting_manager)
) -> dict:
    """Reveal all votes and show statistics (Scrum Master only)
    
    Calculates statistics, broadcasts results to all participants
    """

@router.get("/{session_id}/round", response_model=Optional[VotingRound])
async def get_current_round(
    session_id: str,
    voting_manager: VotingManager = Depends(get_voting_manager)
) -> Optional[VotingRound]:
    """Get current voting round state"""
```

### User Management Endpoints (`src/api/users.py`)

```python
@router.post("/{session_id}/kick/{user_id}", response_model=dict)
async def kick_user(
    session_id: str,
    user_id: str,
    kicker_id: str,
    user_manager: UserManager = Depends(get_user_manager)
) -> dict:
    """Remove user from session (Scrum Master only)"""

@router.post("/{session_id}/transfer", response_model=dict)
async def transfer_scrum_master(
    session_id: str,
    new_master_id: str,
    current_master_id: str,
    user_manager: UserManager = Depends(get_user_manager)
) -> dict:
    """Transfer Scrum Master role to another participant"""

@router.post("/{session_id}/reconnect", response_model=dict)
async def reconnect_user(
    session_id: str,
    user_id: str,
    user_manager: UserManager = Depends(get_user_manager)
) -> dict:
    """Reconnect user after disconnection"""
```

### Server-Sent Events Endpoint (`src/api/sse.py`)

```python
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
import json
import asyncio

@router.get("/{session_id}")
async def stream_session_events(
    session_id: str,
    sse_manager: SSEManager = Depends(get_sse_manager)
):
    """Server-Sent Events stream for session updates
    
    Provides real-time updates for:
    - User join/leave events
    - Voting state changes
    - Session status updates
    - Scrum Master changes
    """
    
    async def event_generator():
        # Register client connection
        client_id = await sse_manager.register_client(session_id)
        
        try:
            while True:
                # Get pending events for this session
                events = await sse_manager.get_pending_events(session_id, client_id)
                
                for event in events:
                    yield f"data: {json.dumps(event.dict())}\n\n"
                
                await asyncio.sleep(0.1)  # 100ms polling interval
                
        except asyncio.CancelledError:
            # Client disconnected
            await sse_manager.unregister_client(session_id, client_id)
            raise
    
    return StreamingResponse(
        event_generator(),
        media_type="text/plain",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "Access-Control-Allow-Origin": "*",
        }
    )
```

## Service Layer Implementation

### Session Manager (`src/services/session_manager.py`)

```python
from typing import Dict, List, Optional
from datetime import datetime, timedelta
import asyncio
from ..models.session import Session, SessionSummary, CreateSessionRequest
from ..models.user import User
from ..utils.guid_generator import generate_guid
from ..core.constants import MAX_SESSIONS, SESSION_TIMEOUT_MINUTES, MAX_USERS_PER_SESSION

class SessionManager:
    """Core session management service"""
    
    def __init__(self):
        self._sessions: Dict[str, Session] = {}
        self._user_sessions: Dict[str, str] = {}  # user_id -> session_id
        self._lock = asyncio.Lock()
    
    async def create_session(self, request: CreateSessionRequest) -> tuple[str, Session]:
        """Create new planning poker session
        
        Returns: (session_id, session_object)
        Raises: RuntimeError if max sessions reached
        """
        async with self._lock:
            if len(self._sessions) >= MAX_SESSIONS:
                raise RuntimeError("Maximum number of concurrent sessions reached")
            
            session_id = generate_guid()
            user_id = generate_guid()
            
            # Create Scrum Master user
            scrum_master = User(
                id=user_id,
                name=request.creator_name,
                avatar=request.creator_avatar
            )
            
            # Create session
            session = Session(
                id=session_id,
                name=request.name,
                participants=[scrum_master],
                scrum_master_id=user_id,
                expires_at=datetime.utcnow() + timedelta(minutes=SESSION_TIMEOUT_MINUTES)
            )
            
            self._sessions[session_id] = session
            self._user_sessions[user_id] = session_id
            
            return session_id, session
    
    async def join_session(self, session_id: str, user_name: str, avatar: str) -> tuple[str, Session]:
        """Add user to existing session
        
        Returns: (user_id, updated_session)
        Raises: ValueError for invalid session, RuntimeError if session full
        """
        async with self._lock:
            session = self._get_active_session(session_id)
            
            if len(session.participants) >= MAX_USERS_PER_SESSION:
                raise RuntimeError("Session is full")
            
            user_id = generate_guid()
            user = User(id=user_id, name=user_name, avatar=avatar)
            
            session.participants.append(user)
            session.last_activity = datetime.utcnow()
            self._user_sessions[user_id] = session_id
            
            return user_id, session
    
    async def leave_session(self, session_id: str, user_id: str) -> Session:
        """Remove user from session"""
        async with self._lock:
            session = self._get_active_session(session_id)
            
            # Remove user from participants
            session.participants = [p for p in session.participants if p.id != user_id]
            
            if user_id in self._user_sessions:
                del self._user_sessions[user_id]
            
            # Handle Scrum Master leaving
            if session.scrum_master_id == user_id:
                if session.participants:
                    # Transfer to first remaining participant
                    session.scrum_master_id = session.participants[0].id
                else:
                    # No participants left, mark for cleanup
                    session.expires_at = datetime.utcnow()
            
            session.last_activity = datetime.utcnow()
            return session
    
    def _get_active_session(self, session_id: str) -> Session:
        """Get session if exists and not expired"""
        if session_id not in self._sessions:
            raise ValueError("Session not found")
        
        session = self._sessions[session_id]
        if datetime.utcnow() > session.expires_at:
            raise ValueError("Session has expired")
        
        return session
    
    async def cleanup_expired_sessions(self):
        """Background task to remove expired sessions"""
        async with self._lock:
            current_time = datetime.utcnow()
            expired_sessions = [
                sid for sid, session in self._sessions.items()
                if current_time > session.expires_at
            ]
            
            for session_id in expired_sessions:
                session = self._sessions[session_id]
                # Remove user mappings
                for participant in session.participants:
                    if participant.id in self._user_sessions:
                        del self._user_sessions[participant.id]
                # Remove session
                del self._sessions[session_id]
```

### Voting Manager (`src/services/voting_manager.py`)

```python
from typing import Dict, Optional
from datetime import datetime
from ..models.voting import VotingRound, Vote, VoteValue, VotingStatistics
from ..utils.statistics import calculate_voting_statistics
from ..utils.guid_generator import generate_guid

class VotingManager:
    """Handles voting logic and statistics calculation"""
    
    def __init__(self, session_manager):
        self.session_manager = session_manager
    
    async def start_voting_round(self, session_id: str, user_id: str) -> VotingRound:
        """Start new voting round (Scrum Master only)"""
        session = await self.session_manager.get_session(session_id)
        
        if session.scrum_master_id != user_id:
            raise PermissionError("Only Scrum Master can start voting")
        
        # Create new voting round
        round_id = generate_guid()
        voting_round = VotingRound(id=round_id)
        
        session.current_round = voting_round
        session.status = 'voting'
        session.last_activity = datetime.utcnow()
        
        return voting_round
    
    async def submit_vote(self, session_id: str, user_id: str, vote_value: VoteValue) -> bool:
        """Submit vote for current round"""
        session = await self.session_manager.get_session(session_id)
        
        if not session.current_round:
            raise ValueError("No active voting round")
        
        if session.current_round.is_revealed:
            raise ValueError("Voting round already completed")
        
        # Verify user is participant
        user_exists = any(p.id == user_id for p in session.participants)
        if not user_exists:
            raise ValueError("User not in session")
        
        # Add vote to current round
        vote = Vote(user_id=user_id, value=vote_value)
        session.current_round.votes[user_id] = vote
        session.last_activity = datetime.utcnow()
        
        return True
    
    async def reveal_votes(self, session_id: str, user_id: str) -> VotingStatistics:
        """Reveal votes and calculate statistics (Scrum Master only)"""
        session = await self.session_manager.get_session(session_id)
        
        if session.scrum_master_id != user_id:
            raise PermissionError("Only Scrum Master can reveal votes")
        
        if not session.current_round:
            raise ValueError("No active voting round")
        
        if session.current_round.is_revealed:
            raise ValueError("Votes already revealed")
        
        # Calculate statistics
        statistics = calculate_voting_statistics(session.current_round.votes)
        
        # Update round state
        session.current_round.is_revealed = True
        session.current_round.statistics = statistics
        session.status = 'results'
        session.last_activity = datetime.utcnow()
        
        return statistics
```

### SSE Manager (`src/services/sse_manager.py`)

```python
import asyncio
from typing import Dict, List, Set
from datetime import datetime
from ..models.events import SSEEvent
import json

class SSEManager:
    """Manages Server-Sent Events connections and broadcasting"""
    
    def __init__(self):
        self._session_clients: Dict[str, Set[str]] = {}  # session_id -> client_ids
        self._client_queues: Dict[str, asyncio.Queue] = {}  # client_id -> event queue
        self._lock = asyncio.Lock()
    
    async def register_client(self, session_id: str) -> str:
        """Register new SSE client connection"""
        async with self._lock:
            client_id = f"{session_id}_{datetime.utcnow().timestamp()}"
            
            if session_id not in self._session_clients:
                self._session_clients[session_id] = set()
            
            self._session_clients[session_id].add(client_id)
            self._client_queues[client_id] = asyncio.Queue(maxsize=100)
            
            return client_id
    
    async def unregister_client(self, session_id: str, client_id: str):
        """Unregister SSE client connection"""
        async with self._lock:
            if session_id in self._session_clients:
                self._session_clients[session_id].discard(client_id)
                
                if not self._session_clients[session_id]:
                    del self._session_clients[session_id]
            
            if client_id in self._client_queues:
                del self._client_queues[client_id]
    
    async def broadcast_to_session(self, session_id: str, event: SSEEvent):
        """Broadcast event to all clients in session"""
        async with self._lock:
            if session_id not in self._session_clients:
                return
            
            event_data = event.dict()
            
            # Add to each client's queue
            for client_id in self._session_clients[session_id]:
                if client_id in self._client_queues:
                    try:
                        self._client_queues[client_id].put_nowait(event_data)
                    except asyncio.QueueFull:
                        # Drop oldest event if queue full
                        try:
                            self._client_queues[client_id].get_nowait()
                            self._client_queues[client_id].put_nowait(event_data)
                        except asyncio.QueueEmpty:
                            pass
    
    async def get_pending_events(self, session_id: str, client_id: str) -> List[dict]:
        """Get pending events for specific client"""
        if client_id not in self._client_queues:
            return []
        
        events = []
        queue = self._client_queues[client_id]
        
        # Get all available events without blocking
        try:
            while True:
                event = queue.get_nowait()
                events.append(event)
        except asyncio.QueueEmpty:
            pass
        
        return events
```

## In-Memory Storage Design

### Session Storage Strategy
- **Primary Storage**: Python dictionary with session_id as key
- **User Mapping**: Separate dictionary mapping user_id to session_id for fast lookups
- **TTL Management**: Each session has expires_at timestamp checked during access
- **Cleanup Strategy**: Background asyncio task runs every 2 minutes to remove expired sessions

### Memory Management
- **Session Limit**: Hard limit of 3 concurrent sessions enforced in SessionManager
- **User Limit**: Maximum 16 users per session enforced during join operations
- **Data Retention**: No persistent storage - all data lost on server restart
- **Memory Estimation**: ~50KB per fully loaded session (16 users, voting history)

### Concurrency Control
- **AsyncIO Locks**: Per-service locks prevent race conditions during session modifications
- **Atomic Operations**: Critical operations like join/leave are atomic within locks
- **Event Broadcasting**: SSE events queued per client to handle varying connection speeds

## Error Handling Strategy

### Exception Hierarchy (`src/core/exceptions.py`)
```python
class PlanningPokerException(Exception):
    """Base exception for planning poker errors"""
    pass

class SessionFullException(PlanningPokerException):
    """Raised when session reaches maximum capacity"""
    pass

class SessionNotFoundException(PlanningPokerException):
    """Raised when session doesn't exist or expired"""
    pass

class MaxSessionsException(PlanningPokerException):
    """Raised when maximum concurrent sessions reached"""
    pass

class UnauthorizedException(PlanningPokerException):
    """Raised when user lacks permission for action"""
    pass

class VotingException(PlanningPokerException):
    """Raised for voting-related errors"""
    pass
```

### FastAPI Error Handlers (`src/main.py`)
```python
@app.exception_handler(SessionNotFoundException)
async def session_not_found_handler(request, exc):
    return JSONResponse(
        status_code=404,
        content={"error": "Session not found", "detail": str(exc)}
    )

@app.exception_handler(SessionFullException)
async def session_full_handler(request, exc):
    return JSONResponse(
        status_code=409,
        content={"error": "Session full", "detail": str(exc)}
    )
```

## Testing Strategy

### Unit Test Structure

#### Model Tests (`tests/test_models/`)
- **test_user_model.py**: User validation, serialization, timestamps
- **test_session_model.py**: Session lifecycle, participant management, expiry
- **test_voting_model.py**: Vote validation, statistics calculation, round management
- **test_events_model.py**: SSE event serialization, type validation

#### Service Tests (`tests/test_services/`)
- **test_session_manager.py**: Session CRUD operations, capacity limits, expiry cleanup
- **test_voting_manager.py**: Voting flow, statistics calculation, permission checks
- **test_sse_manager.py**: Client registration, event broadcasting, queue management
- **test_user_manager.py**: User management, role transfers, kick functionality

#### API Tests (`tests/test_api/`)
- **test_sessions_api.py**: Session endpoints, error responses, data validation
- **test_voting_api.py**: Voting endpoints, authentication, state changes
- **test_sse_api.py**: SSE connection handling, event streaming, disconnection cleanup

### Integration Test Strategy

#### Real-time Flow Tests (`tests/test_integration/`)
```python
async def test_complete_voting_flow():
    """Test full voting cycle with multiple users"""
    # Create session
    # Join multiple users
    # Start voting round
    # Submit votes from different users
    # Reveal votes
    # Verify statistics and SSE events
    
async def test_scrum_master_disconnection():
    """Test Scrum Master role transfer on disconnection"""
    # Create session with Scrum Master
    # Add participants
    # Simulate Scrum Master disconnect
    # Verify role transfer
    # Test new Scrum Master permissions
```

### Performance Testing Requirements
- **Concurrent Users**: Test with 16 users in single session
- **Multiple Sessions**: Test with 3 concurrent sessions (48 total users)
- **SSE Load**: Measure event broadcasting latency under load
- **Memory Usage**: Monitor memory consumption over 2-hour session duration
- **Connection Handling**: Test SSE reconnection scenarios

## Deployment Configuration

### Requirements Files

#### Production Dependencies (`requirements.txt`)
```
fastapi==0.104.1
uvicorn[standard]==0.24.0
pydantic==2.5.0
python-multipart==0.0.6
python-dateutil==2.8.2
```

#### Development Dependencies (`requirements-dev.txt`)
```
pytest==7.4.3
pytest-asyncio==0.21.1
httpx==0.25.2
black==23.11.0
ruff==0.1.6
mypy==1.7.1
coverage==7.3.2
```

### Environment Configuration (`src/core/config.py`)
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Server Configuration
    host: str = "0.0.0.0"
    port: int = 8000
    debug: bool = False
    
    # Session Configuration
    max_sessions: int = 3
    max_users_per_session: int = 16
    session_timeout_minutes: int = 10
    cleanup_interval_seconds: int = 120
    
    # CORS Configuration
    cors_origins: List[str] = ["http://localhost:5173"]
    cors_credentials: bool = True
    cors_methods: List[str] = ["GET", "POST", "DELETE"]
    cors_headers: List[str] = ["*"]
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### FastAPI Application Setup (`src/main.py`)
```python
import asyncio
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from contextlib import asynccontextmanager

from .api import sessions, voting, users, sse
from .services.cleanup_service import CleanupService
from .core.config import settings

# Background cleanup task
cleanup_service = CleanupService()

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup
    cleanup_task = asyncio.create_task(cleanup_service.start_cleanup_loop())
    yield
    # Shutdown
    cleanup_task.cancel()
    try:
        await cleanup_task
    except asyncio.CancelledError:
        pass

app = FastAPI(
    title="MagnaPP Planning Poker API",
    description="Real-time collaborative planning poker backend",
    version="1.0.0",
    lifespan=lifespan
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.cors_origins,
    allow_credentials=settings.cors_credentials,
    allow_methods=settings.cors_methods,
    allow_headers=settings.cors_headers,
)

# Include routers
app.include_router(sessions.router)
app.include_router(voting.router)
app.include_router(users.router)
app.include_router(sse.router, prefix="/api/sse")

@app.get("/health")
async def health_check():
    return {"status": "healthy", "timestamp": datetime.utcnow().isoformat()}
```

## Implementation Phases

### Phase 1: Foundation (Days 1-3)
**Goal**: Basic project setup and core models

1. **Project Initialization**
   - Set up FastAPI project structure
   - Configure dependencies and virtual environment
   - Set up basic FastAPI app with health check endpoint
   - Configure CORS for frontend development

2. **Core Models Implementation**
   - Implement all Pydantic models (User, Session, Voting, Events)
   - Add model validation and serialization
   - Write comprehensive model unit tests
   - Verify model JSON serialization works correctly

3. **Basic Configuration**
   - Implement configuration management with environment variables
   - Set up logging configuration
   - Add basic error handling and exception classes
   - Create utility functions (GUID generation, validators)

### Phase 2: Session Management (Days 4-6)
**Goal**: Complete session lifecycle management

1. **Session Manager Service**
   - Implement in-memory session storage
   - Add session creation, joining, leaving logic
   - Implement session capacity limits (3 sessions, 16 users each)
   - Add session expiry and cleanup mechanisms

2. **Session API Endpoints**
   - Implement all session management endpoints
   - Add comprehensive error handling and validation
   - Write API tests for all endpoints
   - Test session limits and error conditions

3. **User Management**
   - Implement user management within sessions
   - Add Scrum Master role handling and transfers
   - Implement user kick functionality
   - Test user lifecycle and role management

### Phase 3: Voting System (Days 7-9)
**Goal**: Complete voting functionality

1. **Voting Manager Service**
   - Implement voting round management
   - Add vote submission and validation logic
   - Implement statistics calculation (average, distribution, consensus)
   - Add voting state management and transitions

2. **Voting API Endpoints**
   - Implement voting endpoints (start, submit, reveal)
   - Add permission checks for Scrum Master actions
   - Write comprehensive voting tests
   - Test all voting scenarios and edge cases

3. **Statistics and Results**
   - Implement Fibonacci sequence handling
   - Add coffee vote tracking
   - Implement consensus detection
   - Test statistics calculations with various vote combinations

### Phase 4: Real-time Communication (Days 10-12)
**Goal**: Server-Sent Events implementation

1. **SSE Manager Service**
   - Implement SSE client connection management
   - Add event broadcasting system
   - Implement client registration and cleanup
   - Add event queuing and delivery system

2. **SSE Endpoint Implementation**
   - Create SSE streaming endpoint
   - Implement proper headers and connection handling
   - Add reconnection support and error recovery
   - Test SSE with multiple concurrent clients

3. **Event Integration**
   - Integrate SSE broadcasting with session and voting operations
   - Implement all event types (user_joined, vote_submitted, etc.)
   - Add event serialization and delivery
   - Test real-time synchronization across multiple clients

### Phase 5: Testing and Polish (Days 13-15)
**Goal**: Comprehensive testing and production readiness

1. **Integration Testing**
   - Write complete user flow integration tests
   - Test multi-user scenarios with concurrent operations
   - Add SSE integration tests with real connections
   - Test session expiry and cleanup under load

2. **Performance Testing**
   - Load test with maximum users (16 per session, 3 sessions)
   - Test SSE performance with rapid event broadcasting
   - Monitor memory usage and optimize if needed
   - Test reconnection scenarios and recovery

3. **Production Readiness**
   - Add comprehensive logging for debugging
   - Implement health checks and monitoring endpoints
   - Add graceful shutdown handling
   - Write deployment documentation and examples

## Key Implementation Notes

### Real-time Synchronization
- **Sub-second Updates**: All state changes must broadcast SSE events within 100ms
- **Event Ordering**: Events are queued per client to maintain order during network issues
- **Connection Recovery**: Clients automatically reconnect via EventSource built-in functionality
- **State Consistency**: Server-side state is source of truth, frontend applies updates optimistically

### Session Management Constraints  
- **Hard Limits**: Enforce 3 concurrent sessions and 16 users per session at API level
- **Automatic Cleanup**: Background task removes expired sessions every 2 minutes
- **Scrum Master Transfer**: Automatically transfer role when Scrum Master disconnects
- **Grace Period**: 5-minute grace period for Scrum Master reconnection (pause session)

### Performance Considerations
- **Memory Usage**: Estimated 150KB maximum memory for 3 full sessions
- **Event Broadcasting**: Batch SSE events when possible to reduce connection overhead  
- **Connection Pooling**: SSE manager reuses connections and cleans up inactive clients
- **Async Operations**: All I/O operations use async/await for better concurrency

### Error Handling Patterns
- **Graceful Degradation**: API continues working even if SSE connections fail
- **Clear Error Messages**: All errors include actionable messages for frontend display
- **Logging Strategy**: Log all errors with session/user context for debugging
- **Recovery Mechanisms**: Failed operations can be retried without side effects

### Security Considerations  
- **No Authentication**: Open sessions as specified in requirements
- **Input Validation**: All inputs validated through Pydantic models
- **Session Isolation**: Sessions are completely isolated in memory
- **CORS Configuration**: Properly configured for frontend domain
- **Rate Limiting**: Consider implementing basic rate limiting for production

This comprehensive implementation plan provides step-by-step guidance for building a robust, scalable Planning Poker backend that meets all specified requirements while maintaining code quality and performance standards.