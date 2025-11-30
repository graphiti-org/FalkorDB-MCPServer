# Data Flow Analysis

## Overview

The FalkorDB MCP Server is a Model Context Protocol (MCP) compliant REST API server that provides a bridge between MCP clients and FalkorDB graph database. The system implements several key data flows:

1. **Graph Query Execution Flow** - Processing Cypher queries against FalkorDB graphs
2. **Metadata Retrieval Flow** - Providing server capabilities and graph information
3. **Graph Listing Flow** - Enumerating available graphs in the database
4. **Authentication Flow** - Validating API key-based authentication
5. **Error Handling Flow** - Managing and reporting errors across the system
6. **Database Connection Management Flow** - Initializing and maintaining FalkorDB connections
7. **Server Lifecycle Flow** - Application startup and graceful shutdown

Each flow follows a layered architecture pattern: Express middleware -> Routes -> Controllers -> Services -> FalkorDB client.

---

## 1. Graph Query Execution Flow

### Description

This is the primary data flow for executing Cypher queries against a specified FalkorDB graph. A client sends an MCP context request containing a graph name, Cypher query, and optional parameters. The request passes through authentication, is routed to the controller, validated, executed against FalkorDB, and returns results with metadata.

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Express
    participant AuthMiddleware
    participant MCPRoutes
    participant MCPController
    participant FalkorDBService
    participant FalkorDB

    Client->>Express: POST /api/mcp/context
    Note over Client,Express: Headers: x-api-key<br/>Body: {graphName, query, params}

    Express->>AuthMiddleware: authenticateMCP()
    AuthMiddleware->>AuthMiddleware: Validate API key
    alt Invalid/Missing API Key
        AuthMiddleware-->>Client: 401/403 Error
    else Valid API Key
        AuthMiddleware->>MCPRoutes: next()
        MCPRoutes->>MCPController: processContextRequest()

        MCPController->>MCPController: Validate request body
        alt Missing query
            MCPController-->>Client: 400 "Query is required"
        else Missing graphName
            MCPController-->>Client: 400 "Graph name is required"
        else Valid Request
            MCPController->>MCPController: Record start time
            MCPController->>FalkorDBService: executeQuery(graphName, query, params)

            FalkorDBService->>FalkorDBService: Check client initialized
            alt Client not initialized
                FalkorDBService-->>MCPController: Error "Client not initialized"
            else Client ready
                FalkorDBService->>FalkorDBService: selectGraph(graphName)
                FalkorDBService->>FalkorDB: graph.query(query, params)
                FalkorDB-->>FalkorDBService: Query results
                FalkorDBService-->>MCPController: Return results

                MCPController->>MCPController: Calculate query time
                MCPController->>MCPController: Format MCP response
                Note over MCPController: Add metadata:<br/>- timestamp<br/>- queryTime<br/>- provider<br/>- source
                MCPController-->>Client: 200 OK {data, metadata}
            end
        end
    end

    alt Error during execution
        MCPController->>MCPController: Log error
        MCPController-->>Client: 500 {error, metadata}
    end
