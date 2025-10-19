<!--
=============================================================================
SYNC IMPACT REPORT - Constitution Update
=============================================================================
Version Change: 1.0.0 → 1.1.0 (MINOR - New principle added)

Modified Principles:
  - NEW: V. Documentation Language Standard (NON-NEGOTIABLE)

Added Sections:
  - Documentation Language Standard principle

Removed Sections: None

Templates Requiring Updates:
  ✅ spec-template.md - Language requirement note added
  ✅ plan-template.md - Language requirement note added

Follow-up TODOs: None

Previous Changes (v1.0.0):
  Initial constitution ratified for duotify-membership-v1 service.
  Established core principles:
  - I. Code Quality & Maintainability
  - II. Testing Standards (NON-NEGOTIABLE)
  - III. User Experience Consistency
  - IV. Performance Requirements

Current Amendment (v1.1.0):
  Added documentation language standard requiring Traditional Chinese (zh-TW)
  for all specifications, plans, and user-facing documentation.
  This ensures consistency for the target Taiwanese market.
=============================================================================
-->

# Duotify Membership Service Constitution

## Core Principles

### I. Code Quality & Maintainability

**Non-Negotiable Standards:**
- MUST write idiomatic Go code following [Effective Go](https://go.dev/doc/effective_go) guidelines
- MUST handle all errors explicitly; no silent failures or ignored errors
- MUST use structured logging with context (no fmt.Println in production code)
- MUST document all exported functions, types, and packages with godoc comments
- MUST keep functions under 50 lines; complex logic MUST be broken into composable units
- MUST use meaningful variable and function names that express intent
- MUST run `gofmt`, `goimports`, and `go vet` before every commit

**Rationale:**
Clean, maintainable code reduces cognitive load, prevents bugs, and enables team scalability. Go's idioms exist to create consistent, readable codebases. Explicit error handling is critical for reliability in production systems.

### II. Testing Standards (NON-NEGOTIABLE)

**Test-First Development:**
- Tests MUST be written before implementation (Red-Green-Refactor cycle)
- All tests MUST fail initially, then pass after implementation
- No feature is complete without passing tests

**Coverage Requirements:**
- MUST maintain minimum 80% code coverage across the codebase
- MUST have unit tests for all business logic and service layers
- MUST have integration tests for API endpoints and database interactions
- MUST have contract tests for external service integrations

**Test Quality Standards:**
- Tests MUST be deterministic and isolated (no shared state)
- Tests MUST clean up resources (use t.Cleanup() or defer)
- Tests MUST use table-driven tests for multiple scenarios
- Tests MUST run in under 5 seconds for unit tests, 30 seconds for integration suite
- MUST use testify/assert or similar for clear failure messages

**Rationale:**
Test-first development catches design flaws early. High test coverage ensures reliability and enables confident refactoring. Fast tests enable rapid iteration.

### III. User Experience Consistency

**API Response Standards:**
- All REST APIs MUST return consistent JSON structure:
  - Success: `{"success": true, "data": {...}}`
  - Error: `{"success": false, "error": {"code": "ERROR_CODE", "message": "...", "details": {...}}}`
- HTTP status codes MUST be semantically correct (200/201/400/401/403/404/500)
- Error messages MUST be user-friendly and actionable
- MUST include request ID in responses for traceability

**Validation Standards:**
- MUST validate all input at API boundaries
- MUST return comprehensive validation errors (all fields, not just first failure)
- MUST sanitize user input to prevent injection attacks

**API Versioning:**
- MUST version APIs using path prefix (e.g., `/api/v1/memberships`)
- MUST maintain backward compatibility within major versions
- Breaking changes MUST increment major version

**Rationale:**
Consistent API patterns reduce client integration errors and support costs. Clear error handling improves developer experience and debugging efficiency.

### IV. Performance Requirements

**Response Time Targets:**
- MUST respond to read operations (GET) in under 100ms at p95
- MUST respond to write operations (POST/PUT) in under 200ms at p95
- MUST respond to bulk operations in under 1 second at p95

**Scalability Standards:**
- MUST handle 1,000 concurrent requests without degradation
- MUST support horizontal scaling (stateless design)
- MUST implement connection pooling for database and external services
- MUST use context timeouts (max 30s for requests)

**Resource Efficiency:**
- MUST limit memory usage to under 512MB per instance under normal load
- MUST implement graceful degradation under high load
- MUST use efficient data structures (avoid N+1 queries)
- MUST implement pagination for list endpoints (max 100 items per page)

**Rationale:**
Performance directly impacts user satisfaction and operational costs. Defined targets enable performance regression detection and capacity planning.

### V. Documentation Language Standard (NON-NEGOTIABLE)

**Language Requirements:**
- All specifications (spec.md files) MUST be written in Traditional Chinese (zh-TW)
- All implementation plans (plan.md files) MUST be written in Traditional Chinese (zh-TW)
- All user-facing documentation MUST be written in Traditional Chinese (zh-TW)
- API documentation and user guides MUST be in Traditional Chinese (zh-TW)
- README files targeting end-users MUST be in Traditional Chinese (zh-TW)

**Exceptions:**
- Code comments and godoc comments MAY be in English for technical accuracy
- Variable names, function names, and code identifiers MUST be in English
- Technical error codes and log messages MAY be in English
- Git commit messages MAY be in English
- Internal developer documentation MAY be in English if marked as such

**Rationale:**
Duotify serves the Taiwanese market. Consistent use of Traditional Chinese in specifications and user-facing documentation ensures clarity for stakeholders, reduces miscommunication, and provides better user experience for the target audience.

## Performance Standards

### Benchmarking Requirements
- MUST include Go benchmarks for critical paths (authentication, membership queries)
- Benchmarks MUST run in CI and fail if performance regresses by >10%
- MUST profile CPU and memory for endpoints handling >10 req/s

### Caching Strategy
- MUST implement caching for frequently accessed, rarely changing data
- Cache invalidation MUST be explicit and correct
- MUST document cache TTL and invalidation strategies

### Database Performance
- All queries MUST have appropriate indexes
- MUST use prepared statements to prevent SQL injection
- MUST implement query timeouts (max 5s)
- N+1 queries are PROHIBITED; MUST use joins or batch loading

## Quality Gates

### Pre-Commit Checks (Developer Workstation)
- `gofmt -s` and `goimports` formatting
- `go vet` static analysis
- `golangci-lint run` with strict configuration
- All tests pass locally

### Pre-Merge Checks (CI/CD)
- All unit and integration tests pass
- Code coverage meets 80% minimum threshold
- No high/critical security vulnerabilities (go list -m all | nancy sleuth)
- API contract tests pass
- Performance benchmarks within acceptable range
- Documentation builds without errors

### Pre-Release Checks
- Full integration test suite passes in staging environment
- Load testing validates performance targets
- Security scan completed
- Release notes updated
- Database migration scripts tested

## Governance

**Constitution Authority:**
This constitution supersedes all other development practices and guidelines. When conflicts arise, constitution principles take precedence.

**Amendment Process:**
- Amendments require team consensus and must be documented in git history
- MAJOR version bump: Removing principles or adding non-negotiable constraints
- MINOR version bump: Adding new principles or significantly expanding guidance
- PATCH version bump: Clarifications, typo fixes, non-semantic refinements

**Compliance Verification:**
- All pull requests MUST be reviewed for constitution compliance
- Code reviewers MUST verify principle adherence before approval
- Automated CI checks MUST enforce quality gates
- Quarterly constitution review to ensure principles remain relevant

**Complexity Justification:**
Any violation of principles MUST be justified in writing with:
- Why the violation is necessary
- What simpler alternatives were rejected and why
- Plan to remediate the violation if possible

**Development Guidance:**
For runtime development practices and workflow details, refer to `AGENTS.md` and `.github/prompts/` slash command documentation.

**Version**: 1.1.0 | **Ratified**: 2025-10-19 | **Last Amended**: 2025-10-19
