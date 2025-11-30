# Architecture Diagrams

## Overview

FalkorDB MCP Server is a TypeScript/Node.js application that implements the Model Context Protocol (MCP) specification to provide a RESTful API interface to FalkorDB, a graph database. The system follows a classic 3-tier architecture pattern with clear separation between presentation (routes), business logic (controllers/services), and data access (FalkorDB service) layers.

The application uses Express.js for HTTP routing, provides authentication middleware, and exposes MCP-compliant endpoints for graph database operations including querying, metadata retrieval, and graph listing.

## 1. System Architecture (Layered View)

### Description

The system follows a classic layered architecture with clear separation of concerns:

- **Presentation Layer**: Express routes and middleware handling HTTP requests/responses
- **Business Logic Layer**: Controllers processing MCP requests and coordinating operations
- **Data Access Layer**: FalkorDB service managing database connections and query execution
- **External Systems**: FalkorDB database and MCP clients

```mermaid
graph TB
    subgraph "Presentation Layer"
        A[Express App<br/>src/index.ts]
        B[MCP Routes<br/>src/routes/mcp.routes.ts]
        C[Auth Middleware<br/>src/middleware/auth.middleware.ts]
    end

    subgraph "Business Logic Layer"
        D[MCP Controller<br/>src/controllers/mcp.controller.ts]
        E[Type Definitions<br/>src/models/mcp.types.ts]
        F[Client Config<br/>src/models/mcp-client-config.ts]
    end

    subgraph "Data Access Layer"
        G[FalkorDB Service<br/>src/services/falkordb.service.ts]
        H[Connection Parser<br/>src/utils/connection-parser.ts]
    end

    subgraph "Configuration Layer"
        I[Config Module<br/>src/config/index.ts]
        J[.env File]
    end

    subgraph "External Systems"
        K[(FalkorDB<br/>Graph Database)]
        L[MCP Clients]
    end

    L -->|HTTP Request| A
    A --> C
    C -->|Authenticate| B
    B --> D
    D --> E
    D --> G
    G --> H
    G --> K
    I --> J
    A --> I
    G --> I
    C --> I
```

### Key Points

- **Clean Separation**: Each layer has a distinct responsibility with minimal coupling
- **Configuration Management**: Centralized configuration using environment variables via dotenv
- **Authentication**: API key-based authentication enforced at the presentation layer
- **Singleton Pattern**: FalkorDB service uses singleton pattern for connection pooling
- **Type Safety**: Strong TypeScript typing throughout with dedicated type definitions

## 2. Component Relationships

### Description

This diagram shows how major components interact with each other, highlighting the request flow and dependencies between modules.

```mermaid
graph LR
    A[index.ts<br/>Application Entry] --> B[config/index.ts<br/>Configuration]
    A --> C[routes/mcp.routes.ts<br/>Route Definitions]
    A --> D[middleware/auth.middleware.ts<br/>Authentication]
    A --> E[services/falkordb.service.ts<br/>DB Service]

    C --> F[controllers/mcp.controller.ts<br/>Request Handlers]
    F --> E
    F --> G[models/mcp.types.ts<br/>Type Definitions]

    D --> B
    E --> B
    E --> H[utils/connection-parser.ts<br/>Connection Utils]

    I[External: FalkorDB SDK] -.->|npm dependency| E
    J[External: Express] -.->|npm dependency| A
    J -.->|npm dependency| C
    J -.->|npm dependency| D
    J -.->|npm dependency| F

    style A fill:#e1f5ff
    style E fill:#fff4e1
    style F fill:#e8f5e9
    style B fill:#f3e5f5
```

### Key Points

- **Central Entry Point**: `index.ts` orchestrates application initialization and component wiring
- **Dependency Injection**: Components receive dependencies (like config, services) through imports
- **Route-Controller Pattern**: Routes delegate business logic to controllers
- **Service Layer**: Controllers depend on services for data operations, not direct database access
- **External Dependencies**: Minimal external dependencies (Express, FalkorDB SDK, dotenv)

## 3. Class Hierarchies

### Description

This diagram shows the object-oriented structure of the main classes in the system, their properties, methods, and relationships.

