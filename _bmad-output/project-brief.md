# Project Constitution

## Project Structure Philosophy
- Feature-based modularization: group by domain, not technical role
- Strict separation of concerns: unidirectional dependencies (routes → controller → service → repository)
- Co-located tests, types, schemas next to implementation
- Convention over configuration: predictable file naming and organization
- Progressive disclosure: start simple, add complexity when needed
- Multi-service pattern: `backend/`, `frontend/`, `[specialized-service]/`, `docs/`, `testing/`, `release/`
- Each service is an independent git-tracked package (not monorepo workspace)
- Must have k6 for HTTP endpoint and load testing
- Separate performance testing for frontend, backend, and microservices
- Frontend MUST prefer TanStack modules (Query, Router, Table)
- Shared contracts via `packages/dto/` internal package (T1 resolved)
- Data layer: `src/repository/{schemas,models,cache}` + per-module `*.repository.ts` (T3 resolved)
- Promises: async/await default; `.then().catch()` only via Cat #15 exception registry (T2 deferred)

## Backend Architecture Standards

### Module File Structure (per feature)
- `{feature}.routes.ts` — Express Router, middleware chains
- `{feature}.controller.ts` — request parsing + response formatting
- `{feature}.service.ts` — business logic (orchestration)
- `{feature}.repository.ts` — Prisma data access layer
- `{feature}.dto.ts` — Zod schemas + inferred types
- `{feature}.service.test.ts`, `{feature}.routes.test.ts`, `{feature}.repository.test.ts`

### Dependency Flow
- Routes → Controller → Service → Repository → Prisma
- Controllers never call repositories directly
- Services never format HTTP responses
- Controllers perform Zod validation via DTO schemas (not services)
- Controllers never write inline JSON errors — throw typed exceptions or `next(err)`

### Controller Pattern
- Static methods (class) or standalone exported functions
- Try/catch wrapping: Zod parse input → service call → `AppResponse.Success(data)` or `AppResponse.Failure(err)`
- Zod validation via DTO schemas at controller entry
- Never format response inline — always delegate to `AppResponse`
- Keep under 30 lines

### Response Envelope (AppResponse)
- `AppResponse.Success<D>(data: D)` — wraps data in `{ success: true, data }`
- `AppResponse.Failure(error: AppException)` — wraps error in `{ success: false, error: { message, code, status, details } }`
- Lives in `utils/AppResponse.ts` — duplicated per service with different body formats
- Controllers never write raw JSON — always use `AppResponse`

### Service Pattern
- Static methods or standalone functions
- Zod validation at controller layer (not service)
- Throw domain exceptions for business logic failures
- Under 50 lines per method

### DTO / Zod Pattern
- One `dto.ts` per module
- Zod schemas define all API input validation
- Controllers call `.parse()` on DTO schemas to validate input
- Export inferred types: `type X = z.infer<typeof xSchema>`
- Use `z.literal()` for enum-like constraints

### Repository Pattern
- Static methods or standalone functions wrapping Prisma queries
- Safe-user projection: strip sensitive fields before returning
- Owner-check: `findFirst` with both `id` + `userId` filters
- Filename suffix: `.repository.ts`

### Exception Pattern
- Base class: `AppException` extends `Error` — code, status, details fields
- Domain exception files with static factory methods
- `throw AuthException.invalidCredentials()` — never `throw new AuthException(...)`
- One exception file per domain, not per error type
- Error middleware order: `AppException` → `ZodError` (400 with field details) → unknown (500)

### Middleware
- Auth: Bearer token extraction → JWT RS256 verification → `req.user` attachment
- Upload: multer memoryStorage, 50MB limit, PDF-only (mimetype + magic bytes)
- Error: global Express 4-arg handler, typed error → structured response via `AppResponse.Failure`
- Logging: log incoming requests with `X-Trace-Id` header as correlation ID; log outgoing responses with same traceId, status, duration

### Database Conventions
- All columns: snake_case with `@map()`
- All tables: plural snake_case with `@@map()`
- All IDs: `String @id @default(uuid())`
- Timestamps: `createdAt` (now), `updatedAt` (@updatedAt)
- Foreign keys: explicit `@map()`
- Indexes on foreign key columns
- Cascade deletes on all relations

