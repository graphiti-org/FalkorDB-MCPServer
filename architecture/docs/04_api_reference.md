# API Reference

## Overview

The FalkorDB MCP Server is a Model Context Protocol (MCP) implementation that provides a standardized interface for AI models to query and interact with FalkorDB graph databases. It exposes a RESTful API that translates MCP requests into FalkorDB operations and formats responses according to the MCP specification.

**Key Features:**
- RESTful API following MCP standards
- Graph query execution using Cypher
- Graph discovery and metadata retrieval
- Authentication via API keys
- Automatic connection management with retry logic
- Comprehensive error handling

**Technology Stack:**
- Node.js with TypeScript
- Express.js web framework
- FalkorDB client library (v6.2.07)
- Model Context Protocol SDK (v1.8.0)

---

## Table of Contents
- [Core Services](#core-services)
- [Controllers](#controllers)
- [Middleware](#middleware)
- [Models & Types](#models--types)
- [Utilities](#utilities)
- [Configuration](#configuration)
- [REST API Endpoints](#rest-api-endpoints)
- [Usage Examples](#usage-examples)
- [Best Practices](#best-practices)
- [Appendix](#appendix)

---

## Core Services

### FalkorDBService

**File**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:4](../../../src/services/falkordb.service.ts#L4)

**Description**: Singleton service that manages FalkorDB connections and provides methods for executing graph queries. Handles automatic connection initialization, retry logic, and connection pooling.

#### Constructor

##### `constructor()`

**Signature**:
```typescript
constructor()
```

**Description**: Initializes the FalkorDBService singleton instance. Automatically calls the private `init()` method to establish a connection to FalkorDB.

**Parameters**: None

**Returns**: `FalkorDBService` instance

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:7](../../../src/services/falkordb.service.ts#L7)

---

#### Methods

##### `executeQuery()`

**Signature**:
```typescript
async executeQuery(
  graphName: string,
  query: string,
  params?: Record<string, any>
): Promise<any>
```

**Description**: Executes a Cypher query against the specified graph in FalkorDB. Supports parameterized queries for security and performance.

**Parameters**:
- `graphName` (string) - The name of the graph to query against. Required.
- `query` (string) - The Cypher query to execute. Required.
- `params` (Record<string, any>) - Optional query parameters for parameterized queries. Default: `undefined`

**Returns**:
- `Promise<any>` - The query result object from FalkorDB, containing records and metadata

**Throws**:
- `Error` - "FalkorDB client not initialized" if the client connection is not established
- `Error` - FalkorDB-specific errors if the query fails (invalid syntax, graph not found, etc.)

**Example**:
```typescript
// Simple query without parameters
const result = await falkorDBService.executeQuery(
  'myGraph',
  'MATCH (n:Person) RETURN n LIMIT 10'
);

// Parameterized query
const result = await falkorDBService.executeQuery(
  'socialGraph',
  'MATCH (p:Person {name: $name}) RETURN p',
  { name: 'John Doe' }
);
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:33](../../../src/services/falkordb.service.ts#L33)

---

##### `listGraphs()`

**Signature**:
```typescript
async listGraphs(): Promise<string[]>
```

**Description**: Retrieves a list of all available graphs in the connected FalkorDB instance.

**Parameters**: None

**Returns**:
- `Promise<string[]>` - Array of graph names as strings

**Throws**:
- `Error` - "FalkorDB client not initialized" if the client connection is not established
- `Error` - Database errors if the list operation fails

**Example**:
```typescript
const graphs = await falkorDBService.listGraphs();
console.log('Available graphs:', graphs);
// Output: ['graph1', 'graph2', 'socialNetwork']
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:53](../../../src/services/falkordb.service.ts#L53)

---

##### `close()`

**Signature**:
```typescript
async close(): Promise<void>
```

**Description**: Gracefully closes the FalkorDB client connection and sets the client to null. Safe to call multiple times.

**Parameters**: None

**Returns**:
- `Promise<void>` - Resolves when the connection is closed

**Example**:
```typescript
// Called during graceful shutdown
process.on('SIGTERM', async () => {
  await falkorDBService.close();
  process.exit(0);
});
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:67](../../../src/services/falkordb.service.ts#L67)

---

##### `init()` (Private)

**Signature**:
```typescript
private async init(): Promise<void>
```

**Description**: Initializes the FalkorDB connection with automatic retry logic. Attempts to connect using configuration from environment variables and tests the connection with a ping. If connection fails, retries after 5 seconds.

**Parameters**: None

**Returns**:
- `Promise<void>` - Resolves when connection is established and tested

**Connection Configuration**:
- Uses `config.falkorDB.host`, `config.falkorDB.port`, `config.falkorDB.username`, and `config.falkorDB.password`
- Automatically retries on failure with 5-second delay

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:11](../../../src/services/falkordb.service.ts#L11)

---

### Singleton Export

The service is exported as a singleton instance for application-wide use:

```typescript
export const falkorDBService = new FalkorDBService();
```

**Usage**:
```typescript
import { falkorDBService } from './services/falkordb.service';

// Service is ready to use immediately
const graphs = await falkorDBService.listGraphs();
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:76](../../../src/services/falkordb.service.ts#L76)

---

## Controllers

### MCPController

**File**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:10](../../../src/controllers/mcp.controller.ts#L10)

**Description**: Handles MCP protocol requests, providing endpoints for context queries, metadata retrieval, and graph listing. Formats all responses according to MCP standards with appropriate metadata.

#### Methods

##### `processContextRequest()`

**Signature**:
```typescript
async processContextRequest(
  req: Request,
  res: Response
): Promise<Response<any, Record<string, any>>>
```

**Description**: Processes MCP context requests by executing graph queries and returning formatted results with timing metadata.

**Parameters**:
- `req` (Request) - Express request object containing the MCP context request in the body
- `res` (Response) - Express response object for sending the response

**Request Body** (`MCPContextRequest`):
```typescript
{
  graphName: string;      // Required: Name of the graph to query
  query: string;          // Required: Cypher query to execute
  params?: object;        // Optional: Query parameters
  context?: object;       // Optional: Additional context
  options?: object;       // Optional: Query options (timeout, maxResults)
}
```

**Returns**:
- `Promise<Response>` - HTTP response with status code and JSON body

**Success Response** (200):
```typescript
{
  data: any,              // Query results from FalkorDB
  metadata: {
    timestamp: string,    // ISO timestamp
    queryTime: number,    // Execution time in milliseconds
    provider: string,     // "FalkorDB MCP Server"
    source: string        // "falkordb"
  }
}
```

**Error Responses**:
- 400: Missing query or graph name
- 500: Query execution error

**Example**:
```typescript
// Request
POST /api/mcp/context
{
  "graphName": "socialNetwork",
  "query": "MATCH (p:Person)-[:KNOWS]->(f:Person) RETURN p, f LIMIT 5"
}

// Response
{
  "data": {
    "records": [...],
    "statistics": {...}
  },
  "metadata": {
    "timestamp": "2025-11-29T17:30:00.000Z",
    "queryTime": 45,
    "provider": "FalkorDB MCP Server",
    "source": "falkordb"
  }
}
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:14](../../../src/controllers/mcp.controller.ts#L14)

---

##### `processMetadataRequest()`

**Signature**:
```typescript
async processMetadataRequest(
  req: Request,
  res: Response
): Promise<Response<any, Record<string, any>>>
```

**Description**: Returns metadata about the FalkorDB MCP Server, including supported capabilities, graph types, and query languages.

**Parameters**:
- `req` (Request) - Express request object
- `res` (Response) - Express response object

**Returns**:
- `Promise<Response>` - HTTP response with provider metadata

**Response** (200):
```typescript
{
  provider: "FalkorDB MCP Server",
  version: "1.0.0",
  capabilities: [
    "graph.query",
    "graph.list",
    "node.properties",
    "relationship.properties"
  ],
  graphTypes: ["property", "directed"],
  queryLanguages: ["cypher"]
}
```

**Example**:
```typescript
// Request
GET /api/mcp/metadata

// Response
{
  "provider": "FalkorDB MCP Server",
  "version": "1.0.0",
  "capabilities": ["graph.query", "graph.list", "node.properties", "relationship.properties"],
  "graphTypes": ["property", "directed"],
  "queryLanguages": ["cypher"]
}
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:64](../../../src/controllers/mcp.controller.ts#L64)

---

##### `listGraphs()`

**Signature**:
```typescript
async listGraphs(
  req: Request,
  res: Response
): Promise<Response<any, Record<string, any>>>
```

**Description**: Lists all available graphs in the connected FalkorDB instance with metadata about the count.

**Parameters**:
- `req` (Request) - Express request object
- `res` (Response) - Express response object

**Returns**:
- `Promise<Response>` - HTTP response with graph list

**Response** (200):
```typescript
{
  data: [
    { name: "graph1" },
    { name: "graph2" },
    { name: "socialNetwork" }
  ],
  metadata: {
    timestamp: string,
    count: number
  }
}
```

**Example**:
```typescript
// Request
GET /api/mcp/graphs

// Response
{
  "data": [
    { "name": "socialNetwork" },
    { "name": "knowledgeGraph" },
    { "name": "recommendations" }
  ],
  "metadata": {
    "timestamp": "2025-11-29T17:30:00.000Z",
    "count": 3
  }
}
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:90](../../../src/controllers/mcp.controller.ts#L90)

---

### Controller Export

The controller is exported as a singleton instance:

```typescript
export const mcpController = new MCPController();
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:116](../../../src/controllers/mcp.controller.ts#L116)

---

## Middleware

### Authentication Middleware

**File**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts:7](../../../src/middleware/auth.middleware.ts#L7)

**Description**: Express middleware that validates API keys for MCP requests. Supports both header-based and query parameter authentication.

#### `authenticateMCP`

**Signature**:
```typescript
function authenticateMCP(
  req: Request,
  res: Response,
  next: NextFunction
): Response<any, Record<string, any>> | void
```

**Description**: Validates the API key from either the `x-api-key` header or `apiKey` query parameter. Skips authentication in development mode if no API key is configured.

**Parameters**:
- `req` (Request) - Express request object
- `res` (Response) - Express response object for sending error responses
- `next` (NextFunction) - Express next function to continue the middleware chain

**Returns**:
- `Response | void` - Returns error response if authentication fails, otherwise calls `next()`

**Behavior**:
- **Development Mode**: If `NODE_ENV=development` and `MCP_API_KEY` is not set, authentication is skipped with a warning
- **Missing Key**: Returns 401 if no API key is provided
- **Invalid Key**: Returns 403 if the API key doesn't match the configured value
- **Valid Key**: Calls `next()` to proceed to the route handler

**Authentication Methods**:
1. Header: `x-api-key: your-api-key`
2. Query Parameter: `?apiKey=your-api-key`

**Error Responses**:
- 401 Unauthorized: `{ "error": "Missing API key" }`
- 403 Forbidden: `{ "error": "Invalid API key" }`

**Example**:
```typescript
// Using in routes
import { authenticateMCP } from './middleware/auth.middleware';

app.use('/api/mcp', authenticateMCP, mcpRoutes);

// Client request with header
fetch('http://localhost:3000/api/mcp/context', {
  method: 'POST',
  headers: {
    'x-api-key': 'your-api-key',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ graphName: 'test', query: 'MATCH (n) RETURN n' })
});

// Client request with query parameter
fetch('http://localhost:3000/api/mcp/graphs?apiKey=your-api-key');
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts:7](../../../src/middleware/auth.middleware.ts#L7)

---

## Models & Types

### MCP Protocol Types

**File**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts](../../../src/models/mcp.types.ts)

**Description**: TypeScript interfaces and types that define the Model Context Protocol request and response structures.

---

#### `MCPContextRequest`

**Description**: Interface for MCP context query requests.

**Properties**:
```typescript
interface MCPContextRequest {
  graphName: string;                  // Required: Target graph name
  query: string;                      // Required: Cypher query to execute
  params?: Record<string, any>;       // Optional: Query parameters for parameterization
  context?: Record<string, any>;      // Optional: Additional context information
  options?: MCPOptions;               // Optional: Query execution options
}
```

**Usage Example**:
```typescript
const request: MCPContextRequest = {
  graphName: 'socialNetwork',
  query: 'MATCH (p:Person {name: $name}) RETURN p',
  params: { name: 'Alice' },
  options: { timeout: 5000, maxResults: 100 }
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:5](../../../src/models/mcp.types.ts#L5)

---

#### `MCPOptions`

**Description**: Options for configuring query execution behavior.

**Properties**:
```typescript
interface MCPOptions {
  timeout?: number;                   // Query timeout in milliseconds
  maxResults?: number;                // Maximum number of results to return
  [key: string]: any;                 // Additional custom options
}
```

**Usage Example**:
```typescript
const options: MCPOptions = {
  timeout: 10000,      // 10 second timeout
  maxResults: 1000     // Limit to 1000 results
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:13](../../../src/models/mcp.types.ts#L13)

---

#### `MCPResponse`

**Description**: Standard response format for MCP operations.

**Properties**:
```typescript
interface MCPResponse {
  data: any;                          // Query results or operation output
  metadata: MCPMetadata;              // Response metadata
}
```

**Usage Example**:
```typescript
const response: MCPResponse = {
  data: { records: [...], statistics: {...} },
  metadata: {
    timestamp: new Date().toISOString(),
    queryTime: 45,
    provider: 'FalkorDB MCP Server',
    source: 'falkordb'
  }
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:19](../../../src/models/mcp.types.ts#L19)

---

#### `MCPMetadata`

**Description**: Metadata included with all MCP responses.

**Properties**:
```typescript
interface MCPMetadata {
  timestamp: string;                  // ISO 8601 timestamp
  queryTime: number;                  // Execution time in milliseconds
  provider?: string;                  // Provider name
  source?: string;                    // Data source identifier
  [key: string]: any;                 // Additional custom metadata
}
```

**Usage Example**:
```typescript
const metadata: MCPMetadata = {
  timestamp: '2025-11-29T17:30:00.000Z',
  queryTime: 142,
  provider: 'FalkorDB MCP Server',
  source: 'falkordb',
  graphName: 'socialNetwork'
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:24](../../../src/models/mcp.types.ts#L24)

---

#### `MCPProviderMetadata`

**Description**: Metadata describing the MCP provider's capabilities and supported features.

**Properties**:
```typescript
interface MCPProviderMetadata {
  provider: string;                   // Provider name
  version: string;                    // Provider version
  capabilities: string[];             // Supported capabilities
  graphTypes: string[];               // Supported graph types
  queryLanguages: string[];           // Supported query languages
  [key: string]: any;                 // Additional custom metadata
}
```

**Usage Example**:
```typescript
const providerMetadata: MCPProviderMetadata = {
  provider: 'FalkorDB MCP Server',
  version: '1.0.0',
  capabilities: [
    'graph.query',
    'graph.list',
    'node.properties',
    'relationship.properties'
  ],
  graphTypes: ['property', 'directed'],
  queryLanguages: ['cypher']
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:32](../../../src/models/mcp.types.ts#L32)

---

### Client Configuration Types

**File**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp-client-config.ts](../../../src/models/mcp-client-config.ts)

**Description**: Configuration interfaces for MCP server and client setups.

---

#### `MCPServerConfig`

**Description**: Configuration for running the MCP server in different deployment modes.

**Properties**:
```typescript
interface MCPServerConfig {
  mcpServers: {
    [key: string]: {
      command: string;                // Command to run the server
      args: string[];                 // Command arguments
    };
  };
}
```

**Usage Example**:
```typescript
const serverConfig: MCPServerConfig = {
  mcpServers: {
    "falkordb": {
      command: "docker",
      args: [
        "run", "-i", "--rm",
        "-p", "3000:3000",
        "--env-file", ".env",
        "falkordb-mcpserver",
        "falkordb://host.docker.internal:6379"
      ]
    }
  }
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp-client-config.ts:5](../../../src/models/mcp-client-config.ts#L5)

---

#### `MCPClientConfig`

**Description**: Client-side configuration for connecting to MCP servers.

**Properties**:
```typescript
interface MCPClientConfig {
  defaultServer?: string;             // Default server to use
  servers: {
    [key: string]: {
      url: string;                    // Server URL
      apiKey?: string;                // Optional API key
    };
  };
}
```

**Usage Example**:
```typescript
const clientConfig: MCPClientConfig = {
  defaultServer: "falkordb",
  servers: {
    "falkordb": {
      url: "http://localhost:3000/api/mcp",
      apiKey: "your_api_key_here"
    },
    "production": {
      url: "https://api.example.com/mcp",
      apiKey: "prod_api_key"
    }
  }
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp-client-config.ts:14](../../../src/models/mcp-client-config.ts#L14)

---

#### Sample Configurations

**Sample MCP Client Config**:
```typescript
export const sampleMCPClientConfig: MCPClientConfig = {
  defaultServer: "falkordb",
  servers: {
    "falkordb": {
      url: "http://localhost:3000/api/mcp",
      apiKey: "your_api_key_here"
    }
  }
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp-client-config.ts:27](../../../src/models/mcp-client-config.ts#L27)

**Sample MCP Server Config**:
```typescript
export const sampleMCPServerConfig: MCPServerConfig = {
  mcpServers: {
    "falkordb": {
      command: "docker",
      args: [
        "run", "-i", "--rm",
        "-p", "3000:3000",
        "--env-file", ".env",
        "falkordb-mcpserver",
        "falkordb://host.docker.internal:6379"
      ]
    }
  }
};
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp-client-config.ts:40](../../../src/models/mcp-client-config.ts#L40)

---

## Utilities

### Connection Parser

**File**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/utils/connection-parser.ts](../../../src/utils/connection-parser.ts)

**Description**: Utility for parsing FalkorDB connection strings into structured configuration objects.

---

#### `parseFalkorDBConnectionString()`

**Signature**:
```typescript
function parseFalkorDBConnectionString(
  connectionString: string
): FalkorDBConnectionOptions
```

**Description**: Parses a FalkorDB connection string in the format `falkordb://[username:password@]host:port` and returns a structured configuration object. Handles missing or malformed input gracefully with sensible defaults.

**Parameters**:
- `connectionString` (string) - Connection string to parse. Format: `falkordb://[username:password@]host:port`

**Returns**:
- `FalkorDBConnectionOptions` - Parsed connection configuration

**Connection String Format**:
```
falkordb://[username:password@]host:port
```

**Components**:
- `falkordb://` - Protocol prefix (optional)
- `username:password@` - Authentication credentials (optional)
- `host` - Database host (required, defaults to 'localhost')
- `port` - Database port (optional, defaults to 6379)

**Return Type**:
```typescript
interface FalkorDBConnectionOptions {
  host: string;
  port: number;
  username?: string;
  password?: string;
}
```

**Examples**:
```typescript
// Full connection string with authentication
const config1 = parseFalkorDBConnectionString(
  'falkordb://admin:secret@db.example.com:6379'
);
// Result: { host: 'db.example.com', port: 6379, username: 'admin', password: 'secret' }

// Connection string without authentication
const config2 = parseFalkorDBConnectionString(
  'falkordb://localhost:6379'
);
// Result: { host: 'localhost', port: 6379 }

// Connection string with password only
const config3 = parseFalkorDBConnectionString(
  'falkordb://mypassword@localhost:6379'
);
// Result: { host: 'localhost', port: 6379, password: 'mypassword' }

// Without protocol prefix
const config4 = parseFalkorDBConnectionString(
  'db.example.com:7000'
);
// Result: { host: 'db.example.com', port: 7000 }

// Host only (uses default port)
const config5 = parseFalkorDBConnectionString(
  'falkordb://myhost'
);
// Result: { host: 'myhost', port: 6379 }

// Empty or invalid input (returns defaults)
const config6 = parseFalkorDBConnectionString('');
// Result: { host: 'localhost', port: 6379 }
```

**Error Handling**:
- Gracefully handles malformed input
- Returns default values on parsing errors
- Logs errors to console without throwing

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/utils/connection-parser.ts:19](../../../src/utils/connection-parser.ts#L19)

---

#### `FalkorDBConnectionOptions`

**Description**: Interface for FalkorDB connection configuration.

**Properties**:
```typescript
interface FalkorDBConnectionOptions {
  host: string;        // Database host
  port: number;        // Database port
  username?: string;   // Optional username
  password?: string;   // Optional password
}
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/utils/connection-parser.ts:5](../../../src/utils/connection-parser.ts#L5)

---

## Configuration

### Environment Variables

**File**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts](../../../src/config/index.ts)

**Description**: Centralized configuration management using environment variables loaded via dotenv.

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `PORT` | number | No | 3000 | Port number for the HTTP server |
| `NODE_ENV` | string | No | 'development' | Node environment (development, production, test) |
| `FALKORDB_HOST` | string | No | 'localhost' | FalkorDB server hostname or IP address |
| `FALKORDB_PORT` | number | No | 6379 | FalkorDB server port number |
| `FALKORDB_USERNAME` | string | No | '' | Username for FalkorDB authentication (if required) |
| `FALKORDB_PASSWORD` | string | No | '' | Password for FalkorDB authentication (if required) |
| `MCP_API_KEY` | string | Yes* | '' | API key for authenticating MCP requests (*skipped in dev mode) |

**Configuration Object**:
```typescript
export const config = {
  server: {
    port: process.env.PORT || 3000,
    nodeEnv: process.env.NODE_ENV || 'development',
  },
  falkorDB: {
    host: process.env.FALKORDB_HOST || 'localhost',
    port: parseInt(process.env.FALKORDB_PORT || '6379'),
    username: process.env.FALKORDB_USERNAME || '',
    password: process.env.FALKORDB_PASSWORD || '',
  },
  mcp: {
    apiKey: process.env.MCP_API_KEY || '',
  },
};
```

**Configuration Loading**:
```typescript
import dotenv from 'dotenv';

// Load environment variables from .env file
dotenv.config();

// Access configuration
import { config } from './config';

console.log('Server port:', config.server.port);
console.log('FalkorDB host:', config.falkorDB.host);
```

**Example `.env` file**:
```env
# Server Configuration
PORT=3000
NODE_ENV=development

# FalkorDB Configuration
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
FALKORDB_USERNAME=
FALKORDB_PASSWORD=

# MCP Configuration
MCP_API_KEY=your_mcp_api_key_here
```

**Example `.env` file for production**:
```env
# Server Configuration
PORT=3000
NODE_ENV=production

# FalkorDB Configuration
FALKORDB_HOST=falkordb.production.example.com
FALKORDB_PORT=6379
FALKORDB_USERNAME=admin
FALKORDB_PASSWORD=secure_password_here

# MCP Configuration
MCP_API_KEY=production_api_key_here
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts:6](../../../src/config/index.ts#L6)

---

## REST API Endpoints

### Base URL
```
http://localhost:3000
```

### Authentication
All `/api/mcp/*` endpoints require authentication via API key.

**Methods**:
1. Header: `x-api-key: your-api-key`
2. Query Parameter: `?apiKey=your-api-key`

---

### GET /

**Description**: Health check endpoint returning basic server information.

**Authentication**: None

**Response**:
```typescript
Status: 200 OK
{
  "name": "FalkorDB MCP Server",
  "version": "1.0.0",
  "status": "running"
}
```

**Example**:
```bash
curl http://localhost:3000/
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts:18](../../../src/index.ts#L18)

---

### POST /api/mcp/context

**Description**: Execute a Cypher query against a specified graph in FalkorDB.

**Authentication**: Required (API Key)

**Request**:
```typescript
POST /api/mcp/context
Headers:
  x-api-key: your-api-key
  Content-Type: application/json

Body:
{
  "graphName": "socialNetwork",
  "query": "MATCH (n:Person) RETURN n LIMIT 10",
  "params": {},                          // Optional
  "context": {},                         // Optional
  "options": {                           // Optional
    "timeout": 5000,
    "maxResults": 100
  }
}
```

**Success Response**:
```typescript
Status: 200 OK
{
  "data": {
    "records": [
      // Query results
    ],
    "statistics": {
      "nodesCreated": 0,
      "nodesDeleted": 0,
      // ...
    }
  },
  "metadata": {
    "timestamp": "2025-11-29T17:30:00.000Z",
    "queryTime": 45,
    "provider": "FalkorDB MCP Server",
    "source": "falkordb"
  }
}
```

**Error Responses**:
```typescript
// Missing query
Status: 400 Bad Request
{
  "error": "Query is required"
}

// Missing graph name
Status: 400 Bad Request
{
  "error": "Graph name is required"
}

// Query execution error
Status: 500 Internal Server Error
{
  "error": "Graph 'socialNetwork' not found",
  "metadata": {
    "timestamp": "2025-11-29T17:30:00.000Z"
  }
}

// Authentication error
Status: 401 Unauthorized
{
  "error": "Missing API key"
}
```

**Examples**:
```bash
# Simple query
curl -X POST http://localhost:3000/api/mcp/context \
  -H "x-api-key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "graphName": "myGraph",
    "query": "MATCH (n) RETURN n LIMIT 5"
  }'

# Parameterized query
curl -X POST http://localhost:3000/api/mcp/context \
  -H "x-api-key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "graphName": "socialNetwork",
    "query": "MATCH (p:Person {name: $name}) RETURN p",
    "params": {
      "name": "Alice"
    }
  }'

# With options
curl -X POST http://localhost:3000/api/mcp/context \
  -H "x-api-key: your-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "graphName": "knowledgeGraph",
    "query": "MATCH (n)-[r]->(m) RETURN n, r, m",
    "options": {
      "timeout": 10000,
      "maxResults": 1000
    }
  }'
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts:7](../../../src/routes/mcp.routes.ts#L7)

---

### GET /api/mcp/metadata

**Description**: Get metadata about the FalkorDB MCP Server, including capabilities and supported features.

**Authentication**: Required (API Key)

**Request**:
```typescript
GET /api/mcp/metadata
Headers:
  x-api-key: your-api-key
```

**Response**:
```typescript
Status: 200 OK
{
  "provider": "FalkorDB MCP Server",
  "version": "1.0.0",
  "capabilities": [
    "graph.query",
    "graph.list",
    "node.properties",
    "relationship.properties"
  ],
  "graphTypes": ["property", "directed"],
  "queryLanguages": ["cypher"]
}
```

**Example**:
```bash
curl http://localhost:3000/api/mcp/metadata \
  -H "x-api-key: your-api-key"
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts:8](../../../src/routes/mcp.routes.ts#L8)

---

### GET /api/mcp/graphs

**Description**: List all available graphs in the connected FalkorDB instance.

**Authentication**: Required (API Key)

**Request**:
```typescript
GET /api/mcp/graphs
Headers:
  x-api-key: your-api-key
```

**Response**:
```typescript
Status: 200 OK
{
  "data": [
    { "name": "socialNetwork" },
    { "name": "knowledgeGraph" },
    { "name": "recommendations" }
  ],
  "metadata": {
    "timestamp": "2025-11-29T17:30:00.000Z",
    "count": 3
  }
}
```

**Example**:
```bash
curl http://localhost:3000/api/mcp/graphs \
  -H "x-api-key: your-api-key"

# Alternative: Using query parameter for API key
curl "http://localhost:3000/api/mcp/graphs?apiKey=your-api-key"
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts:9](../../../src/routes/mcp.routes.ts#L9)

---

### GET /api/mcp/health

**Description**: Health check endpoint for the MCP API.

**Authentication**: Required (API Key)

**Request**:
```typescript
GET /api/mcp/health
Headers:
  x-api-key: your-api-key
```

**Response**:
```typescript
Status: 200 OK
{
  "status": "ok"
}
```

**Example**:
```bash
curl http://localhost:3000/api/mcp/health \
  -H "x-api-key: your-api-key"
```

**Source**: [/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts:12](../../../src/routes/mcp.routes.ts#L12)

---

## Usage Examples

### Quick Start

**1. Installation and Setup**:
```bash
# Clone the repository
git clone https://github.com/falkordb/falkordb-mcpserver.git
cd falkordb-mcpserver

# Install dependencies
npm install

# Create environment file
cp .env.example .env

# Edit .env with your configuration
nano .env
```

**2. Configure Environment**:
```env
PORT=3000
NODE_ENV=development
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
MCP_API_KEY=my-secret-api-key
```

**3. Start the Server**:
```bash
# Development mode (with hot reload)
npm run dev

# Production mode
npm run build
npm start
```

**4. Test the API**:
```bash
# Check server status
curl http://localhost:3000/

# List graphs (requires API key)
curl http://localhost:3000/api/mcp/graphs \
  -H "x-api-key: my-secret-api-key"
```

---

### Common Usage Patterns

#### Pattern 1: Executing a Simple Query

**Scenario**: Query all Person nodes in a social network graph.

```typescript
// Using fetch API
const response = await fetch('http://localhost:3000/api/mcp/context', {
  method: 'POST',
  headers: {
    'x-api-key': 'my-secret-api-key',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    graphName: 'socialNetwork',
    query: 'MATCH (p:Person) RETURN p.name, p.age LIMIT 10'
  })
});

const result = await response.json();
console.log('Query results:', result.data);
console.log('Execution time:', result.metadata.queryTime, 'ms');
```

**cURL equivalent**:
```bash
curl -X POST http://localhost:3000/api/mcp/context \
  -H "x-api-key: my-secret-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "graphName": "socialNetwork",
    "query": "MATCH (p:Person) RETURN p.name, p.age LIMIT 10"
  }'
```

---

#### Pattern 2: Parameterized Query for Security

**Scenario**: Find a specific person by name using parameters to prevent injection attacks.

```typescript
const findPerson = async (name: string) => {
  const response = await fetch('http://localhost:3000/api/mcp/context', {
    method: 'POST',
    headers: {
      'x-api-key': 'my-secret-api-key',
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      graphName: 'socialNetwork',
      query: 'MATCH (p:Person {name: $name})-[:KNOWS]->(friend) RETURN p, friend',
      params: { name }
    })
  });

  return await response.json();
};

// Usage
const result = await findPerson('Alice');
```

**cURL equivalent**:
```bash
curl -X POST http://localhost:3000/api/mcp/context \
  -H "x-api-key: my-secret-api-key" \
  -H "Content-Type: application/json" \
  -d '{
    "graphName": "socialNetwork",
    "query": "MATCH (p:Person {name: $name})-[:KNOWS]->(friend) RETURN p, friend",
    "params": {
      "name": "Alice"
    }
  }'
```

---

#### Pattern 3: Listing Available Graphs

**Scenario**: Discover all available graphs before querying.

```typescript
const listGraphs = async () => {
  const response = await fetch('http://localhost:3000/api/mcp/graphs', {
    method: 'GET',
    headers: {
      'x-api-key': 'my-secret-api-key'
    }
  });

  const result = await response.json();
  return result.data.map(g => g.name);
};

// Usage
const graphs = await listGraphs();
console.log('Available graphs:', graphs);
// Output: ['socialNetwork', 'knowledgeGraph', 'recommendations']
```

**cURL equivalent**:
```bash
curl http://localhost:3000/api/mcp/graphs \
  -H "x-api-key: my-secret-api-key"
```

---

#### Pattern 4: Error Handling

**Scenario**: Robust error handling for production applications.

```typescript
const executeGraphQuery = async (graphName: string, query: string) => {
  try {
    const response = await fetch('http://localhost:3000/api/mcp/context', {
      method: 'POST',
      headers: {
        'x-api-key': 'my-secret-api-key',
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({ graphName, query })
    });

    // Check for HTTP errors
    if (!response.ok) {
      const error = await response.json();
      throw new Error(`API Error (${response.status}): ${error.error}`);
    }

    const result = await response.json();

    // Validate response structure
    if (!result.data || !result.metadata) {
      throw new Error('Invalid response format');
    }

    return result;

  } catch (error) {
    if (error instanceof TypeError) {
      console.error('Network error:', error.message);
      throw new Error('Failed to connect to MCP server');
    } else if (error instanceof Error) {
      console.error('Query error:', error.message);
      throw error;
    } else {
      console.error('Unknown error:', error);
      throw new Error('An unexpected error occurred');
    }
  }
};

// Usage with error handling
try {
  const result = await executeGraphQuery(
    'socialNetwork',
    'MATCH (n) RETURN n LIMIT 5'
  );
  console.log('Success:', result.data);
} catch (error) {
  console.error('Failed to execute query:', error.message);
  // Handle error appropriately (show user message, retry, etc.)
}
```

---

#### Pattern 5: Getting Server Metadata

**Scenario**: Check server capabilities before sending requests.

```typescript
const checkServerCapabilities = async () => {
  const response = await fetch('http://localhost:3000/api/mcp/metadata', {
    method: 'GET',
    headers: {
      'x-api-key': 'my-secret-api-key'
    }
  });

  const metadata = await response.json();

  // Check if server supports required capabilities
  const hasQueryCapability = metadata.capabilities.includes('graph.query');
  const supportsCypher = metadata.queryLanguages.includes('cypher');

  console.log('Server capabilities:', metadata.capabilities);
  console.log('Can execute queries:', hasQueryCapability);
  console.log('Supports Cypher:', supportsCypher);

  return metadata;
};

// Usage
const metadata = await checkServerCapabilities();
if (metadata.capabilities.includes('graph.list')) {
  // Server supports listing graphs
  const graphs = await listGraphs();
}
```

---

### Integration Examples

#### Using with TypeScript/JavaScript MCP Client

```typescript
import { MCPContextRequest, MCPResponse } from './models/mcp.types';

class FalkorDBMCPClient {
  private baseUrl: string;
  private apiKey: string;

  constructor(baseUrl: string, apiKey: string) {
    this.baseUrl = baseUrl;
    this.apiKey = apiKey;
  }

  async query(
    graphName: string,
    query: string,
    params?: Record<string, any>
  ): Promise<MCPResponse> {
    const request: MCPContextRequest = {
      graphName,
      query,
      params
    };

    const response = await fetch(`${this.baseUrl}/context`, {
      method: 'POST',
      headers: {
        'x-api-key': this.apiKey,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(request)
    });

    if (!response.ok) {
      const error = await response.json();
      throw new Error(error.error);
    }

    return await response.json();
  }

  async listGraphs(): Promise<string[]> {
    const response = await fetch(`${this.baseUrl}/graphs`, {
      headers: { 'x-api-key': this.apiKey }
    });

    const result = await response.json();
    return result.data.map((g: any) => g.name);
  }

  async getMetadata() {
    const response = await fetch(`${this.baseUrl}/metadata`, {
      headers: { 'x-api-key': this.apiKey }
    });

    return await response.json();
  }
}

// Usage
const client = new FalkorDBMCPClient(
  'http://localhost:3000/api/mcp',
  'my-secret-api-key'
);

// Execute queries
const result = await client.query(
  'socialNetwork',
  'MATCH (p:Person) RETURN p LIMIT 5'
);

// List graphs
const graphs = await client.listGraphs();

// Get capabilities
const metadata = await client.getMetadata();
```

---

#### Docker Deployment

**Build and run with Docker**:

```bash
# Build the Docker image
docker build -t falkordb-mcpserver .

# Run the container
docker run -d \
  --name falkordb-mcp \
  -p 3000:3000 \
  -e FALKORDB_HOST=host.docker.internal \
  -e FALKORDB_PORT=6379 \
  -e MCP_API_KEY=your-api-key \
  falkordb-mcpserver

# Or use environment file
docker run -d \
  --name falkordb-mcp \
  -p 3000:3000 \
  --env-file .env \
  falkordb-mcpserver

# Check logs
docker logs falkordb-mcp

# Stop the container
docker stop falkordb-mcp
```

**Docker Compose**:

```yaml
# docker-compose.yml
version: '3.8'

services:
  falkordb:
    image: falkordb/falkordb:latest
    ports:
      - "6379:6379"
    volumes:
      - falkordb-data:/data

  mcp-server:
    build: .
    ports:
      - "3000:3000"
    environment:
      - FALKORDB_HOST=falkordb
      - FALKORDB_PORT=6379
      - MCP_API_KEY=your-api-key
      - NODE_ENV=production
    depends_on:
      - falkordb

volumes:
  falkordb-data:
```

**Run with Docker Compose**:
```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f mcp-server

# Stop all services
docker-compose down
```

---

#### Using from Python

```python
import requests
import json

class FalkorDBMCPClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.headers = {
            'x-api-key': api_key,
            'Content-Type': 'application/json'
        }

    def query(self, graph_name: str, query: str, params: dict = None):
        """Execute a Cypher query"""
        payload = {
            'graphName': graph_name,
            'query': query,
            'params': params or {}
        }

        response = requests.post(
            f'{self.base_url}/context',
            headers=self.headers,
            json=payload
        )
        response.raise_for_status()
        return response.json()

    def list_graphs(self):
        """List available graphs"""
        response = requests.get(
            f'{self.base_url}/graphs',
            headers=self.headers
        )
        response.raise_for_status()
        result = response.json()
        return [g['name'] for g in result['data']]

    def get_metadata(self):
        """Get server metadata"""
        response = requests.get(
            f'{self.base_url}/metadata',
            headers=self.headers
        )
        response.raise_for_status()
        return response.json()

# Usage
client = FalkorDBMCPClient(
    'http://localhost:3000/api/mcp',
    'my-secret-api-key'
)

# Execute query
result = client.query(
    'socialNetwork',
    'MATCH (p:Person) RETURN p.name LIMIT 5'
)
print(f"Query results: {result['data']}")
print(f"Execution time: {result['metadata']['queryTime']}ms")

# List graphs
graphs = client.list_graphs()
print(f"Available graphs: {graphs}")
```

---

## Best Practices

### 1. Connection Management

**Always use the singleton service instance**:
```typescript
// Good
import { falkorDBService } from './services/falkordb.service';
const graphs = await falkorDBService.listGraphs();

// Bad - Don't create new instances
const service = new FalkorDBService(); // Creates duplicate connection
```

**Implement graceful shutdown**:
```typescript
// In your main application file
process.on('SIGTERM', async () => {
  console.log('Shutting down gracefully...');
  await falkorDBService.close();
  process.exit(0);
});

process.on('SIGINT', async () => {
  console.log('Shutting down gracefully...');
  await falkorDBService.close();
  process.exit(0);
});
```

**Let the service handle reconnection**:
- The service automatically retries failed connections
- Don't manually call `init()` - it's called automatically
- Monitor logs for connection issues

---

### 2. Error Handling

**Always validate input before querying**:
```typescript
// Good
if (!graphName || !query) {
  return res.status(400).json({ error: 'Graph name and query are required' });
}

// Handle specific errors
try {
  const result = await falkorDBService.executeQuery(graphName, query);
} catch (error) {
  if (error.message.includes('not initialized')) {
    return res.status(503).json({ error: 'Service temporarily unavailable' });
  }
  // Handle other errors
}
```

**Use parameterized queries to prevent injection**:
```typescript
// Good - Parameterized query
const query = 'MATCH (p:Person {name: $name}) RETURN p';
const params = { name: userInput };
await falkorDBService.executeQuery(graphName, query, params);

// Bad - String interpolation (vulnerable to injection)
const query = `MATCH (p:Person {name: '${userInput}'}) RETURN p`;
await falkorDBService.executeQuery(graphName, query);
```

**Provide meaningful error messages**:
```typescript
// Good
catch (error: any) {
  console.error('Query execution failed:', error);
  return res.status(500).json({
    error: 'Failed to execute query',
    message: error.message,
    metadata: {
      timestamp: new Date().toISOString(),
      graphName,
      queryLength: query.length
    }
  });
}

// Bad - Generic error
catch (error) {
  return res.status(500).json({ error: 'Error' });
}
```

---

### 3. Security

**Always require API keys in production**:
```env
# Production .env
NODE_ENV=production
MCP_API_KEY=your-strong-random-api-key-here
```

**Use strong, random API keys**:
```bash
# Generate a secure API key
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

**Never commit API keys to version control**:
```gitignore
# .gitignore
.env
.env.local
.env.production
```

**Use HTTPS in production**:
```typescript
// Use a reverse proxy like nginx or traefik
// Configure SSL/TLS certificates
// Redirect HTTP to HTTPS
```

**Implement rate limiting** (recommended addition):
```typescript
import rateLimit from 'express-rate-limit';

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

app.use('/api/mcp', limiter, authenticateMCP, mcpRoutes);
```

**Sanitize log output**:
```typescript
// Good - Sanitize before logging
const sanitizedGraphName = graphName.replace(/\n|\r/g, "");
console.error('Error on graph %s:', sanitizedGraphName, error);

// Bad - Direct logging of user input
console.error('Error on graph', graphName, error);
```

---

### 4. Query Optimization

**Use LIMIT clauses to prevent large result sets**:
```cypher
-- Good
MATCH (n:Person) RETURN n LIMIT 100

-- Bad - Could return millions of nodes
MATCH (n:Person) RETURN n
```

**Create indexes for frequently queried properties**:
```cypher
-- In FalkorDB, create indexes to improve query performance
CREATE INDEX FOR (p:Person) ON (p.name)
CREATE INDEX FOR (p:Person) ON (p.email)
```

**Use query parameters instead of string interpolation**:
```typescript
// Good - Parameters are cached and optimized
const query = 'MATCH (p:Person {age: $age}) RETURN p';
const params = { age: 25 };

// Bad - Creates a new query plan each time
const query = `MATCH (p:Person {age: ${age}}) RETURN p`;
```

**Profile slow queries**:
```cypher
-- Use PROFILE to analyze query performance
PROFILE MATCH (p:Person)-[:KNOWS]->(f:Person) RETURN p, f
```

**Avoid Cartesian products**:
```cypher
-- Good - Specific relationship
MATCH (p:Person)-[:KNOWS]->(f:Person) RETURN p, f

-- Bad - Cartesian product
MATCH (p:Person), (f:Person) WHERE p.city = f.city RETURN p, f
```

---

### 5. Testing

**Test with the provided test suite**:
```bash
# Run all tests
npm test

# Run with coverage
npm run test:coverage

# Run specific test file
npm test -- mcp.controller.test.ts
```

**Mock the FalkorDB service in tests**:
```typescript
// Example from the test suite
jest.mock('../services/falkordb.service', () => ({
  falkorDBService: {
    executeQuery: jest.fn(),
    listGraphs: jest.fn()
  }
}));
```

**Test error scenarios**:
```typescript
it('should handle database errors gracefully', async () => {
  (falkorDBService.executeQuery as jest.Mock).mockRejectedValue(
    new Error('Connection timeout')
  );

  const response = await request(app)
    .post('/api/mcp/context')
    .send({ graphName: 'test', query: 'MATCH (n) RETURN n' });

  expect(response.status).toBe(500);
  expect(response.body.error).toBeDefined();
});
```

---

## Appendix

### Type Definitions Summary

| Type | File | Purpose |
|------|------|---------|
| `MCPContextRequest` | [mcp.types.ts:5](../../../src/models/mcp.types.ts#L5) | Request format for context queries |
| `MCPOptions` | [mcp.types.ts:13](../../../src/models/mcp.types.ts#L13) | Query execution options |
| `MCPResponse` | [mcp.types.ts:19](../../../src/models/mcp.types.ts#L19) | Standard response format |
| `MCPMetadata` | [mcp.types.ts:24](../../../src/models/mcp.types.ts#L24) | Response metadata structure |
| `MCPProviderMetadata` | [mcp.types.ts:32](../../../src/models/mcp.types.ts#L32) | Provider capabilities |
| `MCPServerConfig` | [mcp-client-config.ts:5](../../../src/models/mcp-client-config.ts#L5) | Server deployment config |
| `MCPClientConfig` | [mcp-client-config.ts:14](../../../src/models/mcp-client-config.ts#L14) | Client connection config |
| `FalkorDBConnectionOptions` | [connection-parser.ts:5](../../../src/utils/connection-parser.ts#L5) | Database connection options |

---

### Error Codes

| Status | Error | Cause | Solution |
|--------|-------|-------|----------|
| 400 | "Query is required" | Missing query in request body | Include `query` field in request |
| 400 | "Graph name is required" | Missing graphName in request body | Include `graphName` field in request |
| 401 | "Missing API key" | No API key provided | Add `x-api-key` header or `apiKey` query param |
| 403 | "Invalid API key" | API key doesn't match configured value | Use correct API key from `.env` file |
| 500 | "FalkorDB client not initialized" | Database connection not established | Check FalkorDB is running and connection config is correct |
| 500 | Graph-specific errors | Query syntax errors, missing graph, etc. | Review error message and fix query/graph name |
| 503 | Service unavailable | Server is starting or shutting down | Wait and retry request |

---

### Environment Configuration Examples

**Development (Local FalkorDB)**:
```env
PORT=3000
NODE_ENV=development
FALKORDB_HOST=localhost
FALKORDB_PORT=6379
FALKORDB_USERNAME=
FALKORDB_PASSWORD=
MCP_API_KEY=dev-api-key
```

**Production (Remote FalkorDB with Auth)**:
```env
PORT=3000
NODE_ENV=production
FALKORDB_HOST=falkordb.production.example.com
FALKORDB_PORT=6379
FALKORDB_USERNAME=production_user
FALKORDB_PASSWORD=secure_password_123
MCP_API_KEY=prod-api-key-abc123def456
```

**Docker Deployment**:
```env
PORT=3000
NODE_ENV=production
FALKORDB_HOST=host.docker.internal
FALKORDB_PORT=6379
FALKORDB_USERNAME=
FALKORDB_PASSWORD=
MCP_API_KEY=docker-api-key
```

---

### Cypher Query Examples

**Basic Pattern Matching**:
```cypher
-- Find all nodes
MATCH (n) RETURN n LIMIT 10

-- Find nodes by label
MATCH (p:Person) RETURN p

-- Find nodes with property
MATCH (p:Person {name: 'Alice'}) RETURN p

-- Find relationships
MATCH (p:Person)-[r:KNOWS]->(f:Person) RETURN p, r, f
```

**Parameterized Queries**:
```cypher
-- Using parameters (recommended)
MATCH (p:Person {name: $name}) RETURN p

-- Multiple parameters
MATCH (p:Person {name: $name, age: $age}) RETURN p

-- Parameter in WHERE clause
MATCH (p:Person) WHERE p.age > $minAge RETURN p
```

**Creating Data**:
```cypher
-- Create node
CREATE (p:Person {name: 'Bob', age: 30}) RETURN p

-- Create relationship
MATCH (a:Person {name: 'Alice'}), (b:Person {name: 'Bob'})
CREATE (a)-[r:KNOWS]->(b)
RETURN a, r, b
```

**Aggregation**:
```cypher
-- Count nodes
MATCH (p:Person) RETURN count(p)

-- Group by property
MATCH (p:Person) RETURN p.city, count(p) as population

-- Average, sum, etc.
MATCH (p:Person) RETURN avg(p.age), sum(p.age), max(p.age)
```

---

### Project Structure

```
FalkorDB-MCPServer/
├── src/
│   ├── config/
│   │   └── index.ts              # Configuration management
│   ├── controllers/
│   │   ├── mcp.controller.ts     # MCP request handlers
│   │   └── mcp.controller.test.ts
│   ├── middleware/
│   │   └── auth.middleware.ts    # API key authentication
│   ├── models/
│   │   ├── mcp.types.ts          # MCP type definitions
│   │   └── mcp-client-config.ts  # Client configuration types
│   ├── routes/
│   │   └── mcp.routes.ts         # API route definitions
│   ├── services/
│   │   ├── falkordb.service.ts   # FalkorDB connection service
│   │   └── falkordb.service.test.ts
│   ├── utils/
│   │   └── connection-parser.ts  # Connection string parser
│   └── index.ts                  # Application entry point
├── .env.example                  # Example environment variables
├── Dockerfile                    # Docker container definition
├── package.json                  # Node.js dependencies
├── tsconfig.json                 # TypeScript configuration
└── README.md                     # Project documentation
```

---

### Additional Resources

**FalkorDB Documentation**:
- [FalkorDB Official Docs](https://docs.falkordb.com/)
- [Cypher Query Language](https://neo4j.com/docs/cypher-manual/current/)
- [FalkorDB GitHub](https://github.com/FalkorDB/FalkorDB)

**Model Context Protocol**:
- [MCP SDK Documentation](https://github.com/modelcontextprotocol/sdk)
- [MCP Specification](https://spec.modelcontextprotocol.io/)

**Related Technologies**:
- [Express.js Documentation](https://expressjs.com/)
- [TypeScript Documentation](https://www.typescriptlang.org/docs/)
- [Docker Documentation](https://docs.docker.com/)

---

### Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0.0 | 2025-11-29 | Initial API documentation release |

---

### Support

For issues, questions, or contributions:
- GitHub: [https://github.com/FalkorDB/FalkorDB-MCPServer](https://github.com/FalkorDB/FalkorDB-MCPServer)
- Issues: [Submit an issue](https://github.com/FalkorDB/FalkorDB-MCPServer/issues)

---

**Document Generated**: 2025-11-29
**API Version**: 1.0.0
**Documentation Version**: 1.0.0