```mermaid
classDiagram
    class MCPController {
        +processContextRequest(req, res) Promise~Response~
        +processMetadataRequest(req, res) Promise~Response~
        +listGraphs(req, res) Promise~Response~
    }

    class FalkorDBService {
        -client: FalkorDB | null
        +constructor()
        -init() Promise~void~
        +executeQuery(graphName, query, params) Promise~any~
        +listGraphs() Promise~string[]~
        +close() Promise~void~
    }

    class MCPContextRequest {
        +graphName: string
        +query: string
        +params?: Record~string, any~
        +context?: Record~string, any~
        +options?: MCPOptions
    }

    class MCPOptions {
        +timeout?: number
        +maxResults?: number
    }

    class MCPResponse {
        +data: any
        +metadata: MCPMetadata
    }

    class MCPMetadata {
        +timestamp: string
        +queryTime: number
        +provider?: string
        +source?: string
    }

    class MCPProviderMetadata {
        +provider: string
        +version: string
        +capabilities: string[]
        +graphTypes: string[]
        +queryLanguages: string[]
    }

    class FalkorDBConnectionOptions {
        +host: string
        +port: number
        +username?: string
        +password?: string
    }

    class MCPClientConfig {
        +defaultServer?: string
        +servers: Record~string, ServerConfig~
    }

    class MCPServerConfig {
        +mcpServers: Record~string, CommandConfig~
    }

    MCPController ..> FalkorDBService : uses
    MCPController ..> MCPContextRequest : processes
    MCPController ..> MCPResponse : returns
    MCPController ..> MCPProviderMetadata : returns

    MCPResponse *-- MCPMetadata : contains
    MCPContextRequest *-- MCPOptions : contains

    FalkorDBService ..> FalkorDBConnectionOptions : uses

    note for FalkorDBService "Singleton instance\nexported as falkorDBService"
    note for MCPController "Singleton instance\nexported as mcpController"
```

### Key Points

- **Singleton Pattern**: Both `MCPController` and `FalkorDBService` are exported as singleton instances
- **Interface-Based Design**: Heavy use of TypeScript interfaces for type safety and contract definitions
- **Composition**: Complex types like `MCPResponse` compose simpler types like `MCPMetadata`
- **Async Operations**: All service and controller methods are asynchronous returning Promises
- **Type Safety**: Strong typing ensures compile-time validation of data structures

## 4. Module Dependencies

### Description

This diagram shows the import relationships between modules, revealing the dependency structure and potential circular dependencies.

```mermaid
graph TD
    A[index.ts] --> B[config/index.ts]
    A --> C[routes/mcp.routes.ts]
    A --> D[middleware/auth.middleware.ts]
    A --> E[services/falkordb.service.ts]

    C --> F[controllers/mcp.controller.ts]

    F --> E
    F --> G[models/mcp.types.ts]

    D --> B

    E --> B
    E --> H[utils/connection-parser.ts]

    B --> I[External: dotenv]
    A --> J[External: express]
    C --> J
    D --> J
    F --> J
    E --> K[External: falkordb]

    L[models/mcp-client-config.ts]

    style A fill:#ff9999
    style B fill:#99ccff
    style E fill:#99ff99
    style F fill:#ffcc99
    style G fill:#cc99ff
    style L fill:#ffff99

    classDef external fill:#e0e0e0,stroke:#333,stroke-width:2px
    class I,J,K external
```

### Key Points

- **No Circular Dependencies**: Clean dependency graph with unidirectional flow
- **Configuration as Foundation**: `config/index.ts` is imported by multiple modules but imports nothing (except dotenv)
- **Layered Dependencies**: Higher layers depend on lower layers, never the reverse
- **External Dependencies**: Only three main external dependencies (dotenv, express, falkordb)
- **Loose Coupling**: `mcp-client-config.ts` is standalone with no dependencies, providing configuration templates

## 5. Sequence Diagram - Typical Request Flow

### Description

This diagram shows the complete lifecycle of a typical graph query request through the system, from client request to database response.

