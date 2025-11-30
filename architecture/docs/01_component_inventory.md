# Component Inventory

## Overview

The FalkorDB MCP Server is a **hybrid TypeScript/Python project** that implements a Model Context Protocol (MCP) server for FalkorDB graph databases. The main application is built with TypeScript/Node.js, providing an Express-based REST API that allows AI models to query and interact with FalkorDB instances. The project contains minimal Python code (a single hello-world entry point) alongside a comprehensive TypeScript implementation.

**Technology Stack:**
- **Primary Language**: TypeScript (Node.js)
- **Web Framework**: Express.js
- **Graph Database**: FalkorDB
- **MCP Integration**: @modelcontextprotocol/sdk v1.8.0
- **Python Component**: Minimal (single main.py file)

**Project Statistics:**
- **Total Python Files**: 1 (main.py)
- **Total TypeScript Files**: 11 (application code)
- **Primary Codebase**: TypeScript-based REST API server
- **Architecture Pattern**: MVC (Model-View-Controller) with service layer

---

## Public API

### Python Modules

#### main.py
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/main.py`
**Purpose**: Simple Python entry point demonstrating basic project setup

**Functions**:
- `main()` (line 1): Prints a hello message for the falkordb-mcpserver project

**Entry Point**:
- `if __name__ == "__main__"` block (line 5) calls `main()`

**Note**: This appears to be a placeholder or testing file and is not integrated with the main TypeScript application.

---

### TypeScript Modules

### Core Application

#### index.ts (Main Application Entry Point)
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`
**Purpose**: Express application initialization, middleware configuration, and server lifecycle management

**Key Components**:
- Express app initialization (line 8)
- Middleware setup for JSON and URL-encoded request parsing (lines 11-12)
- MCP route mounting with authentication (line 15)
- Root endpoint providing server metadata (lines 18-24)
- Server startup on configured port (lines 27-31)
- Graceful shutdown handlers for SIGTERM and SIGINT (lines 34-46)

**Public API**:
- `GET /` - Returns server name, version, and status
- All MCP routes mounted at `/api/mcp` (authenticated)

**Dependencies**:
- express
- ./config
- ./routes/mcp.routes
- ./middleware/auth.middleware
- ./services/falkordb.service

---

### Configuration

#### config/index.ts
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts`
**Purpose**: Centralized configuration management using environment variables

**Exports**:
- `config` object (line 6) containing:
  - `server.port`: Server port (default: 3000)
  - `server.nodeEnv`: Environment mode (default: 'development')
  - `falkorDB.host`: FalkorDB host (default: 'localhost')
  - `falkorDB.port`: FalkorDB port (default: 6379)
  - `falkorDB.username`: FalkorDB username (optional)
  - `falkorDB.password`: FalkorDB password (optional)
  - `mcp.apiKey`: API key for MCP authentication (optional)

**Dependencies**: dotenv (for loading .env files)

---

### Controllers

#### controllers/mcp.controller.ts
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`
**Purpose**: Handles MCP API request processing and response formatting

**Class**: `MCPController`

**Public Methods**:

1. **processContextRequest** (line 14)
   - **Purpose**: Execute graph queries on FalkorDB
   - **Parameters**: Express Request and Response objects
   - **Validation**: Requires `query` and `graphName` in request body
   - **Returns**: MCPResponse with query results and metadata
   - **HTTP Codes**: 200 (success), 400 (validation error), 500 (server error)

2. **processMetadataRequest** (line 64)
   - **Purpose**: Return server capabilities and metadata
   - **Returns**: MCPProviderMetadata containing:
     - Provider name and version
     - Supported capabilities (graph.query, graph.list, node.properties, relationship.properties)
     - Graph types (property, directed)
     - Query languages (cypher)
   - **HTTP Codes**: 200 (success), 500 (server error)

3. **listGraphs** (line 90)
   - **Purpose**: List all available graphs in the FalkorDB instance
   - **Returns**: Array of graph objects with names and metadata
   - **HTTP Codes**: 200 (success), 500 (server error)

**Singleton Export**: `mcpController` (line 116)

**Dependencies**:
- express (Request, Response types)
- ../services/falkordb.service
- ../models/mcp.types

---

### Services

