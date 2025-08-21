# Technology Stack Decisions

## Date: 2025-08-21

### Frontend Framework
**Decision**: React with TypeScript
**Rationale**: 
- Component-based architecture ideal for real-time UI updates
- Strong TypeScript support for type safety
- Large ecosystem with excellent WebSocket/Socket.io integration
- Proven performance for real-time collaborative applications

### Backend Framework
**Decision**: Python FastAPI
**Rationale**:
- Modern async Python framework with excellent performance
- Native WebSocket support with async/await patterns
- Built-in data validation with Pydantic
- Automatic API documentation generation
- Clean architecture for REST + WebSocket hybrid approach

### Real-time Communication
**Decision**: Socket.io (python-socketio for backend, socket.io-client for frontend)
**Rationale**:
- Automatic reconnection with exponential backoff (PRD requirement)
- Room-based communication perfect for session management
- Fallback transport mechanisms ensure reliability
- Built-in heartbeat for connection monitoring
- Event-based architecture matches our voting round workflow

### State Management
**Decision**: Redux Toolkit
**Rationale**:
- Predictable state updates crucial for real-time collaboration
- Redux DevTools for debugging complex state changes
- RTK Query can handle REST API calls efficiently
- Excellent TypeScript support
- Time-travel debugging useful for multi-round voting sessions

### UI Framework
**Decision**: Material-UI (MUI) v5
**Rationale**:
- Comprehensive component library reduces development time
- Built-in responsive design components
- Consistent design language
- Accessibility features built-in (PRD requirement)
- Theme customization for light/dark mode support

### Testing Strategy
**Decision**: Vitest + Playwright
**Rationale**:
- Vitest: Fast unit testing with native TypeScript support
- Playwright: Cross-browser E2E testing (Chrome, Edge, Firefox, Safari as per PRD)
- Both tools have excellent async/WebSocket testing capabilities
- Playwright can test real-time multi-user scenarios

### Additional Technical Decisions

#### Build Tools
- **Frontend**: Vite (fast builds, native TypeScript, excellent DX)
- **Backend**: Poetry for Python dependency management
- **Monorepo**: Separate frontend/backend folders with independent builds

#### Code Quality
- **Frontend**: ESLint + Prettier for React/TypeScript
- **Backend**: Ruff (fast Python linter) + Black (formatter)
- **Pre-commit hooks**: Husky + lint-staged

#### Development Environment
- **Frontend Dev Server**: Vite dev server with HMR
- **Backend Dev Server**: Uvicorn with --reload
- **API Proxy**: Vite proxy to backend during development

#### Deployment Considerations
- **Frontend**: Static build served via nginx or CDN
- **Backend**: ASGI server (Uvicorn) behind reverse proxy
- **WebSocket**: Ensure proxy supports WebSocket upgrade

#### Session Storage
- **Implementation**: Python dict with TTL management
- **Cleanup**: Background task for expired session removal
- **Capacity**: Track active sessions to enforce 3-session limit