### Prisma Multi-File Schema
- Prisma v6.7.0+ required for GA multi-file support
- Layout: `backend/prisma/{schema.prisma (generator+datasource only), models/{user,auth,...}.prisma, migrations/}`
- Config: `package.json: { "prisma": { "schema": "./prisma" } }` or prisma.config.ts (v7+)
- Commands: `prisma validate`, `prisma generate`, `prisma migrate dev --schema ./prisma`
- Rules: schema.prisma must sit at configured schema root; do not split datasource without verifying version; avoid duplicate definitions

### Cross-cutting Structure
- `middleware/`, `exceptions/`, `utils/`, `types/`

## Frontend Architecture Standards (Next.js App Router)

### Directory Structure
- `app/` — Next.js App Router pages (layout, login, dashboard, etc.)
- `components/{feature}/` — React components grouped by domain
- `lib/api/` — API client + TanStack Query hooks
- `lib/providers/` — ReduxProvider, QueryProvider
- `lib/exceptions/` — Frontend exception class
- `store/` — Redux Toolkit (auth slice)
- `__tests__/` — Test setup

### Component Patterns
- Interactive components: `'use client'`
- Default exports only
- PascalCase matching filename
- Under 100 lines (guideline)
- Accessibility: aria-label, role="alert", role="progressbar", aria-invalid, keyboard support

### Styling Options
- Inline style props, Tailwind CSS, CSS modules, styled-components/Emotion, CSS variables + design tokens

### State Management
- Client state: Redux Toolkit (auth token, user, UI), Zustand, Jotai, MobX
- Server state: TanStack Query (standard)
- Forms: React Hook Form, Formik, TanStack Form
- No `useEffect` + `fetch` for server data in new code
- Redux slices must not store server-fetched entities
- Table UIs use TanStack Table before MUI DataGrid alternatives

### API Client & Hooks
- Axios instance with baseURL + timeout
- Request interceptor: attach Bearer token from localStorage
- Response interceptor: 401 → clear storage + redirect to /login
- SSR guard: `typeof window !== 'undefined'`
- Hook naming: `use{Resource}` or `use{Resource}With{Feature}`
- Response unwrapping: `response.data.data`
- Mutations return `response.data.data`
- Upload: `onUploadProgress` for progress tracking

### Error Handling
- `getApiErrorMessage(error)` helper extracting message from Axios error response
- Display errors with `role="alert"`, red color
- Frontend exception class (optional)

### TanStack-First Frontend
- **Server state:** TanStack Query
- **Routing:** TanStack Router (SPA) or Next App Router
- **Tables:** TanStack Table
- **Client UI:** Redux Toolkit or React state
- **Greenfield checklist:** QueryClientProvider, API hooks under src/lib/api/ with query keys, no useEffect+fetch, Redux slices must not store server entities, TanStack Table before MUI DataGrid
- Brownfield: new files TanStack-only; legacy Redux-server code → migration ticket; ADR for third-party widget exceptions
- Query key convention: `{ users: ['users'], user: (id) => ['users', id] }`

## Microservices Architecture
- Create separate service when: different scaling (CPU/memory-intensive), different runtime (Python for ML), different security boundaries, team ownership separation
- Service structure: `src/{routes, workflows, utils}/`, `server.ts`
- Inter-service communication: synchronous HTTP REST (backend → service), async message queue (planned), shared S3-compatible storage (MinIO)

## Cross-Cutting Standards

### D2 Diagrams

