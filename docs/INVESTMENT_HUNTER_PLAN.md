# Investment Hunter v2.0 - Canonical Plan

## Project Overview
Investment Hunter v2.0 is a comprehensive vehicle investment analysis platform that eliminates 90% of resource waste through intelligent automation and real-time data processing.

## Technology Stack
- **Frontend**: React 18 + TypeScript + Vite + Shadcn-ui + Tailwind CSS
- **Backend**: Node.js + Express + PostgreSQL + Prisma ORM
- **Automation**: n8n v1.111.0 (Self-hosted Docker + SQLite)
- **AI**: OpenAI GPT-4 API
- **Real-time**: WebSocket + Server-Sent Events

## Development Phases
### Phase 1: Foundation Setup (Week 1)
- Git repository and environment setup
- Database integration (PostgreSQL)
- API proxy integration
- n8n workflow automation

### Phase 2: UI Migration & Optimization (Week 2)
- Chat interface migration
- Vehicle component updates
- Performance optimization

### Phase 3: Advanced Features (Week 3)
- Advanced search implementation
- Real-time features
- WebSocket integration

### Phase 4: Production Readiness (Week 4)
- Data processing enhancement
- Comprehensive testing
- Production deployment preparation

## Key Principles
- **No Mock Data**: All data comes from real sources (PostgreSQL, n8n, API)
- **Git-based Development**: Feature branches, PR reviews, no-ff merges
- **Performance First**: <2s load time, optimized queries
- **Real-time Updates**: Live price changes, notifications
