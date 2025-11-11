# Library Check-in / Check-out System

## Goal
Design a system that supports:
- Checking out a book (loaning to a user)
- Returning a book
- Adding a book to the system

---

## Requirements

Functional
- Users can check out a copy of a book if available.
- Users can return a book they checked out.
- Admins/librarians can add new book records and physical copies.
- Support search and view of book availability and user loans.
- User authentication, based on unique membership number.

Non-functional
- High availability (99.95%+ for mission-critical).
- Low latency for reads (<200ms typical).
- Correctness: no double-loaning of the same physical copy.
- Auditability and observability.

---

## Key Assumptions & Constraints
- "Book" = a bibliographic record; "Copy" = a physical (or unique) item with barcode/ID.
- Each checkout is for a single copy (not fractional).
- System must support concurrent checkout attempts for the same copy.
- System must prevent double-loan even under concurrent requests.
- If offline kiosks are needed (out of scope), add sync mechanism later.
- Membership number is unique and primary auth credential (could be combined with password or multi-factor).
- Scale: moderate writes, heavy reads (catalog search); can grow with read replicas and caches.

---

## Microservices (functional responsibilities)
- API Gateway / Edge
  - Front door, TLS termination, rate limiting, and JWT validation (optional).
- Auth Service (Identity)
  - Responsible for membership authentication and issuing short-lived tokens.
  - Stores (or relies on managed provider for) membership number -> user id mapping and credentials.
- User Service
  - Application profile for members: name, membership number (unique), contact info, account status, loan limits.
- Inventory & Catalog Service
  - Bibliographic records: book (title, authors, ISBN, subjects).
  - Handles add/update book metadata.
  - Physical copies: copy_id, book_id, barcode, location (shelf), status (AVAILABLE, LOANED, LOST).
  - Add/remove copies, update physical location.
- Circulation Service
  - Core loan logic: checkout, return, renew, hold logic.
  - Enforces correctness (no double-loan) and loan business rules.
  - Uses transactions and locking with the authoritative DB.
- Search Service
  - Indexes catalog + availability + copy location for low-latency search (OpenSearch).

Cross-cutting components
- Idempotency Store (can be a small DB table or Redis) for safe retries.
- Observability (Logging, Tracing, Metrics) across all services.

## Physical & Logical Design

Logical architecture (data model highlights)
- User { user_id (PK), membership_number (unique), name, email, status, loan_limit }
- Book { book_id (PK), isbn, title, authors, subjects, metadata }
- Copy { copy_id (PK), book_id (FK), barcode, location_id, status, tenant_id? }
- Loan { loan_id (PK), copy_id (FK), user_id (FK), checkout_at, due_at, returned_at, status }

Principles:
- Authoritative state in RDBMS (Postgres): Users, Books, Copies, Loans, Audit.
- Denormalized search index (OpenSearch) for listing/searching availability and fast queries.
- Short-lived caches (Redis) for hot reads and idempotency keys.

Physical architecture
- API Gateway AWS API Gateway.
- Identity: AWS Cognito for membership authentication.
- Services deployed in containers: AWS EKS.
- Primary DB: Amazon Aurora (Postgres-compatible) — Multi-AZ, read replicas.
- Search: Amazon OpenSearch Service for text + availability queries.
- Cache: Amazon ElastiCache (Redis) for hot lookups / idempotency.
- Monitoring: Amazon CloudWatch, AWS X-Ray (distributed tracing), OpenTelemetry.
- Secrets: AWS Secrets Manager / Parameter Store.
- CI/CD: GitLab Pipelines / Repo -> CodePipeline -> ECR -> EKS/ECS.

## Concurrency & Correctness (prevent double-loan)
- Circulation Service uses DB transactions with SELECT ... FOR UPDATE on the Copy row:
  - Begin TX
  - SELECT * FROM copies WHERE copy_id = ? FOR UPDATE
  - Validate status == AVAILABLE
  - INSERT INTO loans(...)
  - UPDATE copies SET status = 'LOANED'
  - COMMIT
- Add a partial unique index to ensure at most one active loan per copy:
  - CREATE UNIQUE INDEX ux_loans_copy_active ON loans(copy_id) WHERE status = 'ACTIVE';
- Use idempotency keys for write APIs.

## API surface (concise)
- POST /auth/login  (returns JWT) — uses membership_number + password
- GET  /books?query=...
- POST /books  (ADMIN) — create bibliographic record
- POST /copies  (ADMIN) — add physical copy
- POST /loans/checkout  (Auth required)
  - Body: { copy_id } or { book_id, prefer_copy_id }
  - Headers: Authorization: Bearer <JWT>, Idempotency-Key
- POST /loans/return  (Auth required)
  - Body: { copy_id }
- GET  /users/{user_id}/loans  (Auth required; only own or admin)

Auth and membership number
- Membership number is unique identifier for login (mapped to user_id).
- Prefer token-based auth (JWT) with short-lived access tokens; services validate JWT locally against IdP's JWKS.
- For simple deployments, a small Auth Service can validate membership_number + password and issue JWTs; for production SaaS use Cognito/managed IdP.

## Idempotency and retries
- Accept Idempotency-Key header for checkout/return.
- Persist idempotency entries in Redis with durability TTL policy.
- For reliability, mark idempotency after commit.

## Auditability & Observability
- Write every domain-changing operation into AuditEvent table (or produce events + store immutable log).
- Log structured traces (OpenTelemetry) and send to CloudWatch.
- Capture request id, user_id, membership_number, service, timestamp.

## Non-functional & mission-critical architectural principles
- High availability
  - Multi-AZ DB, autoscaling service layer, health checks and automatic restarts.
  - Use load balancers and multiple AZs for pods/instances.
- Data durability
  - Automated DB backups, PITR, cross-region read replicas for DR.
- Failure isolation
  - Circuit breakers, bulkheads, rate limits per client.
- Consistency and correctness
  - Use strong DB transactions for critical paths; keep eventual consistency for search indices and caches.
- Security
  - TLS everywhere, least privilege IAM, secrets rotation, input validation, rate limiting.
- Observability & SLOs
  - Define SLOs e.g. 99.95% availability, P95 read latency <200ms.
  - Monitor error budgets, latency, lock wait times, DB deadlocks.
- Disaster Recovery
  - RTO/RPO target, runbooks, automated failover testing.
- Scalability
  - Horizontal scaling for read path (caches, read replicas), vertical scale for write DB or use partitioning/sharding if needed.

## Sequence diagrams (critical flows)
- Checkout flow (with JWT validation and idempotency)
- Return flow

## Key trade-offs
- Strong DB transactions (Postgres) vs serverless scale (DynamoDB): choose Postgres for correctness and simpler modeling.
- Outbox/eventing vs synchronous notifications: outbox is more reliable at cost of slightly more complexity.
- Managed IdP (Cognito) vs self-hosted: managed reduces ops burden.