All architecture and flow diagrams must be authored as `.d2` files using the [D2](https://d2lang.com) declarative diagramming language and rendered to SVG.

- **Source:** `.d2` file committed alongside the document
- **Output:** SVG rendered via `d2` CLI, committed at the same path
- **Embedding:** `![description](path/to/diagram.svg)` in markdown
- **Syntax reference:** see `src/agent-archiver/references/d2-guide.md`
- **When to diagram:** system context in TDDs, user flows in milestones, architecture comparisons in ADRs, entity relationships in ERDs

### Constants Abstraction Pattern

All protocol-level and domain string values must be defined as named namespace objects — never inlined as raw strings.

**Pattern:**
```typescript
export const HeaderConstants = {
  APIKEY: 'x-api-key',
  TRACE_ID: 'x-trace-id',
} as const;

// Usage
req.headers[HeaderConstants.APIKEY]
```

**Rules:**
1. **Group by domain** — one file per domain (`header.constants.ts`, `error.constants.ts`, `config.constants.ts`)
2. **Namespace object** — `export const DomainConstants = { KEY: 'value' } as const`
3. **`as const`** — ensures literal types and autocomplete
4. **Bracket access** — `obj[DomainConstants.KEY]` for dynamic keys
5. **No inline protocol strings** — headers, status codes, config keys must be constants even if used once
6. **Env wrapper** — access env vars via `EnvConstants`, not `process.env` directly

### Naming Conventions
| Element | Convention | Example |
|---------|------------|---------|
| Files | Dot notation (lowercase) | `auth.service.ts` |
| Classes | PascalCase matching filename | `AuthService` |
| Functions | camelCase | `getUserById` |
| Variables | camelCase | `authHeader` |
| Constants | UPPER_SNAKE_CASE | `BCRYPT_COST` |
| Types/Interfaces | PascalCase | `AuthenticatedRequest` |
| Booleans | is/has/can prefix | `isAuthenticated` |
| Repositories | `{module}.repository.ts` | `auth.repository.ts` |
| Test files | Co-located, same name + `.test.ts` | `auth.service.test.ts` |
| Directories | kebab-case | `auth/`, `file/` |
| Component exports | Default export only | `export default function LoginForm()` |

### Testing Standards
- Unit: Jest + ts-jest, co-located `*.test.ts`, test pure logic with mocked deps
- Integration: Jest + Supertest (backend), Testing Library (frontend) — test HTTP endpoints with mocked services
- E2E: Playwright, `frontend/e2e/`, full user flows against running services
- Load/Perf: k6, `testing/load/k6-scripts/`, concurrent users and response times
- Visual Regression: Chromatic, Percy
- Mock external libs at module level with `jest.mock()`
- Use `jest.mocked()` for type safety
- NEVER call real DBs, JWT, bcrypt, S3, or fs in tests
- Coverage targets: service 90%+, routes 90%+, middleware 90%+, components 80%+

### k6 Load Testing Grid

| Layer | Protocol | Metrics |
|-------|----------|---------|
| Backend | HTTP — latency, throughput, error rate | p95 < 500ms |
| Microservices | Protocol — SLOs, timeouts | Per-service thresholds |
| Frontend | Browser module — web vitals, render, critical flows | LCP < 2.5s |

- Layout: `testing/load/{shared/config.js + thresholds.js, backend/*.js, microservices/*.js, frontend/*.browser.js}`
- NPM scripts: load:backend, load:frontend, load:all

### TSDoc Conventions

Every function — exported or internal — **MUST** have a TSDoc comment. Types and interfaces **MUST** have one as well. No exceptions.

**Format:**
```typescript
/**
 * One-line summary of what this does. Use action verbs.
 *
 * @param paramName - Description of the parameter
 * @returns Description of the return value
 * @throws {ExceptionName} When/why this throws
 */
```

**Rules:**
- Lead with a verb in present tense (Creates, Validates, Fetches, Transforms)
- `@param` — required for every parameter, describe what it represents
- `@returns` — required when return type isn't trivially obvious from the name
- `@throws` — required when the function throws a typed exception
- `@typeParam` — for generic type parameters when needed
- `@deprecated` — with reason + replacement path
- `@example` — optional, add when usage is non-trivial
- Do NOT restate what the type signature already says (`@param id - The id`)
- Test files included — every test function gets a brief TSDoc describing what scenario it covers
- Use `{@link OtherSymbol}` to cross-reference types and functions
- Use Markdown inside descriptions for lists or emphasis

**Frontend components:** Use a brief TSDoc above the component function describing what it renders and its props. Prop documentation via inline type is preferred over duplicating in `@param`.

**Example:**
```typescript
/**
 * Creates a new user record after validating the input payload.
 *
 * @param payload - Validated create-user DTO (guaranteed by controller)
 * @param options.ip - Request IP for audit logging
 * @returns The created user without the password hash
 * @throws {AuthException} When email already exists
 * @throws {ValidationException} When business rules fail
 */
async function createUser(
  payload: CreateUserDto,
  options: { ip: string }
): Promise<UserResponse> { ... }
```

### Code Quality Rules
- DO: Follow existing patterns, Zod validation in controllers (not services), typed exceptions, async/await, strict TS, co-located tests
- DO: Extract shared logic to `utils/` (backend) or `lib/` (frontend)
- DO: Write TSDoc for every function (see TSDoc Conventions above)
- DO: Use D2 diagrams (`.d2` → SVG) for architecture and flow visualizations
- DO: Define all protocol strings as namespace constant objects (`Container.KEY` pattern)
- DO NOT: Business logic in controllers, console.log in production, cross-imports between services, skip tests, commit secrets, use `.then()` chains, change AppResponse envelope format, add TODO/placeholder code, create one exception file per error type, commit any function without TSDoc, hardcode protocol strings inline

### Utility Taxonomy
- Start atomic, promote at ≥3 related functions; domain over generic; no business logic; max class size <150 lines
- Directory: `src/utils/{datetime/{formatIso, parseSafe, DateTimeUtils}, auth/password.utils.ts}`
- Naming: atomic (formatIso.ts), grouped (DateTimeUtils.ts), domain (auth/password.utils.ts)
- Tests: co-located *.test.ts per utility file

### Git Conventions
- Branch: `<type>/<short-description>` — types: feat, fix, chore, refactor, docs
- Commit: `<type>(<scope>): <description>` — scopes: backend, frontend, root, docs
- PR: title matches commit convention, description with what/why/how, link to artifacts, all tests pass, no console.log, no .env

### Architecture Principles
- Modules are independent — no cross-imports between modules
- Shared infrastructure at `src/` root
- Dependency direction: routes → controllers → services → repositories
- Prefer duplication over premature abstraction (extract after 3rd occurrence); document why duplication exists and when to remove it
- No separate shared-types library between frontend and backend — use `packages/dto/` for API boundary contracts only

### AI Agent Coding Rules
- Before: read rules.md, read engineering docs, read neighboring files, check deferred-decisions.md
- While: mirror module structure, use existing deps, use project abstractions, co-locate tests, add TSDoc to every function, no inline comments restating code, author diagrams as `.d2` files rendered to SVG, define protocol strings as namespace constants (`Container.KEY` pattern)
- After: verify compilation (`tsc --noEmit`), verify tests pass, update docs, update story artifacts
- Forbidden: console.log in production, sync I/O in handlers, business logic in controllers, per-error exception files, new state mgmt libs beyond RTK+TQ, `any` without justification, TODO placeholders, .env commits, any function without TSDoc, hardcoded protocol strings, hand-drawn or GUI-generated diagrams

## Developer Experience
- Local dev: Husky git hooks, Docker Compose for infrastructure, `cp .env.example .env`, `npx prisma generate`, `npx prisma migrate dev`, `npm run dev`
- Docker Compose services: postgres (16-alpine) + MinIO for S3-compatible storage
- Config safety: `.configignore` defines sensitive patterns, Husky auto-generates `.example` files pre-commit
- Hot reload: Nodemon for backend, Next.js fast refresh for frontend, Docker volume mounts
- Volta for Node version pinning (node 24.13.0, npm 10.5.0)
- Mirror Docs Cartographer (automated doc-sync tool): code paths → docs/ mirrored structure; post-commit: git diff HEAD~1 → filter (.ts, .tsx, .prisma, .js) → create/update mirrors → coverage report; tracked file types: TypeScript, Prisma, k6, config schemas

## Libraries of Choice
- Backend: Express 4.18, Prisma 5.x, Zod 3.x, paseto, bcryptjs, Multer, @aws-sdk/client-s3, http-status-codes, Pino, Jest/ts-jest, Supertest
- Frontend: Next.js 14, React 18, Redux Toolkit, TanStack Query, Axios, Tailwind CSS, Playwright
- Specialized: LangChain/LangGraph (AI workflows), sharp (image processing), pdf-lib (PDF manipulation)
- Dev tools: TypeScript, Husky, Volta, Docker Compose
- DTO package: `packages/dto/` with TypeScript types + Zod schemas, no business logic

## DX Guardrails (Cat #6)
- Dry-run destructive changes with explicit diff before apply
- Cap file changes per session with undo instructions
- YOLO defaults exist — one keystroke to skip Q&A
- Never more than 2 architecture choices at once
- Session memory: never re-ask answered questions
- Ship partial usable docs — do not block on perfection
- Incremental scan — only changed files, not whole repo

## What to Standardize Per Project

Use this table during project initialization to record decisions:

| Area | Backend | Frontend |
|------|---------|----------|
| Framework | — | — |
| API style | — | — |
| Styling | — | — |
| Components | — | — |
| Validation | — | — |
| ORM/Data | — | — |
| State (client) | — | — |
| State (server) | — | — |
| Forms | — | — |
| Auth | — | — |
| Logging | — | — |
| Testing | — | — |
| Realtime | — | — |
| Queue/Async | — | — |
| Cache | — | — |
| Monitoring | — | — |
| Hosting/Infra | — | — |
| CI/CD | — | — |
| Monorepo | — | — |
| Build tool | — | — |
| Runtime | — | — |
| Package mgr | — | — |
| Formatting | — | — |
| Architecture | — | — |


# Architecture Decision Records

## ADR 001 — Shared DTO Package
- **Status**: Accepted. **Date**: 2026-05-19. **Decider**: Prajwal
- Context: BE and FE must share API contracts without duplicating types; brainstorming resolved T1 for internal package at packages/dto/
- Decision: Create packages/dto/ as workspace-internal package; export TypeScript types + Zod schemas used at API boundaries; FE and BE depend via package name; no business logic in DTO package
- Versioning: lockstep with API changes; breaking changes require coordinated release
- Structure: `packages/dto/{package.json, tsconfig.json, src/{index.ts, auth/{auth.dto.ts, auth.types.ts}}}`
- Positives: single contract source, compiler catches drift
- Negatives: extra package to publish/build in CI
- Risks: scope creep (services in dto) — forbid via lint/rule
- Migration (brownfield): extract types from BE controllers → packages/dto/ → add FE dep on @workspace/dto → replace FE inline types with DTO imports → add lint rule

## ADR 002 — Repository Triad Pattern
- **Status**: Accepted. **Date**: 2026-05-19. **Decider**: Prajwal
- Context: data access needs consistent placement for schemas, domain models, cache layers; per-module *.repository.ts orchestrates calls
- Decision: `src/repository/{schemas/, models/, cache/}` + `src/modules/*/*.repository.ts` (orchestration only)
- Module repositories must not embed schema definitions inline; services call module repositories; repositories call triad layers
- Positives: predictable data layer, easier onboarding
- Negatives: more files per entity, migration effort on brownfield
- Risks: triad becomes dumping ground — enforce naming + review
- Migration (brownfield): create src/repository/ structure → move schemas → move models → extract cache → update module repositories

## ADR 003 — Promise Exception Registry
- **Status**: Accepted. **Date**: 2026-05-20. **Decider**: Prajwal
- Resolves T2 tension between constitution (async/await only) and braindump allowance for .then().catch()
- Default: async/await in application code; Exception: native callbacks in promisify, stream pipelines, third-party APIs with no async variant; Never: .then() for business logic
- Registry entries: `fs.promises` pipeline with streams (*.repository.ts, src/utils/stream*.ts — rationale: streams with known error types, backpressure control); `zlib` stream chaining (src/utils/compression*.ts — multi-step transform pipes); third-party callback APIs (*.adapter.ts — no native async variant)
- Process: PR → explain why async/await not viable → 1 senior reviewer approval → add entry with date
- Future: `@typescript-eslint/no-restricted-syntax` with allowlist