#### services/falkordb.service.ts
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`
**Purpose**: FalkorDB database connection and query execution service

**Class**: `FalkorDBService`

**Private Properties**:
- `client`: FalkorDB | null (line 5) - Database client instance

**Methods**:

1. **constructor** (line 7)
   - Initializes the service and starts connection

2. **init** (private, line 11)
   - **Purpose**: Establish connection to FalkorDB
   - **Features**:
     - Connection testing via ping
     - Automatic retry with 5-second delay on failure
     - Connection logging
   - **Configuration**: Uses config object for host, port, username, password

3. **executeQuery** (line 33)
   - **Purpose**: Execute Cypher queries on a specific graph
   - **Parameters**:
     - graphName: string - Target graph name
     - query: string - Cypher query to execute
     - params?: Record<string, any> - Optional query parameters
   - **Returns**: Query results from FalkorDB
   - **Error Handling**: Logs sanitized graph name and rethrows errors

4. **listGraphs** (line 53)
   - **Purpose**: Retrieve list of all graphs in FalkorDB instance
   - **Returns**: Array of graph name strings
   - **Error Handling**: Logs errors and rethrows

5. **close** (line 67)
   - **Purpose**: Gracefully close database connection
   - **Cleanup**: Sets client to null after closing

**Singleton Export**: `falkorDBService` (line 76)

**Dependencies**:
- falkordb (FalkorDB client library)
- ../config

---

### Routes

#### routes/mcp.routes.ts
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts`
**Purpose**: Define MCP API endpoints and route handlers

**Routes**:
- `POST /context` (line 7) - Process graph query requests
- `GET /metadata` (line 8) - Get server metadata and capabilities
- `GET /graphs` (line 9) - List available graphs
- `GET /health` (line 12) - Health check endpoint (returns `{ status: 'ok' }`)

**Export**: `mcpRoutes` (line 16) - Express Router instance

**Dependencies**:
- express (Router)
- ../controllers/mcp.controller

**Note**: All routes are mounted at `/api/mcp` in the main application (with authentication middleware)

---

### Middleware

#### middleware/auth.middleware.ts
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`
**Purpose**: Authentication middleware for MCP API requests

**Function**: `authenticateMCP` (line 7)
- **Purpose**: Validate API key for MCP requests
- **API Key Sources**:
  - HTTP header: `x-api-key`
  - Query parameter: `apiKey`
- **Development Mode**: Skips authentication if no API key is configured (with warning)
- **HTTP Codes**:
  - 401 (missing API key)
  - 403 (invalid API key)
  - Calls next() for valid requests

**Dependencies**:
- express (Request, Response, NextFunction types)
- ../config

**Security Features**:
- Configurable API key validation
- Development mode bypass
- Clear error messages for authentication failures

---

### Models (Type Definitions)

#### models/mcp.types.ts
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts`
**Purpose**: TypeScript type definitions for MCP protocol

**Interfaces**:

1. **MCPContextRequest** (line 5)
   - graphName: string - Target graph
   - query: string - Cypher query
   - params?: Record<string, any> - Query parameters
   - context?: Record<string, any> - Additional context
   - options?: MCPOptions - Query options

2. **MCPOptions** (line 13)
   - timeout?: number
   - maxResults?: number
   - Extensible with additional properties

3. **MCPResponse** (line 19)
   - data: any - Query results
   - metadata: MCPMetadata - Response metadata

4. **MCPMetadata** (line 24)
   - timestamp: string - ISO timestamp
   - queryTime: number - Query execution time in milliseconds
   - provider?: string - Provider name
   - source?: string - Data source
   - Extensible with additional properties

5. **MCPProviderMetadata** (line 32)
   - provider: string - Provider name
   - version: string - Provider version
   - capabilities: string[] - Supported features
   - graphTypes: string[] - Supported graph types
   - queryLanguages: string[] - Supported query languages
   - Extensible with additional properties

---

#### models/mcp-client-config.ts
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp-client-config.ts`
**Purpose**: Configuration types and samples for MCP client/server setup

**Interfaces**:

1. **MCPServerConfig** (line 5)
   - mcpServers: Configuration for Docker-based server deployment

2. **MCPClientConfig** (line 14)
   - defaultServer?: string
   - servers: Server connection configurations

**Constants**:

1. **sampleMCPClientConfig** (line 27)
   - Example client configuration with URL and API key

2. **sampleMCPServerConfig** (line 40)
   - Example Docker-based server deployment configuration

---

### Utilities

#### utils/connection-parser.ts
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/utils/connection-parser.ts`
**Purpose**: Parse FalkorDB connection strings

**Interface**: `FalkorDBConnectionOptions` (line 5)
- host: string
- port: number
- username?: string
- password?: string

