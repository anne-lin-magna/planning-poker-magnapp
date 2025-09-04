# Technology Stack Decisions

## Date: 2025-08-21

### Frontend Framework
**Decision**: React with TypeScript
**Rationale**: 
- Component-based architecture ideal for real-time UI updates
- Strong TypeScript support for type safety
- Large ecosystem with excellent EventSource/SSE integration
- Proven performance for real-time collaborative applications

### Backend Framework
**Decision**: Python FastAPI
**Rationale**:
- Modern async Python framework with excellent performance
- Native Server-Sent Events support with async/await patterns
- Built-in data validation with Pydantic
- Automatic API documentation generation
- Clean architecture for REST + SSE hybrid approach

### Real-time Communication
**Decision**: Server-Sent Events (SSE) with EventSource API
**Rationale**:
- Simple HTTP-based streaming, no complex connection management
- Automatic reconnection with exponential backoff (PRD requirement)
- Works through corporate firewalls and proxies without configuration
- Event-based architecture perfect for voting status updates
- Actions sent via standard HTTP POST requests to REST API
- No bidirectional complexity - server pushes updates, client sends actions

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
- Both tools have excellent async/SSE testing capabilities
- Playwright can test real-time multi-user scenarios with EventSource

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
- **SSE**: Standard HTTP streaming, no special proxy configuration needed

#### Session Storage
- **Implementation**: Python dict with TTL management
- **Cleanup**: Background task for expired session removal
- **Capacity**: Track active sessions to enforce 3-session limit

#### Security Decisions
- **Authentication**: None required - open sessions for Phase 1
- **Session Links**: GUID-based, not time-limited (10min timeout handles security)
- **Data Protection**: HTTPS transport encryption only, no additional encryption needed
- **Access Control**: Link-sharing model sufficient for internal corporate use