```mermaid
sequenceDiagram
    participant Client as MCP Client
    participant Express as Express App
    participant Auth as Auth Middleware
    participant Routes as MCP Routes
    participant Controller as MCP Controller
    participant Service as FalkorDB Service
    participant DB as FalkorDB Database

    Client->>Express: POST /api/mcp/context<br/>{graphName, query, params}
    Express->>Auth: authenticateMCP(req, res, next)
    Auth->>Auth: Check x-api-key header

    alt API Key Invalid
        Auth-->>Client: 401/403 Unauthorized
    else API Key Valid
        Auth->>Routes: next()
        Routes->>Controller: processContextRequest(req, res)

        Controller->>Controller: Validate request body

        alt Validation Failed
            Controller-->>Client: 400 Bad Request
        else Validation Passed
            Controller->>Service: executeQuery(graphName, query, params)
            Service->>Service: Check client initialized

            alt Client Not Initialized
                Service-->>Controller: Error: Client not initialized
                Controller-->>Client: 500 Internal Error
            else Client Ready
                Service->>DB: graph.query(query, params)
                DB-->>Service: Query Results
                Service-->>Controller: Return results

                Controller->>Controller: Format MCPResponse<br/>{data, metadata}
                Controller-->>Client: 200 OK + Results
            end
        end
    end
```

### Key Points

- **Authentication First**: All requests go through API key validation before processing
- **Early Validation**: Request validation happens at controller level before database operations
- **Error Handling**: Multiple error paths with appropriate HTTP status codes (400, 401, 403, 500)
- **Metadata Enrichment**: Controller adds timestamp, query time, and provider information to responses
- **Connection Management**: Service checks client initialization before executing queries
- **Async Flow**: Entire flow is asynchronous with proper error propagation

## 6. Deployment Architecture

### Description

This diagram shows how the system is containerized and deployed using Docker, including the multi-stage build process and runtime configuration.

```mermaid
graph TB
    subgraph "Development Environment"
        A[Source Code<br/>TypeScript]
        B[npm Dependencies<br/>package.json]
    end

    subgraph "Docker Build Stage 1: Builder"
        C[Node 18 Alpine Base]
        D[Install Dependencies<br/>npm ci]
        E[Build TypeScript<br/>npm run build]
        F[Prune Dev Dependencies<br/>npm prune --production]
    end

    subgraph "Docker Build Stage 2: Production"
        G[Node 18 Alpine Base]
        H[Copy Artifacts<br/>dist/, node_modules/]
        I[Set ENV=production]
        J[Expose Port 3000]
    end

    subgraph "Runtime Environment"
        K[Environment Variables<br/>.env or docker --env-file]
        L[FalkorDB Instance<br/>Host: configurable<br/>Port: 6379]
    end

    subgraph "Container Runtime"
        M[FalkorDB MCP Server<br/>Port: 3000]
        N[MCP API Endpoints<br/>/api/mcp/*]
    end

    A --> C
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G

    G --> H
    H --> I
    I --> J
    J --> M

    K -.->|Configuration| M
    M <-->|Graph Queries| L

    O[MCP Clients] -->|HTTP Requests| N
    N --> M

    style C fill:#e3f2fd
    style G fill:#e8f5e9
    style M fill:#fff3e0
    style L fill:#f3e5f5
```

### Key Points

- **Multi-Stage Build**: Optimizes image size by separating build and runtime dependencies
- **Alpine Base**: Uses lightweight Alpine Linux reducing container size
- **Production Optimization**: Development dependencies removed in production image
- **Port Exposure**: Container exposes port 3000 for HTTP traffic
- **Environment-Based Config**: Supports configuration through environment variables or .env files
- **External Database**: Connects to external FalkorDB instance (not containerized together)
- **Stateless Design**: Application container is stateless, all state in FalkorDB

## 7. Data Flow Architecture

### Description

This diagram illustrates how data flows through the system from external requests to database operations and back.

```mermaid
graph LR
    subgraph "Input"
        A[HTTP Request<br/>JSON Payload]
        A1[Headers<br/>x-api-key]
    end

    subgraph "Validation & Auth"
        B[API Key Validation]
        C[Request Body Validation<br/>MCPContextRequest]
    end

    subgraph "Processing"
        D[Extract Parameters<br/>graphName, query, params]
        E[Execute Cypher Query<br/>on FalkorDB]
        F[Measure Query Time]
    end

    subgraph "Response Formation"
        G[Wrap Results<br/>in MCPResponse]
        H[Add Metadata<br/>timestamp, queryTime, provider]
    end

    subgraph "Output"
        I[HTTP Response<br/>JSON with data + metadata]
    end

    A --> B
    A1 --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I

    style A fill:#ffebee
    style B fill:#fff3e0
    style C fill:#fff3e0
    style E fill:#e8f5e9
    style I fill:#e1f5fe
```

