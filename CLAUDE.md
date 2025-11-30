# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

FalkorDB MCP Server is a TypeScript/Node.js application implementing the Model Context Protocol (MCP) to provide AI models with standardized access to FalkorDB graph databases. It acts as a bridge between MCP clients and FalkorDB, exposing a RESTful API for graph operations.

## Common Commands

```bash
# Install dependencies
npm install

# Development with hot reload
npm run dev

# Build for production
npm run build

# Run production build
npm start

# Linting
npm run lint

# Run all tests
npm test

# Run tests with coverage
npm run test:coverage

# Run a single test file
npm test -- src/controllers/mcp.controller.test.ts

# Docker build and run
npm run docker:build
npm run docker:run
```

## Architecture

The project follows a 3-tier layered architecture:

```
src/
├── index.ts                 # Express app entry point, server lifecycle
├── config/index.ts          # Environment configuration (dotenv)
├── routes/mcp.routes.ts     # REST API route definitions
├── controllers/             # Request handlers, validation, response formatting
├── services/                # Business logic, FalkorDB connection management
├── middleware/              # Authentication (API key validation)
├── models/                  # TypeScript type definitions for MCP protocol
└── utils/                   # Connection string parsing utilities
```

**Key patterns:**
- **Singleton services** - `falkorDBService` and `mcpController` are exported as singleton instances
- **Middleware pipeline** - Authentication applied to all `/api/mcp/*` routes
- **Async/await** - All database operations are asynchronous
- **Graceful shutdown** - SIGTERM/SIGINT handlers close DB connections

## API Endpoints

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/` | GET | No | Server health/version info |
| `/api/mcp/context` | POST | Yes | Execute Cypher queries |
| `/api/mcp/metadata` | GET | Yes | Server capabilities |
| `/api/mcp/graphs` | GET | Yes | List available graphs |
| `/api/mcp/health` | GET | Yes | API health check |

Authentication: `x-api-key` header or `apiKey` query parameter.

## Environment Configuration

Copy `.env.example` to `.env`:

```env
PORT=3000
NODE_ENV=development
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
FALKORDB_USERNAME=
FALKORDB_PASSWORD=
MCP_API_KEY=your_api_key_here
```

## Testing

Tests use Jest with ts-jest. Test files are co-located with source files using `.test.ts` suffix:
- `src/controllers/mcp.controller.test.ts`
- `src/services/falkordb.service.test.ts`
- `src/config/index.test.ts`

## Note on `ra_*` Directories

The `ra_orchestrators/`, `ra_agents/`, `ra_tools/`, and `ra_output/` directories contain a separate Repository Analyzer Framework used for codebase analysis. They are not part of the MCP server application.