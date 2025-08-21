# MagnaPP - Planning Poker Application

A real-time web-based Planning Poker application for distributed agile teams, enabling collaborative estimation sessions through an intuitive virtual boardroom interface.

## Features

- 🎯 Real-time collaborative estimation sessions
- 👥 Support for up to 16 participants per session
- 🎴 Fibonacci sequence voting cards (1, 2, 3, 5, 8, 13, 21, ☕)
- 🔄 Automatic session management with expiry
- 📱 Responsive design for desktop, tablet, and mobile
- ⚡ Live synchronization with <1 second latency
- 🔌 Automatic reconnection support
- 📊 Voting statistics and distribution visualization

## Tech Stack

- **Frontend**: React 18 with TypeScript, Material-UI, Redux Toolkit
- **Backend**: Python FastAPI with Socket.io
- **Real-time**: Socket.io for WebSocket communication
- **Testing**: Vitest + Playwright
- **Build Tools**: Vite (frontend), Poetry (backend)

## Project Structure

```
planning-poker-magnapp/
├── frontend/               # React TypeScript application
│   ├── src/
│   │   ├── components/    # Reusable UI components
│   │   ├── pages/        # Page components
│   │   ├── services/     # API and Socket.io services
│   │   ├── store/        # Redux store and slices
│   │   └── utils/        # Utility functions
│   └── package.json
├── backend/               # FastAPI Python application
│   ├── app/
│   │   ├── api/          # REST API endpoints
│   │   ├── models/       # Pydantic models
│   │   ├── services/     # Business logic
│   │   └── utils/        # Utility functions
│   └── pyproject.toml
├── docs/                  # Documentation
├── plans/                 # Project planning documents
└── tests/                # E2E tests with Playwright
```

## Quick Start

### Prerequisites

- Node.js 18+ and npm
- Python 3.11+
- Poetry (Python package manager)

### Backend Setup

```bash
cd backend
poetry install
poetry run uvicorn app.main:app --reload --port 8000
```

### Frontend Setup

```bash
cd frontend
npm install
npm run dev
```

The application will be available at `http://localhost:5173`

## Development

### Available Scripts

#### Frontend
- `npm run dev` - Start development server
- `npm run build` - Build for production
- `npm run test` - Run unit tests
- `npm run lint` - Run ESLint
- `npm run format` - Format code with Prettier

#### Backend
- `poetry run dev` - Start development server
- `poetry run test` - Run pytest suite
- `poetry run lint` - Run Ruff linter
- `poetry run format` - Format with Black

### Environment Variables

Copy `.env.example` to `.env` in both frontend and backend directories and configure as needed.

#### Frontend (.env)
```
VITE_API_URL=http://localhost:8000
VITE_WS_URL=ws://localhost:8000
```

#### Backend (.env)
```
HOST=0.0.0.0
PORT=8000
CORS_ORIGINS=http://localhost:5173
MAX_SESSIONS=3
MAX_USERS_PER_SESSION=16
SESSION_TIMEOUT_MINUTES=10
```

## Testing

### Unit Tests
```bash
# Frontend
cd frontend && npm run test

# Backend
cd backend && poetry run test
```

### E2E Tests
```bash
npm run test:e2e
```

## Contributing

1. Check `plans/TASKS.md` for current development tasks
2. Follow the existing code style and conventions
3. Write tests for new features
4. Update documentation as needed

## Documentation

- [Technology Decisions](docs/DECISIONS.md)
- [Assumptions](docs/ASSUMPTIONS.md)
- [Open Questions](docs/QUESTIONS.md)
- [Build Log](docs/BUILD_LOG.md)
- [Task Plan](plans/TASKS.md)

## License

MIT License - see [LICENSE](LICENSE) file for details

## Support

For questions or issues, please check the [documentation](docs/) or create an issue in the repository.