**Function**: `parseFalkorDBConnectionString` (line 19)
- **Purpose**: Parse connection strings in format `falkordb://[username:password@]host:port`
- **Parameters**: connectionString: string
- **Returns**: FalkorDBConnectionOptions
- **Features**:
  - Protocol prefix handling
  - Authentication parsing (username:password)
  - Host and port extraction
  - Default values (localhost:6379)
  - Error handling with fallback to defaults
- **Example**: `falkordb://user:pass@host.docker.internal:6379`

---

## Internal Implementation

### Internal Modules

The project follows a clean separation of concerns with no hidden internal modules. All TypeScript modules are part of the public API structure:

- **Configuration Layer**: `config/index.ts`
- **Service Layer**: `services/falkordb.service.ts`
- **Controller Layer**: `controllers/mcp.controller.ts`
- **Routing Layer**: `routes/mcp.routes.ts`
- **Middleware Layer**: `middleware/auth.middleware.ts`
- **Type Definitions**: `models/` directory
- **Utilities**: `utils/` directory

### Helper Classes

No dedicated helper classes exist. The codebase uses:
- Singleton pattern for `FalkorDBService` and `MCPController`
- Direct instantiation and export of service/controller instances
- Functional utility exports (e.g., `parseFalkorDBConnectionString`)

### Utility Functions

#### Connection String Parsing
- **Function**: `parseFalkorDBConnectionString` (connection-parser.ts, line 19)
- **Type**: Internal utility
- **Usage**: Parse FalkorDB connection URIs
- **Error Handling**: Returns defaults on parse failure

---

## Entry Points

### Main Entry Points

#### TypeScript Application Entry Point
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`
**Type**: Express.js server application
**Port**: Configurable via PORT environment variable (default: 3000)
**Startup**:
```typescript
app.listen(PORT, () => {
  console.log(`FalkorDB MCP Server listening on port ${PORT}`);
  console.log(`Environment: ${config.server.nodeEnv}`);
});
```

**Lifecycle Management**:
- SIGTERM handler (line 34): Graceful shutdown with DB connection cleanup
- SIGINT handler (line 41): Graceful shutdown with DB connection cleanup

**Scripts** (from package.json):
- `npm run dev`: Development mode with hot-reloading (nodemon + ts-node)
- `npm run build`: TypeScript compilation to dist/
- `npm start`: Production mode (runs compiled JS from dist/index.js)
- `npm test`: Jest test runner
- `npm run lint`: ESLint type checking

#### Python Entry Point
**File**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/main.py`
**Type**: Standalone Python script
**Execution**: `python main.py` or `python3 main.py`
**Functionality**: Prints "Hello from falkordb-mcpserver!"
**Integration**: Not integrated with the main TypeScript application

---

### API Interfaces

#### REST API Interface

**Base URL**: `http://localhost:3000` (configurable)
**Authentication**: API key via `x-api-key` header or `apiKey` query parameter

**Endpoints**:

1. **Root Endpoint**
   - **URL**: `GET /`
   - **Auth**: Not required
   - **Response**: Server metadata (name, version, status)

2. **MCP Context (Query Execution)**
   - **URL**: `POST /api/mcp/context`
   - **Auth**: Required
   - **Body**: MCPContextRequest JSON
   - **Response**: MCPResponse with query results

3. **MCP Metadata (Capabilities)**
   - **URL**: `GET /api/mcp/metadata`
   - **Auth**: Required
   - **Response**: MCPProviderMetadata

4. **List Graphs**
   - **URL**: `GET /api/mcp/graphs`
   - **Auth**: Required
   - **Response**: Array of graph objects

5. **Health Check**
   - **URL**: `GET /api/mcp/health`
   - **Auth**: Required
   - **Response**: `{ status: 'ok' }`

**Example Usage**:
```bash
# Query a graph
curl -X POST http://localhost:3000/api/mcp/context \
  -H "x-api-key: your_api_key" \
  -H "Content-Type: application/json" \
  -d '{
    "graphName": "my_graph",
    "query": "MATCH (n) RETURN n LIMIT 10"
  }'

# List graphs
curl http://localhost:3000/api/mcp/graphs \
  -H "x-api-key: your_api_key"
```

---

## Dependencies and Imports

### External Dependencies

#### Production Dependencies
- **express** (^4.21.2): Web framework for REST API
- **@modelcontextprotocol/sdk** (^1.8.0): MCP protocol implementation
- **falkordb** (^6.2.07): FalkorDB client library
- **dotenv** (^16.4.7): Environment variable management

