# BrightSign Full-Stack Web Application Development Guidelines

When analyzing work/designs/implementation, NEVER give time estimates unless specifically asked for them.

## Essential Guidelines

We use React & TypeScript
* Use React functional components exclusively
* Utilize hooks (useState, useMemo) for state management
* TypeScript best practices with strict typing
  * Prefer types over interfaces (except when extending/merging)
* Prefer named exports over default exports (except Next.js special files)

## Styling & UI

* Use Tailwind CSS classes for styling
* Use shadcn/ui components (https://ui.shadcn.com/) for consistent UI elements

## Code Quality

* Write clean, maintainable code following DRY principle
* Emphasize security, robustness, readability, separation of concerns
* Use relative imports within the same project
* JSDoc comments for public APIs, avoid redundant comments
* Prefer direct named imports (e.g., import { FC } from 'react' not React.FC)

## Testing

* Use @testing-library/react for unit tests
* Test file naming: *.test.tsx or *.spec.tsx
* Playwright for E2E tests
* Ensure components are accessible and responsive

## REST API Design

### API Structure & Design Principles

* Follow RESTful conventions and resource-based URLs
  * Use nouns for resources: `/users`, `/projects`, not `/getUsers`
  * Use HTTP methods to express actions: GET, POST, PUT, PATCH, DELETE
  * Maintain consistent naming: use kebab-case or snake_case consistently
* Version your APIs explicitly
  * Prefer URL versioning: `/api/v1/users` or header-based versioning
  * Maintain backwards compatibility when possible
* Design for idempotency
  * GET, PUT, DELETE should be idempotent
  * POST for creation, PATCH for partial updates

### HTTP Methods & Status Codes

* Use appropriate HTTP methods:
  * GET - Retrieve resources (no side effects)
  * POST - Create new resources
  * PUT - Replace entire resource
  * PATCH - Partial resource update
  * DELETE - Remove resource
* Return proper HTTP status codes:
  * 200 OK - Successful GET, PUT, PATCH, DELETE
  * 201 Created - Successful POST with resource creation
  * 204 No Content - Successful DELETE or update with no response body
  * 400 Bad Request - Invalid client request
  * 401 Unauthorized - Authentication required
  * 403 Forbidden - Authenticated but insufficient permissions
  * 404 Not Found - Resource doesn't exist
  * 409 Conflict - Resource conflict (e.g., duplicate)
  * 422 Unprocessable Entity - Validation errors
  * 500 Internal Server Error - Server-side errors

### Request & Response Patterns

* Use consistent JSON structure for responses:
  ```typescript
  {
    "data": { /* resource data */ },
    "meta": { /* pagination, timestamps */ },
    "error": { /* error details if applicable */ }
  }
  ```
* Implement pagination for list endpoints:
  * Use `limit` and `offset` or cursor-based pagination
  * Include pagination metadata in responses
* Support filtering, sorting, and field selection:
  * Query params: `?status=active&sort=-createdAt&fields=id,name`
* Use ISO 8601 format for dates: `2025-11-14T10:30:00Z`
* Always validate request payloads with Zod or similar

### Error Handling

* Return consistent error response format:
  ```typescript
  {
    "error": {
      "code": "VALIDATION_ERROR",
      "message": "User-friendly error message",
      "details": [{ "field": "email", "issue": "Invalid format" }]
    }
  }
  ```
* Include request IDs for debugging: `X-Request-ID` header
* Log errors server-side with context (never expose stack traces in production)
* Use problem details format (RFC 7807) for complex errors

### Authentication & Authorization

* Use bearer tokens (JWT) in Authorization header: `Bearer <token>`
* Implement refresh token rotation for security
* Validate and sanitize all inputs to prevent injection attacks
* Use HTTPS exclusively in production
* Implement rate limiting per endpoint/user
* Apply principle of least privilege for API keys/tokens

### API Documentation

* Use OpenAPI/Swagger specification for API documentation
* Include example requests and responses
* Document rate limits, authentication requirements, error codes
* Keep documentation in sync with implementation
* Consider tools like Swagger UI or Postman collections

### Performance & Caching

* Implement HTTP caching headers: `Cache-Control`, `ETag`
* Use compression for responses: gzip/brotli
* Implement request throttling and rate limiting
* Consider GraphQL or API aggregation for reducing round trips
* Monitor API performance and set SLAs

## PostgreSQL Database Practices

**We strongly prefer PostgreSQL for all database needs.** Use PostgreSQL unless there is an exceptional, well-documented reason to use another database system.

### Database Design

* Follow normalization principles (typically 3NF) but denormalize when performance requires
* Use appropriate PostgreSQL data types:
  * `UUID` or `BIGSERIAL` for primary keys (avoid sequential integers for exposed IDs)
  * `TIMESTAMP WITH TIME ZONE` (or `TIMESTAMPTZ`) for all timestamps
  * `JSONB` for flexible schema data (never use `JSON`, always `JSONB` for performance)
  * `TEXT` instead of `VARCHAR` (PostgreSQL treats them identically, TEXT is more flexible)
  * `BIGINT` for large numeric values
  * `NUMERIC(precision, scale)` for exact decimal values (e.g., currency)
  * `BOOLEAN` for true/false values
  * Arrays for list data: `TEXT[]`, `INTEGER[]`, etc.
* Implement proper indexing strategy:
  * Index foreign keys, frequently queried columns, and WHERE clause fields
  * Use `CREATE INDEX CONCURRENTLY` to avoid locking tables in production
  * Use composite indexes for multi-column queries
  * Use partial indexes for filtering: `CREATE INDEX ON orders (status) WHERE status = 'pending'`
  * Use GIN indexes for JSONB, arrays, and full-text search
  * Use BRIN indexes for large tables with naturally ordered data (e.g., timestamps)
  * Monitor and remove unused indexes with `pg_stat_user_indexes`
* Design schemas with future scalability in mind
* Use constraints to enforce data integrity:
  * PRIMARY KEY, FOREIGN KEY, UNIQUE, NOT NULL, CHECK constraints
  * Use EXCLUDE constraints for complex uniqueness requirements

### Query Optimization

* Write efficient PostgreSQL queries:
  * Select only needed columns: `SELECT id, name` not `SELECT *`
  * Use `LIMIT` and `OFFSET` for basic pagination (or cursor-based for better performance)
  * Avoid N+1 queries - use JOINs or eager loading
  * Use `EXPLAIN ANALYZE` to understand query plans and identify bottlenecks
  * Use `EXPLAIN (ANALYZE, BUFFERS)` for detailed performance analysis
* Leverage PostgreSQL-specific features:
  * CTEs (Common Table Expressions) with `WITH` clauses for complex queries
  * `WITH RECURSIVE` for hierarchical data
  * Window functions (`ROW_NUMBER()`, `RANK()`, `LAG()`, `LEAD()`) for analytics
  * `LATERAL` joins for correlated subqueries
  * Full-text search with `tsvector`, `tsquery`, and GIN indexes
  * JSONB operators (`->`, `->>`, `@>`, `?`) for JSON querying
  * Array functions and operators for list operations
  * `RETURNING` clause to get inserted/updated data without additional queries
* Cache frequently accessed data at application layer
* Use connection pooling with PgBouncer or similar tools
* Use materialized views for complex, frequently-run queries:
  ```sql
  CREATE MATERIALIZED VIEW user_stats AS
  SELECT user_id, COUNT(*) as total_orders FROM orders GROUP BY user_id;
  REFRESH MATERIALIZED VIEW CONCURRENTLY user_stats;
  ```

### ORM Usage & Query Builders

Do not use any ORM without specific approval first. Always ask before using an ORM.

General ORM guidance to use when selecting an ORM for recommendation (must have excellent PostgreSQL support):

* Use type-safe ORMs/query builders with strong PostgreSQL support:
  * Prisma (preferred for TypeScript projects - excellent PostgreSQL support)
  * Drizzle ORM (lightweight, type-safe, PostgreSQL-first)
  * Knex.js for query building
  * node-postgres (pg) for direct database access
* Ensure ORM supports PostgreSQL-specific features:
  * JSONB operations
  * Array types
  * Full-text search
  * Specific index types (GIN, BRIN, partial)
  * Transaction isolation levels
* Generate types from database schema automatically
* Use migrations for all schema changes:
  ```typescript
  // Example Prisma migration workflow
  // 1. Edit schema.prisma
  // 2. npx prisma migrate dev --name add_user_email
  // 3. Commit migration files
  ```
* Always use parameterized queries (PostgreSQL uses `$1`, `$2` notation):
  ```typescript
  // Direct pg usage
  await pool.query('SELECT * FROM users WHERE email = $1', [email])
  ```
* Use transactions for operations that must be atomic:
  ```typescript
  // Prisma example
  await prisma.$transaction(async (tx) => {
    await tx.user.update(...)
    await tx.account.create(...)
  })

  // Direct pg example
  const client = await pool.connect()
  try {
    await client.query('BEGIN')
    await client.query('UPDATE users SET ...')
    await client.query('INSERT INTO audit_log ...')
    await client.query('COMMIT')
  } catch (e) {
    await client.query('ROLLBACK')
    throw e
  } finally {
    client.release()
  }
  ```

### Migrations & Schema Management

* Always use migrations - never modify production schema manually
* Use migration tools compatible with PostgreSQL:
  * Prisma Migrate
  * Knex migrations
  * node-pg-migrate
  * Flyway or Liquibase for larger projects
* Make migrations reversible when possible (include `down` migrations)
* Test migrations on staging before production
* Keep migrations small and focused
* Use `CREATE INDEX CONCURRENTLY` in migrations to avoid locking
* Version control all migration files
* Document breaking changes and data migrations
* Consider using transactional DDL (PostgreSQL supports this for most operations):
  ```sql
  BEGIN;
  ALTER TABLE users ADD COLUMN email_verified BOOLEAN DEFAULT FALSE;
  UPDATE users SET email_verified = TRUE WHERE email IS NOT NULL;
  COMMIT;
  ```

### Security Best Practices

* **Never** construct SQL queries with string concatenation
* Use parameterized queries/prepared statements exclusively:
  ```typescript
  // ✅ Good
  await db.query('SELECT * FROM users WHERE id = $1', [userId])

  // ❌ Bad - SQL injection risk
  await db.query(`SELECT * FROM users WHERE id = ${userId}`)
  ```
* Apply principle of least privilege for database users:
  * Application should not use superuser credentials
  * Read-only users for reporting/analytics
* Encrypt sensitive data at rest and in transit
* Implement audit logging for sensitive operations
* Regular security updates and patches
* Use environment variables for credentials, never commit them

### Connection Management

* Use connection pooling:
  * Set appropriate pool size based on concurrent load
  * Configure connection timeouts and max lifetime
* Handle connection failures gracefully with retries
* Close connections properly to prevent leaks
* Monitor connection pool metrics (active, idle, waiting)
* Consider read replicas for read-heavy workloads

### Performance Monitoring

* Track slow queries and optimize them:
  * Enable `pg_stat_statements` extension for query tracking
  * Configure `log_min_duration_statement` to log slow queries
  * Use `auto_explain` module in development
* Monitor PostgreSQL-specific metrics:
  * Query execution time via `pg_stat_statements`
  * Connection pool utilization
  * Cache hit ratios (`pg_stat_database`)
  * Index usage statistics (`pg_stat_user_indexes`)
  * Table bloat and dead tuple counts
  * Replication lag (if using replicas)
  * Lock contention (`pg_locks`)
* Set up alerting for anomalies:
  * High connection counts
  * Long-running queries
  * Replication lag
  * Low cache hit ratios
  * Table bloat
* Regular VACUUM and ANALYZE operations:
  * Configure `autovacuum` appropriately (usually enabled by default)
  * Monitor autovacuum activity
  * Manual `VACUUM ANALYZE` for heavily updated tables
  * Consider `VACUUM FULL` for reclaiming space (locks table)
* Archive old data using partitioning to keep tables performant:
  ```sql
  CREATE TABLE orders (
    id BIGSERIAL,
    created_at TIMESTAMPTZ NOT NULL,
    ...
  ) PARTITION BY RANGE (created_at);

  CREATE TABLE orders_2025_01 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');
  ```

### Backup & Recovery

* Implement automated regular backups using PostgreSQL tools:
  * `pg_dump` for logical backups (single database)
  * `pg_dumpall` for cluster-wide backups
  * `pg_basebackup` for physical backups
  * WAL archiving for continuous backup
  * Consider tools like pgBackRest or Barman for advanced backup management
* Test backup restoration procedures regularly
* Use point-in-time recovery (PITR) with WAL archiving:
  ```bash
  # Enable WAL archiving in postgresql.conf
  wal_level = replica
  archive_mode = on
  archive_command = 'cp %p /backup/archive/%f'
  ```
* Keep backups in separate geographic location (different availability zone/region)
* Document recovery procedures and RTO/RPO targets
* Consider streaming replication for high availability
* Use logical replication for selective data replication or upgrades

### Development Workflow

* Use separate PostgreSQL databases for development, testing, staging, production
* Seed databases with realistic test data
* Use Docker for local PostgreSQL consistency:
  ```yaml
  # docker-compose.yml example
  services:
    postgres:
      image: postgres:16-alpine  # Use specific version
      environment:
        POSTGRES_DB: myapp_dev
        POSTGRES_USER: myapp
        POSTGRES_PASSWORD: dev_password
      ports:
        - "5432:5432"
      volumes:
        - postgres_data:/var/lib/postgresql/data
  ```
* Keep PostgreSQL versions consistent across environments
* Use configuration files for environment-specific settings (.env, etc.)
* Install useful PostgreSQL extensions in development:
  * `pg_stat_statements` - query performance tracking
  * `uuid-ossp` or `pgcrypto` - UUID generation
  * `pg_trgm` - trigram matching for fuzzy search
  * `citext` - case-insensitive text type
  * `hstore` - key-value store
* Use psql for interactive database work:
  ```bash
  psql postgresql://user:password@localhost:5432/dbname
  ```
* Consider GUI tools for development:
  * pgAdmin
  * DBeaver
  * TablePlus
  * DataGrip

## Documentation

### Writing Style

* Write like a human, not an AI:
  * Avoid corporate buzzwords and marketing speak ("leverage", "utilize", "streamline", "robust")
  * Skip unnecessary superlatives ("amazing", "incredibly", "super easy")
  * Don't start with throat-clearing phrases like "It's worth noting that..." or "It should be mentioned..."
  * Be direct: Write "Use X for Y" not "You might want to consider using X for Y"
* Keep it simple and concise:
  * One idea per sentence
  * Short paragraphs (2-4 sentences max)
  * Cut filler words: "basically", "actually", "essentially", "in order to"
  * Use active voice: "Run the command" not "The command should be run"
* Be specific and concrete:
  * Show, don't tell: Include code examples
  * Give actual commands/code, not abstract descriptions
  * Use real examples: `getUserById(123)` not `getUserById(id)`

### Structure & Organization

* Start with what, then why, then how:
  ```markdown
  ## User Authentication

  Handles user login and session management. We use JWT tokens
  because they're stateless and work across services.

  ### Setup
  1. Install dependencies: `npm install jsonwebtoken`
  2. Set JWT_SECRET in .env
  ...
  ```
* Use clear headings:
  * H2 for major sections
  * H3 for subsections
  * H4 sparingly, only when truly needed
* Front-load important information:
  * Put most critical info first
  * Don't bury requirements or warnings
  * Use callout blocks for important notes

### Code Examples

* Always include working examples:
  * Complete, runnable code when possible
  * Real variable names, not foo/bar
  * Show imports and setup
* Add brief comments only where code isn't self-explanatory:
  ```typescript
  // ✅ Good - explains why, not what
  // Cache for 5 minutes to reduce API calls
  const ttl = 300;

  // ❌ Bad - just repeats the code
  // Set ttl to 300
  const ttl = 300;
  ```
* Show both correct and incorrect patterns when helpful:
  ```typescript
  // ✅ Good
  const result = await fetchUser(id);

  // ❌ Bad - doesn't handle errors
  const result = fetchUser(id);
  ```

### Diagrams & Visuals

* Use Mermaid for all diagrams unless otherwise specified:
  * Flow charts for processes
  * Sequence diagrams for API interactions
  * ER diagrams for database schemas
  * Class diagrams for architecture
* Keep diagrams simple:
  * Focus on the essential elements
  * Limit to 5-10 nodes when possible
  * Use consistent naming with code

Example:
```mermaid
sequenceDiagram
    Client->>API: POST /login
    API->>Database: Verify credentials
    Database-->>API: User data
    API-->>Client: JWT token
```

### README Structure

Follow this structure for project READMEs:

1. **Project name and one-line description**
2. **Prerequisites** (Node version, PostgreSQL, etc.)
3. **Quick start** (installation and first run)
4. **Configuration** (environment variables)
5. **Common tasks** (development, testing, deployment)
6. **Project structure** (brief file organization)
7. **API documentation** (link to detailed docs)

### API Documentation

* Document every endpoint with:
  * HTTP method and path
  * Brief description (one sentence)
  * Request parameters/body with types
  * Response format with status codes
  * Example request and response
  * Error responses
* Use OpenAPI/Swagger when possible for automatic documentation
* Keep inline with code or generate from code when feasible

### Comments in Code

* Only comment when code can't be self-documenting:
  * Complex algorithms
  * Non-obvious business logic
  * Workarounds for bugs/quirks
  * Performance optimizations
* Don't comment obvious things:
  ```typescript
  // ❌ Bad
  // Increment counter
  counter++;

  // ✅ Good
  // Skip archived projects to improve query performance
  const projects = await db.query('SELECT * FROM projects WHERE archived = false');
  ```
* Use JSDoc for public APIs and exported functions:
  ```typescript
  /**
   * Fetches user by ID from database.
   * Returns null if user not found.
   */
  export async function getUserById(id: string): Promise<User | null> {
    // ...
  }
  ```

### Maintenance

* Update docs when code changes - stale docs are worse than no docs
* Delete outdated documentation rather than letting it accumulate
* Date-stamp major decisions and architectural docs
* Link to related code/files using relative paths

## Project Layout

### Monorepo Structure

* Organize as a monorepo containing both frontend and backend in the same repository
* Treat the project as supporting distributed systems from the start:
  * Services should be independently deployable
  * Each service can scale separately
  * Clear boundaries between components
* Use workspace/package management appropriate to the language ecosystem:
  * Node.js: npm workspaces, pnpm workspaces, or Yarn workspaces
  * Python: Poetry with workspace plugins or monorepo tools
  * Go: Go modules with multi-module workspaces
  * Other languages: Use equivalent tooling

### Top-Level Organization Principles

* Separate concerns at the top level:
  * Frontend applications
  * Backend services/APIs
  * Shared/common code
  * Infrastructure/deployment configuration
  * Documentation
  * Tools and scripts
* Use clear, descriptive names that indicate purpose:
  * Avoid generic names like "src" or "lib" at the root
  * Be explicit: "web-app", "api-server", "shared-types" not "frontend", "backend", "common"
* Group by function or service, not by technical type:
  * ✅ Good: `/user-service/controllers`, `/user-service/models`
  * ❌ Bad: `/controllers/user`, `/models/user`
* Consider future growth:
  * Leave room for multiple frontend apps (admin, customer, mobile)
  * Plan for multiple backend services (auth, orders, notifications)
  * Anticipate shared packages (types, utilities, design system)

### Frontend Structure Considerations

* Each frontend app should be self-contained:
  * Own dependencies, build config, and deployment
  * Can be developed and deployed independently
  * May share common packages from workspace
* Consider the framework's conventions:
  * Next.js: app/pages router, public assets, config files
  * React SPA: src with components, pages, hooks, etc.
  * Other frameworks: follow their established patterns
* Separate UI components from business logic:
  * Shared component library if multiple frontends exist
  * API clients and data fetching logic
  * State management clearly separated

### Backend Structure Considerations

* Each backend service should have clear boundaries:
  * Own database/data layer (or clearly defined shared DB)
  * Own API endpoints and business logic
  * Own dependencies and configuration
  * Can be deployed independently
* Structure services for the chosen architecture:
  * REST APIs: routes, controllers, services, models
  * GraphQL: schema, resolvers, data sources
  * Microservices: domain-driven boundaries
* Common patterns to include:
  * Configuration management (environment-specific)
  * Middleware and authentication
  * Database migrations and seeds
  * Background jobs/workers if needed
  * Testing directories alongside code

### Shared Code Organization

* Create packages for code used across services:
  * TypeScript: shared types, interfaces, validation schemas
  * API contracts: request/response types, OpenAPI specs
  * Utilities: date formatting, validation, common algorithms
  * Database models (if shared database approach)
  * Design system/UI components (for multiple frontends)
* Make shared packages explicitly versioned or reference them via workspace links
* Keep shared code minimal - prefer duplication over wrong abstractions
* Each shared package should be independently testable

### Configuration and Infrastructure

* Keep infrastructure code in the repository:
  * Docker and docker-compose files
  * Kubernetes manifests or Helm charts
  * Terraform or infrastructure-as-code
  * CI/CD pipeline definitions
* Environment-specific configuration:
  * Template .env files (.env.example) with all variables documented
  * Separate configs for dev, staging, production
  * Never commit secrets (use .gitignore)
* Root-level configuration files:
  * Workspace/monorepo config
  * Linting and formatting (ESLint, Prettier, etc.)
  * TypeScript config (with extends for sub-projects)
  * Testing configuration
  * Editor config (.editorconfig, .vscode)

### Build and Deployment Considerations

* Each service should have its own build process:
  * Independent Docker images
  * Separate build scripts
  * Own deployment configuration
* Optimize for build caching:
  * Structure Dockerfiles to cache dependencies
  * Separate build and runtime dependencies
  * Use multi-stage builds
* Support local development:
  * docker-compose for running full stack locally
  * Scripts for common tasks (setup, test, seed data)
  * Clear documentation for getting started

### Database and Migrations

* Structure database code appropriately:
  * Migrations in version control, organized by service if using separate DBs
  * Seed data for development/testing
  * Schema documentation or auto-generated from migrations
* Consider data ownership:
  * Shared database: migrations in shared location
  * Per-service databases: migrations with each service
  * Clear documentation of data relationships across services

### Testing Structure

* Organize tests based on scope:
  * Unit tests: alongside code or in parallel structure
  * Integration tests: may be in service or separate package
  * E2E tests: typically separate package that can test across services
  * Performance/load tests: separate if they exist
* Share test utilities:
  * Test fixtures and factories
  * Mock data generators
  * Testing utilities and helpers

### Language-Specific Adaptations

When creating structure for a specific language/framework:

* **TypeScript/Node.js projects**: Use workspace features, consider tsconfig inheritance, share types across packages
* **Python projects**: Use src-layout or flat-layout per service, consider namespace packages for shared code
* **Go projects**: Follow Go conventions (cmd, internal, pkg), consider multi-module workspaces
* **Framework-specific**: Respect framework conventions (Next.js, NestJS, Django, FastAPI, etc.) within each service
* **Language tooling**: Use language-specific dependency management, build tools, and testing frameworks

### Documentation in Structure

* Each service/package should have its own README:
  * Purpose and responsibilities
  * Local development setup
  * API documentation or link to it
  * Dependencies and environment variables
* Root README provides:
  * Project overview
  * How to run the entire stack
  * Links to individual service docs
  * Architecture decisions

### Key Principles Summary

1. **Separation**: Clear boundaries between services and concerns
2. **Independence**: Each service can be developed, tested, and deployed separately
3. **Scalability**: Structure supports adding new services and frontends
4. **Clarity**: Names and organization make purpose obvious
5. **Convention**: Follow language and framework conventions within each service
6. **Pragmatism**: Start simple, refactor as complexity demands