```

### Flow Steps

1. **Request Reception** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, lines 11-15)
   - Express server receives POST request at `/api/mcp/context`
   - JSON middleware parses request body
   - Authentication middleware is applied to all `/api/mcp/*` routes

2. **Authentication** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`, lines 7-24)
   - Extracts API key from `x-api-key` header or `apiKey` query parameter
   - In development mode without configured API key, authentication is skipped with warning
   - Validates API key against configured value
   - Returns 401 if missing, 403 if invalid, or calls `next()` if valid

3. **Route Handling** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts`, line 7)
   - Router matches POST `/context` endpoint
   - Binds request to `mcpController.processContextRequest` method

4. **Request Validation** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 16-25)
   - Controller validates presence of `query` field (returns 400 if missing)
   - Validates presence of `graphName` field (returns 400 if missing)
   - Extracts optional `params` object from request

5. **Query Execution** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 27-34)
   - Records start time for performance tracking
   - Calls `falkorDBService.executeQuery()` with graphName, query, and params
   - Service validates client is initialized (throws error if not)
   - Selects the specified graph from FalkorDB
   - Executes Cypher query with parameters
   - Returns raw query results

6. **Response Formatting** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 36-49)
   - Calculates query execution time
   - Wraps results in MCP response format with metadata
   - Metadata includes: timestamp, queryTime, provider name, source identifier
   - Returns 200 status with formatted response

7. **Error Handling** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 50-58)
   - Catches any errors during execution
   - Logs error to console with sanitized graph name
   - Returns 500 status with error message and timestamp metadata

### Key Points

- **Graph name sanitization**: The service sanitizes graph names by removing newlines/carriage returns to prevent log injection attacks (line 43 of falkordb.service.ts)
- **Performance tracking**: Every query execution is timed and included in response metadata
- **MCP compliance**: Response format follows Model Context Protocol standards with data and metadata separation
- **Connection pooling**: Service maintains a singleton FalkorDB client instance across requests
- **Graceful degradation**: Development mode allows bypassing authentication for easier testing
- **Type safety**: Full TypeScript typing ensures request/response contract compliance

### Code References

- Request entry point: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts:15`
- Authentication middleware: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts:7-24`
- Route definition: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts:7`
- Controller logic: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:14-58`
- Service implementation: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:33-46`
- Type definitions: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:5-30`

---

## 2. Metadata Retrieval Flow

### Description

This flow handles requests for server metadata and capabilities. Clients can query the server to discover what features are supported, what query languages are available, and what graph types are supported. This is essential for MCP protocol compliance and client capability negotiation.

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Express
    participant AuthMiddleware
    participant MCPRoutes
    participant MCPController

    Client->>Express: GET /api/mcp/metadata
    Note over Client,Express: Headers: x-api-key

    Express->>AuthMiddleware: authenticateMCP()
    AuthMiddleware->>AuthMiddleware: Validate API key

    alt Authentication fails
        AuthMiddleware-->>Client: 401/403 Error
    else Authentication succeeds
        AuthMiddleware->>MCPRoutes: next()
        MCPRoutes->>MCPController: processMetadataRequest()

        MCPController->>MCPController: Build metadata object
        Note over MCPController: Capabilities:<br/>- graph.query<br/>- graph.list<br/>- node.properties<br/>- relationship.properties
        Note over MCPController: Graph Types:<br/>- property<br/>- directed
        Note over MCPController: Query Languages:<br/>- cypher

        MCPController-->>Client: 200 OK {provider, version, capabilities, graphTypes, queryLanguages}
    end

    alt Error during processing
        MCPController->>MCPController: Log error
        MCPController-->>Client: 500 {error}
    end
```

### Flow Steps

1. **Request Reception** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, line 15)
   - Express receives GET request at `/api/mcp/metadata`
   - Authentication middleware processes the request

2. **Authentication** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`, lines 7-24)
   - Same authentication flow as query execution
   - Validates API key from header or query parameter

3. **Route Handling** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts`, line 8)
   - Router matches GET `/metadata` endpoint
   - Binds to `mcpController.processMetadataRequest` method

4. **Metadata Construction** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 66-78)
   - Controller builds static metadata object
   - Includes provider name: "FalkorDB MCP Server"
   - Version: "1.0.0"
   - Capabilities array listing supported operations
   - Graph types: property graphs and directed graphs
   - Query languages: Cypher

5. **Response** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, line 80)
   - Returns 200 status with metadata object
   - No database interaction required - purely informational

6. **Error Handling** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 81-84)
   - Catches any errors (unlikely for static data)
   - Logs and returns 500 with error message

### Key Points

- **Static data**: Metadata is hardcoded and doesn't require database queries
- **MCP discovery**: Clients use this endpoint to understand server capabilities
- **Version tracking**: Server version is explicitly declared for client compatibility checks
- **Capability advertisement**: Lists all supported MCP operations (graph.query, graph.list, node.properties, relationship.properties)
- **No side effects**: Pure read operation with no state changes
- **Fast response**: No I/O operations, immediate response

### Code References

- Route definition: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts:8`
- Controller implementation: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:64-85`
- Type definition: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:32-38`

---

## 3. Graph Listing Flow

### Description

This flow retrieves a list of all available graphs in the connected FalkorDB instance. It allows clients to discover what graphs exist before executing queries. The flow queries the database for graph names and returns them in a structured format with metadata.

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Express
    participant AuthMiddleware
    participant MCPRoutes
    participant MCPController
    participant FalkorDBService
    participant FalkorDB

    Client->>Express: GET /api/mcp/graphs
    Note over Client,Express: Headers: x-api-key

    Express->>AuthMiddleware: authenticateMCP()
    AuthMiddleware->>AuthMiddleware: Validate API key

    alt Authentication fails
        AuthMiddleware-->>Client: 401/403 Error
    else Authentication succeeds
        AuthMiddleware->>MCPRoutes: next()
        MCPRoutes->>MCPController: listGraphs()

        MCPController->>FalkorDBService: listGraphs()

        FalkorDBService->>FalkorDBService: Check client initialized
        alt Client not initialized
            FalkorDBService-->>MCPController: Error "Client not initialized"
        else Client ready
            FalkorDBService->>FalkorDB: client.list()
            FalkorDB-->>FalkorDBService: Array of graph names
            FalkorDBService-->>MCPController: Return graph names

            MCPController->>MCPController: Map names to objects
            Note over MCPController: Transform:<br/>["graph1", "graph2"]<br/>to<br/>[{name: "graph1"}, {name: "graph2"}]

            MCPController->>MCPController: Add metadata
            Note over MCPController: Metadata:<br/>- timestamp<br/>- count

            MCPController-->>Client: 200 OK {data: graphs[], metadata}
        end
    end

    alt Error during execution
        MCPController->>MCPController: Log error
        MCPController-->>Client: 500 {error}
    end
```

### Flow Steps

1. **Request Reception** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, line 15)
   - Express receives GET request at `/api/mcp/graphs`
   - Authentication middleware processes the request

2. **Authentication** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`, lines 7-24)
   - Validates API key using standard authentication flow
   - Ensures only authorized clients can list graphs

3. **Route Handling** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts`, line 9)
   - Router matches GET `/graphs` endpoint
   - Binds to `mcpController.listGraphs` method

4. **Service Call** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, line 92)
   - Controller calls `falkorDBService.listGraphs()`
   - No parameters required

5. **Database Query** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`, lines 53-64)
   - Service checks client is initialized (throws if not)
   - Calls FalkorDB `client.list()` method
   - Returns array of graph name strings
   - Errors are logged and re-thrown

6. **Data Transformation** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 94-100)
   - Maps simple string array to structured objects
   - Each graph name becomes an object: `{name: "graphName"}`
   - Placeholder for future enhancement (additional metadata per graph)

7. **Response Formatting** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 102-108)
   - Wraps graph array in `data` field
   - Adds metadata with timestamp and count
   - Returns 200 status with structured response

8. **Error Handling** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts`, lines 109-112)
   - Catches errors from service layer
   - Logs error details
   - Returns 500 with error message

### Key Points

- **Discovery mechanism**: Essential for clients to know what graphs are available
- **Future extensibility**: Object structure allows adding per-graph metadata later
- **Count tracking**: Response includes total count for client convenience
- **Database agnostic**: Uses FalkorDB's native list method
- **Error propagation**: Database errors are properly caught and returned to client
- **Structured response**: Consistent with other endpoints using data/metadata pattern

### Code References

- Route definition: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts:9`
- Controller implementation: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:90-113`
- Service implementation: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:53-64`

---

## 4. Authentication Flow

### Description

The authentication flow is implemented as Express middleware that validates API key-based authentication for all MCP endpoints. It supports both header-based and query parameter authentication with special handling for development environments.

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Express
    participant AuthMiddleware
    participant Config
    participant NextHandler

    Client->>Express: HTTP Request to /api/mcp/*
    Note over Client,Express: Headers: x-api-key: "key123"<br/>OR<br/>Query: ?apiKey=key123

    Express->>AuthMiddleware: authenticateMCP(req, res, next)

    AuthMiddleware->>AuthMiddleware: Extract API key
    Note over AuthMiddleware: Check headers['x-api-key']<br/>OR query.apiKey

    AuthMiddleware->>Config: Get environment config
    Config-->>AuthMiddleware: nodeEnv, apiKey

    alt Development mode AND no API key configured
        AuthMiddleware->>AuthMiddleware: Log warning
        Note over AuthMiddleware: "Running without API key<br/>authentication in dev mode"
        AuthMiddleware->>NextHandler: next()
        NextHandler-->>Express: Continue to route handler
    else API key not provided
        AuthMiddleware-->>Client: 401 {error: "Missing API key"}
    else API key invalid
        AuthMiddleware->>Config: Compare apiKey
        AuthMiddleware-->>Client: 403 {error: "Invalid API key"}
    else API key valid
        AuthMiddleware->>Config: Compare apiKey
        AuthMiddleware->>NextHandler: next()
        NextHandler-->>Express: Continue to route handler
    end
```

### Flow Steps

1. **Middleware Registration** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, line 15)
   - Authentication middleware is applied to all `/api/mcp` routes
   - Executes before any route handler

2. **API Key Extraction** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`, line 8)
   - Checks `x-api-key` header first
   - Falls back to `apiKey` query parameter
   - Supports both authentication methods for flexibility

3. **Development Mode Check** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`, lines 11-14)
   - Checks if running in development environment
   - If no API key is configured in development, logs warning and skips authentication
   - Allows easier local development and testing
   - Production environments must have API key configured

4. **Presence Validation** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`, lines 16-18)
   - Returns 401 Unauthorized if no API key provided
   - Clear error message: "Missing API key"
   - Stops request processing immediately

5. **Key Validation** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`, lines 20-22)
   - Compares provided key against configured key from environment
   - Returns 403 Forbidden if keys don't match
   - Clear error message: "Invalid API key"

6. **Authorization Success** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts`, line 24)
   - Calls `next()` to continue to route handler
   - Request proceeds to controller layer

### Key Points

- **Dual authentication methods**: Supports both header and query parameter for client flexibility
- **Security by default**: Production requires API key, no bypass option
- **Development convenience**: Optional authentication in dev mode with clear warnings
- **Clear error codes**: 401 for missing credentials, 403 for invalid credentials
- **Simple but effective**: Suitable for server-to-server communication
- **Environment-driven**: Configuration loaded from environment variables
- **Fail-safe**: Missing configuration in production prevents unauthorized access

### Code References

- Middleware implementation: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts:7-25`
- Middleware registration: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts:15`
- Configuration: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts:17-19`

---

## 5. Error Handling Flow

### Description

Error handling is implemented at multiple layers of the application, ensuring errors are caught, logged, and returned to clients in a consistent format. The system handles validation errors, database errors, and unexpected exceptions.

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Client
    participant Express
    participant Middleware/Controller
    participant FalkorDBService
    participant FalkorDB

    Note over Client,FalkorDB: Scenario 1: Validation Error
    Client->>Express: POST /api/mcp/context {invalid data}
    Express->>Middleware/Controller: processContextRequest()
    Middleware/Controller->>Middleware/Controller: Validate request
    alt Missing required field
        Middleware/Controller->>Middleware/Controller: Detect validation error
        Middleware/Controller-->>Client: 400 {error: "Field is required"}
    end

    Note over Client,FalkorDB: Scenario 2: Database Connection Error
    Client->>Express: POST /api/mcp/context {valid data}
    Express->>Middleware/Controller: processContextRequest()
    Middleware/Controller->>FalkorDBService: executeQuery()
    FalkorDBService->>FalkorDBService: Check client initialized
    alt Client not initialized
        FalkorDBService->>FalkorDBService: throw Error
        FalkorDBService-->>Middleware/Controller: Error: "Client not initialized"
        Middleware/Controller->>Middleware/Controller: Log error
        Middleware/Controller-->>Client: 500 {error: message, metadata}
    end

    Note over Client,FalkorDB: Scenario 3: Query Execution Error
    Client->>Express: POST /api/mcp/context {valid data}
    Express->>Middleware/Controller: processContextRequest()
    Middleware/Controller->>FalkorDBService: executeQuery()
    FalkorDBService->>FalkorDB: graph.query()
    FalkorDB-->>FalkorDBService: Error (invalid query syntax)
    FalkorDBService->>FalkorDBService: Log error (sanitized)
    FalkorDBService-->>Middleware/Controller: throw Error
    Middleware/Controller->>Middleware/Controller: Log error
    Middleware/Controller-->>Client: 500 {error: message, metadata}

    Note over Client,FalkorDB: Scenario 4: Unexpected Error
    Client->>Express: POST /api/mcp/context {valid data}
    Express->>Middleware/Controller: processContextRequest()
    Middleware/Controller->>Middleware/Controller: Unexpected error occurs
    Middleware/Controller->>Middleware/Controller: Catch in try/catch
    Middleware/Controller->>Middleware/Controller: Log full error
    Middleware/Controller-->>Client: 500 {error: error.message, metadata}
```

### Flow Steps

1. **Validation Errors** (Multiple locations)
   - **Missing query**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:18-20`
     - Detected before any processing
     - Returns 400 with descriptive error
   - **Missing graphName**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:23-25`
     - Detected before database access
     - Returns 400 with descriptive error
   - **Missing/Invalid API key**: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts:16-22`
     - Returns 401/403 as appropriate

2. **Service Layer Errors** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`)
   - **Client not initialized**: Lines 34-36, 54-56
     - Checked before any database operation
     - Throws error with clear message
     - Propagates to controller layer
   - **Query execution errors**: Lines 42-46
     - Sanitizes graph name to prevent log injection (removes newlines)
     - Logs error with sanitized context
     - Re-throws error to controller

3. **Controller Error Handling** (All controller methods)
   - **processContextRequest**: Lines 50-58
     - Try/catch wraps all processing logic
     - Logs error to console with full context
     - Returns 500 with error message and timestamp metadata
   - **processMetadataRequest**: Lines 81-84
     - Same pattern for consistency
   - **listGraphs**: Lines 109-112
     - Same pattern for consistency

4. **Error Response Format**
   - All error responses include timestamp in metadata when applicable
   - Error messages preserve original error text
   - Status codes follow REST conventions (400, 401, 403, 500)
   - JSON format for programmatic consumption

5. **Logging Strategy**
   - Errors logged at the point of detection
   - Includes context (operation, parameters)
   - Sanitized output prevents injection attacks
   - Console logging (can be extended to logging service)

### Key Points

- **Defense in depth**: Validation at multiple layers (middleware, controller, service)
- **Fail fast**: Validation errors caught before expensive operations
- **Error context**: Logs include relevant details for debugging
- **Security**: Graph names sanitized to prevent log injection attacks
- **Consistency**: All endpoints follow same error handling pattern
- **Client-friendly**: Clear error messages help developers debug issues
- **Separation of concerns**: Validation in controllers, technical errors in services
- **No silent failures**: All errors are logged and returned to client

### Code References

- Authentication errors: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts:16-22`
- Validation errors: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:18-25`
- Service errors: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:34-36, 42-46, 54-56, 61-63`
- Controller error handling: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts:50-58, 81-84, 109-112`

---

## 6. Database Connection Management Flow

### Description

The FalkorDB service manages database connections through a singleton pattern with automatic initialization and retry logic. The connection is established on service instantiation and maintained throughout the application lifecycle with graceful shutdown support.

### Sequence Diagram

```mermaid
sequenceDiagram
    participant App
    participant FalkorDBService
    participant FalkorDB
    participant Config
    participant Process

    Note over App,Process: Application Startup
    App->>FalkorDBService: Import service (singleton)
    FalkorDBService->>FalkorDBService: constructor()
    FalkorDBService->>FalkorDBService: init()

    FalkorDBService->>Config: Load connection config
    Config-->>FalkorDBService: host, port, username, password

    FalkorDBService->>FalkorDB: FalkorDB.connect(config)

    alt Connection successful
        FalkorDB-->>FalkorDBService: client instance
        FalkorDBService->>FalkorDB: client.connection
        FalkorDB-->>FalkorDBService: connection object
        FalkorDBService->>FalkorDB: connection.ping()
        FalkorDB-->>FalkorDBService: pong
        FalkorDBService->>FalkorDBService: Log "Successfully connected"
        FalkorDBService-->>App: Service ready
    else Connection failed
        FalkorDB-->>FalkorDBService: Error
        FalkorDBService->>FalkorDBService: Log error
        FalkorDBService->>FalkorDBService: setTimeout(init, 5000)
        Note over FalkorDBService: Retry after 5 seconds
    end

    Note over App,Process: During Operation
    App->>FalkorDBService: executeQuery() / listGraphs()
    FalkorDBService->>FalkorDBService: Check client != null
    alt Client initialized
        FalkorDBService->>FalkorDB: Perform operation
        FalkorDB-->>FalkorDBService: Result
        FalkorDBService-->>App: Return result
    else Client not initialized
        FalkorDBService-->>App: throw Error
    end

    Note over App,Process: Graceful Shutdown
    Process->>App: SIGTERM / SIGINT
    App->>App: Log shutdown message
    App->>FalkorDBService: close()
    FalkorDBService->>FalkorDBService: Check client != null
    alt Client exists
        FalkorDBService->>FalkorDB: client.close()
        FalkorDB-->>FalkorDBService: Connection closed
        FalkorDBService->>FalkorDBService: Set client = null
    end
    FalkorDBService-->>App: Close complete
    App->>Process: process.exit(0)
```

### Flow Steps

1. **Service Instantiation** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`, lines 75-76)
   - Singleton instance created at module load time
   - Constructor called automatically
   - Private client property initialized to null

2. **Constructor Initialization** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`, lines 7-9)
   - Constructor calls private `init()` method
   - Initialization happens asynchronously
   - Service available immediately (operations check client state)

3. **Connection Setup** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`, lines 11-31)
   - Loads configuration from environment (host, port, username, password)
   - Calls `FalkorDB.connect()` with socket and credentials
   - Awaits connection object
   - Performs ping test to verify connectivity
   - Logs success message

4. **Connection Retry Logic** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`, lines 26-30)
   - If connection fails, error is logged
   - Sets 5-second timeout to retry initialization
   - Continues retrying until successful
   - Application remains running but operations will fail gracefully

5. **Operation Validation** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`, lines 34-36, 54-56)
   - Every operation checks if client is initialized
   - Throws descriptive error if client is null
   - Prevents operations during initialization or after shutdown

6. **Graceful Shutdown** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, lines 34-46)
   - Application listens for SIGTERM and SIGINT signals
   - Logs shutdown message
   - Calls `falkorDBService.close()`
   - Closes FalkorDB client connection
   - Sets client to null
   - Exits process with code 0

7. **Close Method** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts`, lines 67-72)
   - Checks if client exists
   - Calls `client.close()` if present
   - Sets client to null to prevent further use
   - Safe to call multiple times (idempotent)

### Key Points

- **Singleton pattern**: Single connection shared across all requests
- **Automatic retry**: Failed connections retry every 5 seconds indefinitely
- **Health check**: Ping after connection ensures database is responsive
- **Graceful degradation**: Application starts even if database unavailable
- **Clean shutdown**: Properly closes connections on SIGTERM/SIGINT
- **Thread safety**: Single-threaded Node.js eliminates concurrency issues
- **Configuration-driven**: Connection parameters from environment variables
- **Error transparency**: Clear error messages when client not initialized
- **Resource cleanup**: Proper disposal of database connections

### Code References

- Service class: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:4-76`
- Initialization: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:11-31`
- Connection retry: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:26-30`
- Shutdown handlers: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts:34-46`
- Close method: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:67-72`
- Configuration: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts:11-16`

---

## 7. Server Lifecycle Flow

### Description

The server lifecycle encompasses application startup, middleware setup, route registration, server listening, and graceful shutdown. This flow shows how the Express application is bootstrapped and managed throughout its lifetime.

### Sequence Diagram

```mermaid
sequenceDiagram
    participant Node
    participant App
    participant Express
    participant Config
    participant AuthMiddleware
    participant MCPRoutes
    participant FalkorDBService
    participant Process

    Note over Node,Process: Application Startup
    Node->>App: Load index.ts
    App->>Config: Import config
    Config->>Config: Load .env file
    Config-->>App: Configuration object

    App->>FalkorDBService: Import service
    FalkorDBService->>FalkorDBService: Initialize singleton
    Note over FalkorDBService: Async connection to FalkorDB<br/>(runs in background)

    App->>Express: Create Express app
    Express-->>App: app instance

    App->>Express: Register JSON middleware
    App->>Express: Register URL-encoded middleware

    App->>Express: Register MCP routes with auth
    Note over App,Express: app.use('/api/mcp',<br/>authenticateMCP,<br/>mcpRoutes)

    App->>Express: Register root route
    Note over Express: GET / returns server info

    App->>Config: Get PORT
    Config-->>App: PORT value

    App->>Express: app.listen(PORT)
    Express->>Express: Bind to port
    Express-->>App: Server listening
    App->>App: Log startup message
    Note over App: "FalkorDB MCP Server<br/>listening on port ${PORT}"

    Note over Node,Process: During Operation
    loop Handle Requests
        Express->>AuthMiddleware: Authenticate
        AuthMiddleware->>MCPRoutes: Route request
        MCPRoutes->>FalkorDBService: Execute operation
        FalkorDBService-->>Express: Return response
    end

    Note over Node,Process: Graceful Shutdown
    Process->>App: SIGTERM signal
    App->>App: Log "SIGTERM received"
    App->>FalkorDBService: close()
    FalkorDBService->>FalkorDBService: Close DB connection
    FalkorDBService-->>App: Closed
    App->>Process: process.exit(0)

    alt Alternative shutdown
        Process->>App: SIGINT signal (Ctrl+C)
        App->>App: Log "SIGINT received"
        App->>FalkorDBService: close()
        FalkorDBService-->>App: Closed
        App->>Process: process.exit(0)
    end
```

### Flow Steps

1. **Module Loading** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, lines 1-6)
   - Node.js loads index.ts entry point
   - Imports Express framework
   - Imports configuration module (which loads .env)
   - Imports routes module
   - Imports authentication middleware
   - Imports FalkorDB service (triggers singleton initialization)

2. **Configuration Loading** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts`, lines 1-20)
   - dotenv.config() loads environment variables from .env file
   - Configuration object created with defaults
   - Server port, FalkorDB connection details, API key loaded
   - Exported for use throughout application

3. **Express Initialization** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, line 8)
   - Creates Express application instance
   - Returns configured app object

4. **Middleware Registration** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, lines 11-12)
   - `express.json()`: Parses JSON request bodies
   - `express.urlencoded({ extended: true })`: Parses URL-encoded bodies
   - Global middleware applied to all routes

5. **Route Registration** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, line 15)
   - Mounts MCP routes at `/api/mcp` path
   - Authentication middleware applied to all MCP routes
   - Routes imported from separate module for organization

6. **Root Route** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, lines 18-24)
   - GET `/` endpoint for health/info check
   - Returns server name, version, and status
   - No authentication required for basic info

7. **Server Start** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, lines 27-31)
   - Retrieves port from configuration
   - Calls `app.listen(PORT)` to bind to port
   - Logs startup message with port and environment
   - Server now accepting connections

8. **Signal Handlers** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, lines 34-46)
   - Registers SIGTERM handler (graceful shutdown from orchestrator)
   - Registers SIGINT handler (Ctrl+C from terminal)
   - Both handlers follow same pattern:
     - Log shutdown message
     - Close database connections
     - Exit with code 0 (success)

9. **Module Export** (`/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts`, line 48)
   - Exports app for testing purposes
   - Allows test suites to import and test the application

### Key Points

- **Clean separation**: Configuration, routes, middleware, and services in separate modules
- **Middleware order**: Body parsing before routes, authentication before MCP endpoints
- **Health endpoint**: Unauthenticated root endpoint for monitoring
- **Graceful shutdown**: Properly closes database connections before exit
- **Signal handling**: Responds to both SIGTERM (containers) and SIGINT (manual)
- **Environment-driven**: All configuration from environment variables
- **Testability**: App exported for integration testing
- **Logging**: Clear startup and shutdown logging for operations
- **Background initialization**: Database connection happens asynchronously without blocking startup

### Code References

- Main entry point: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts:1-48`
- Configuration loading: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts:1-20`
- Route definitions: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts:1-16`
- Authentication middleware: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts:1-25`
- Service initialization: `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts:75-76`

---

## Data Flow Patterns Summary

### Common Patterns

1. **Request/Response Pattern**
   - All endpoints follow consistent structure: validation → processing → response formatting
   - Error responses always include error message and metadata
   - Success responses wrap data in standardized format with metadata

2. **Layered Architecture**
   - Clear separation: Middleware → Routes → Controllers → Services → Database
   - Each layer has single responsibility
   - Dependencies flow downward only

3. **Error Propagation**
   - Errors caught at source (service layer)
   - Logged with context
   - Re-thrown to controller
   - Caught again, formatted, and returned to client
   - No error silencing

4. **Singleton Services**
   - Database service is singleton
   - Single connection shared across requests
   - Reduces connection overhead
   - Simplified resource management

5. **Async/Await Pattern**
   - All I/O operations use async/await
   - Consistent error handling with try/catch
   - Promise-based flow control
   - No callback hell

6. **Configuration Injection**
   - Environment variables as single source of configuration
   - Centralized config module
   - Defaults provided for development
   - Type-safe configuration access

### Best Practices Observed

1. **Security**
   - API key authentication on all MCP endpoints
   - Log injection prevention (sanitized graph names)
   - Development/production mode separation
   - No credentials in code (environment variables)

2. **Reliability**
   - Automatic connection retry with exponential backoff
   - Graceful shutdown handling
   - Client initialization checks before operations
   - Health check ping after connection

3. **Observability**
   - Comprehensive error logging
   - Performance timing (query execution time)
   - Startup/shutdown logging
   - Request validation logging

4. **Maintainability**
   - TypeScript for type safety
   - Clear module boundaries
   - Single responsibility principle
   - Comprehensive test coverage

5. **API Design**
   - RESTful endpoint design
   - Consistent response formats
   - Clear HTTP status codes
   - Metadata included with all responses

6. **Testing**
   - Unit tests for controllers and services
   - Mocked dependencies
   - Test coverage for error paths
   - Exported app for integration testing

### Areas for Optimization

1. **Connection Pooling**
   - Current implementation uses single connection
   - Could benefit from connection pool under high load
   - FalkorDB SDK may support this natively

2. **Rate Limiting**
   - No rate limiting currently implemented
   - Could add middleware for request throttling
   - Protect against abuse and DoS

3. **Request Validation**
   - Basic validation in controllers
   - Could use JSON schema validation (like express-validator)
   - More comprehensive input sanitization

4. **Logging Enhancement**
   - Currently using console.log
   - Could integrate structured logging (Winston, Pino)
   - Log levels, rotation, external aggregation

5. **Caching**
   - No caching for graph list or metadata
   - Could cache static/semi-static data
   - Reduce database load for frequent queries

6. **Health Checks**
   - Basic health endpoint exists
   - Could add detailed health check (database connectivity, memory, etc.)
   - Readiness vs liveness probes for Kubernetes

7. **Metrics**
   - Query timing captured but not aggregated
   - Could add Prometheus metrics
   - Request counts, error rates, latency percentiles

8. **Async Initialization**
   - Service initialization happens in background
   - First requests may fail if database not ready
   - Could add readiness check or startup probe

---

## Appendix: Data Structures

### Request Formats

#### MCPContextRequest
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:5-11
interface MCPContextRequest {
  graphName: string;        // Required: Name of the graph to query
  query: string;            // Required: Cypher query to execute
  params?: Record<string, any>;  // Optional: Query parameters
  context?: Record<string, any>; // Optional: Additional context
  options?: MCPOptions;     // Optional: Query options
}
```

**Example:**
```json
{
  "graphName": "social_network",
  "query": "MATCH (p:Person {name: $name})-[:FRIENDS_WITH]->(friend) RETURN friend",
  "params": {
    "name": "Alice"
  },
  "options": {
    "timeout": 5000,
    "maxResults": 100
  }
}
```

#### MCPOptions
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:13-17
interface MCPOptions {
  timeout?: number;         // Query timeout in milliseconds
  maxResults?: number;      // Maximum number of results to return
  [key: string]: any;       // Extensible for future options
}
```

### Response Formats

#### MCPResponse
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:19-22
interface MCPResponse {
  data: any;                // Query results or response data
  metadata: MCPMetadata;    // Response metadata
}
```

**Example Success Response:**
```json
{
  "data": {
    "records": [
      {
        "friend": {
          "identity": 2,
          "labels": ["Person"],
          "properties": {
            "name": "Bob",
            "age": 30
          }
        }
      }
    ]
  },
  "metadata": {
    "timestamp": "2025-11-29T17:30:00.000Z",
    "queryTime": 45,
    "provider": "FalkorDB MCP Server",
    "source": "falkordb"
  }
}
```

**Example Error Response:**
```json
{
  "error": "Invalid query syntax near 'METCH'",
  "metadata": {
    "timestamp": "2025-11-29T17:30:00.000Z"
  }
}
```

#### MCPMetadata
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:24-30
interface MCPMetadata {
  timestamp: string;        // ISO 8601 timestamp
  queryTime: number;        // Execution time in milliseconds
  provider?: string;        // Provider name
  source?: string;          // Data source identifier
  [key: string]: any;       // Extensible for additional metadata
}
```

#### MCPProviderMetadata
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts:32-38
interface MCPProviderMetadata {
  provider: string;         // Provider name
  version: string;          // Server version
  capabilities: string[];   // Supported operations
  graphTypes: string[];     // Supported graph types
  queryLanguages: string[]; // Supported query languages
  [key: string]: any;       // Extensible
}
```

**Example:**
```json
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

#### Graph List Response
```json
{
  "data": [
    {"name": "social_network"},
    {"name": "knowledge_graph"},
    {"name": "product_catalog"}
  ],
  "metadata": {
    "timestamp": "2025-11-29T17:30:00.000Z",
    "count": 3
  }
}
```

### Internal Data Types

#### FalkorDB Connection Configuration
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts:11-16
const falkorDB = {
  host: string;      // FalkorDB host (default: 'localhost')
  port: number;      // FalkorDB port (default: 6379)
  username: string;  // Optional username for authentication
  password: string;  // Optional password for authentication
}
```

#### Server Configuration
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts:7-10
const server = {
  port: number | string;  // Server port (default: 3000)
  nodeEnv: string;        // Environment (default: 'development')
}
```

#### MCP Configuration
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts:17-19
const mcp = {
  apiKey: string;  // API key for authentication (empty string if not set)
}
```

#### Connection Parser Types
```typescript
// File: /home/donbr/graphiti-org/FalkorDB-MCPServer/src/utils/connection-parser.ts:5-10
interface FalkorDBConnectionOptions {
  host: string;
  port: number;
  username?: string;
  password?: string;
}
```

**Connection String Format:**
```
falkordb://[username:password@]host:port
```

**Examples:**
- `falkordb://localhost:6379`
- `falkordb://user:pass@db.example.com:6379`
- `falkordb://mypassword@192.168.1.100:6379`

### HTTP Headers

#### Authentication Header
```
x-api-key: your_api_key_here
```

**Alternative (Query Parameter):**
```
GET /api/mcp/metadata?apiKey=your_api_key_here
```

### Status Codes

- **200 OK**: Successful request
- **400 Bad Request**: Missing or invalid parameters
- **401 Unauthorized**: Missing API key
- **403 Forbidden**: Invalid API key
- **500 Internal Server Error**: Server or database error

---

## Document Information

**Generated**: 2025-11-29
**Version**: 1.0
**Coverage**: Complete analysis of FalkorDB MCP Server data flows
**Files Analyzed**: 9 TypeScript source files, 2 test files
**Excluded**: ra_orchestrators/, ra_agents/, ra_tools/, ra_output/, .venv/

### Source Files Referenced

1. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/index.ts` - Main application entry point
2. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.ts` - MCP request handlers
3. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.ts` - Database service layer
4. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/routes/mcp.routes.ts` - Route definitions
5. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/middleware/auth.middleware.ts` - Authentication logic
6. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp.types.ts` - Type definitions
7. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/config/index.ts` - Configuration management
8. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/utils/connection-parser.ts` - Connection string parsing
9. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/models/mcp-client-config.ts` - Client configuration types

### Test Files Referenced

1. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/controllers/mcp.controller.test.ts`
2. `/home/donbr/graphiti-org/FalkorDB-MCPServer/src/services/falkordb.service.test.ts`