#### Development Dependencies
- **typescript** (^5.3.3): TypeScript compiler
- **ts-node** (^10.9.2): TypeScript execution for Node.js
- **nodemon** (^3.0.2): Development server with hot-reloading
- **@types/express** (^4.17.21): TypeScript types for Express
- **@types/node** (^20.11.0): TypeScript types for Node.js
- **jest** (^29.7.0): Testing framework
- **@types/jest** (^29.5.11): TypeScript types for Jest
- **ts-jest** (^29.1.1): Jest TypeScript preprocessor
- **eslint** (^8.56.0): Linting tool
- **@typescript-eslint/eslint-plugin** (^6.15.0): TypeScript ESLint plugin
- **@typescript-eslint/parser** (^6.15.0): TypeScript ESLint parser

#### Python Dependencies (pyproject.toml)
- **claude-agent-sdk** (>=0.1.10): Agent SDK (appears unused in current code)

---

### Internal Dependency Graph

```
index.ts (main entry)
├── config/index.ts (configuration)
├── routes/mcp.routes.ts
│   └── controllers/mcp.controller.ts
│       ├── services/falkordb.service.ts
│       │   └── config/index.ts
│       └── models/mcp.types.ts
├── middleware/auth.middleware.ts
│   └── config/index.ts
└── services/falkordb.service.ts (for graceful shutdown)

Standalone modules:
├── models/mcp-client-config.ts (type definitions only)
└── utils/connection-parser.ts (utility function, not currently imported)
```

**Module Relationships**:
- **config/index.ts**: Imported by index.ts, auth.middleware.ts, falkordb.service.ts
- **services/falkordb.service.ts**: Imported by index.ts (shutdown), mcp.controller.ts (queries)
- **controllers/mcp.controller.ts**: Imported by mcp.routes.ts
- **middleware/auth.middleware.ts**: Imported by index.ts
- **routes/mcp.routes.ts**: Imported by index.ts
- **models/mcp.types.ts**: Imported by mcp.controller.ts
- **models/mcp-client-config.ts**: Not imported (documentation/example types)
- **utils/connection-parser.ts**: Not imported (available for use)

---

## File Reference Index

### Python Files
1. `/home/donbr/graphiti-org/FalkorDB-MCPServer/main.py` - Simple hello-world entry point

### TypeScript Application Files
1. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts` - Main application entry point
2. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts` - Configuration management
3. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts` - MCP request handlers
4. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts` - FalkorDB service layer
5. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts` - API route definitions
6. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts` - Authentication middleware
7. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts` - MCP type definitions
8. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp-client-config.ts` - Client configuration types
9. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/utils/connection-parser.ts` - Connection string parser

### TypeScript Test Files
10. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.test.ts` - Configuration tests
11. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.test.ts` - Service tests
12. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.test.ts` - Controller tests

### Configuration Files
13. `/home/donbr/graphiti-org/FalkorDB-MCPServer/package.json` - Node.js project configuration
14. `/home/donbr/graphiti-org/FalkorDB-MCPServer/pyproject.toml` - Python project configuration
15. `/home/donbr/graphiti-org/FalkorDB-MCPServer/tsconfig.json` - TypeScript compiler configuration

---

## Architecture Summary

The FalkorDB MCP Server follows a **layered architecture** pattern:

1. **Presentation Layer**: Express routes and middleware
2. **Controller Layer**: Request/response handling and validation
3. **Service Layer**: Business logic and database operations
4. **Model Layer**: Type definitions and data structures
5. **Utility Layer**: Helper functions and parsers

**Key Design Patterns**:
- **Singleton Pattern**: Used for FalkorDBService and MCPController
- **Middleware Pattern**: Authentication and request processing
- **MVC Architecture**: Separation of routing, control, and service logic
- **Configuration Management**: Centralized environment-based configuration
- **Graceful Shutdown**: Proper cleanup of database connections

**Code Organization**:
- Clear separation of concerns
- Type-safe TypeScript implementation
- RESTful API design
- Comprehensive error handling
- Production-ready with Docker support

**Notable Features**:
- MCP protocol compliance
- FalkorDB graph database integration
- API key authentication
- Health check endpoints
- Configurable deployment (development/production modes)
- Docker containerization support
- Automatic database connection retry
- Query parameter support for Cypher queries