### Key Points

- **Linear Flow**: Clear unidirectional data flow from input to output
- **Multiple Validation Stages**: API key validation followed by request body validation
- **Metadata Augmentation**: Response enriched with operational metadata (timing, provider info)
- **Type Safety**: Data validated against TypeScript interfaces at each stage
- **Error Propagation**: Errors at any stage short-circuit to error response

## 8. Component Interaction Matrix

### Description

This table shows which components interact with each other and the nature of their interactions.

| Component | Interacts With | Interaction Type | Purpose |
|-----------|----------------|------------------|---------|
| index.ts | config | Import | Load application configuration |
| index.ts | routes/mcp.routes | Import | Mount API routes |
| index.ts | middleware/auth | Import | Apply authentication |
| index.ts | services/falkordb | Import | Graceful shutdown |
| routes/mcp.routes | controllers/mcp | Import | Delegate request handling |
| controllers/mcp | services/falkordb | Import | Execute database operations |
| controllers/mcp | models/mcp.types | Import | Type definitions |
| services/falkordb | config | Import | Database connection config |
| services/falkordb | utils/connection-parser | Import | Parse connection strings |
| middleware/auth | config | Import | Validate API keys |

## 9. API Endpoint Architecture

### Description

This diagram shows the available API endpoints and their relationships.

```mermaid
graph TB
    A[Base URL<br/>http://host:3000]

    subgraph "Public Endpoints"
        B[GET /<br/>Health & Info]
    end

    subgraph "MCP API Endpoints<br/>/api/mcp"
        C[POST /context<br/>Execute Graph Query]
        D[GET /metadata<br/>Get Provider Capabilities]
        E[GET /graphs<br/>List Available Graphs]
        F[GET /health<br/>MCP Health Check]
    end

    A --> B
    A --> G[Auth Middleware<br/>x-api-key required]
    G --> C
    G --> D
    G --> E
    G --> F

    C -.-> H[(FalkorDB<br/>Graph Query)]
    D -.-> I[In-Memory<br/>Metadata]
    E -.-> J[(FalkorDB<br/>List Operation)]
    F -.-> K[Service Health]

    style C fill:#ffccbc
    style D fill:#c5e1a5
    style E fill:#b3e5fc
    style F fill:#f8bbd0
    style G fill:#ffe082
```

### Key Points

- **Authentication Boundary**: `/api/mcp/*` routes protected by API key authentication
- **RESTful Design**: Uses appropriate HTTP methods (GET for retrieval, POST for operations)
- **Health Monitoring**: Multiple health check endpoints at different levels
- **MCP Compliance**: Endpoints follow MCP specification patterns
- **Separation of Concerns**: Public vs authenticated endpoints clearly separated

## Summary

The FalkorDB MCP Server architecture demonstrates several strong architectural patterns:

1. **Layered Architecture**: Clear separation between presentation, business logic, and data access layers
2. **Singleton Services**: Database service and controller use singleton pattern for resource efficiency
3. **Middleware Pattern**: Express middleware for cross-cutting concerns like authentication
4. **Type Safety**: Comprehensive TypeScript interfaces ensuring compile-time validation
5. **Configuration Management**: Centralized, environment-based configuration
6. **Error Handling**: Multiple validation points with appropriate error responses
7. **Containerization**: Production-ready Docker configuration with multi-stage builds
8. **MCP Compliance**: Implements Model Context Protocol specification for graph database access

The architecture is well-suited for:
- **Scalability**: Stateless design allows horizontal scaling
- **Maintainability**: Clear module boundaries and minimal coupling
- **Testability**: Dependency injection and layered design facilitate unit testing
- **Security**: API key authentication and input validation at multiple levels
- **Reliability**: Connection retry logic and graceful shutdown handling
