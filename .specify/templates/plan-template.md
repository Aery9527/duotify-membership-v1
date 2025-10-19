# Implementation Plan: [FEATURE]

**Branch**: `[###-feature-name]` | **Date**: [DATE] | **Spec**: [link]
**Input**: Feature specification from `/specs/[###-feature-name]/spec.md`

**⚠️ LANGUAGE REQUIREMENT**: This plan MUST be written in Traditional Chinese (zh-TW) per Constitution Principle V.

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

[Extract from feature spec: primary requirement + technical approach from research]

## Technical Context

<!--
  ACTION REQUIRED: Replace the content in this section with the technical details
  for the project. The structure here is presented in advisory capacity to guide
  the iteration process.
-->

**Language/Version**: [e.g., Python 3.11, Swift 5.9, Rust 1.75 or NEEDS CLARIFICATION]  
**Primary Dependencies**: [e.g., FastAPI, UIKit, LLVM or NEEDS CLARIFICATION]  
**Storage**: [if applicable, e.g., PostgreSQL, CoreData, files or N/A]  
**Testing**: [e.g., pytest, XCTest, cargo test or NEEDS CLARIFICATION]  
**Target Platform**: [e.g., Linux server, iOS 15+, WASM or NEEDS CLARIFICATION]
**Project Type**: [single/web/mobile - determines source structure]  
**Performance Goals**: [domain-specific, e.g., 1000 req/s, 10k lines/sec, 60 fps or NEEDS CLARIFICATION]  
**Constraints**: [domain-specific, e.g., <200ms p95, <100MB memory, offline-capable or NEEDS CLARIFICATION]  
**Scale/Scope**: [domain-specific, e.g., 10k users, 1M LOC, 50 screens or NEEDS CLARIFICATION]

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Verify compliance with constitution principles (`.specify/memory/constitution.md`):

**I. Code Quality & Maintainability**
- [ ] Design follows idiomatic Go patterns and Effective Go guidelines
- [ ] Error handling strategy defined (all errors explicitly handled)
- [ ] Logging approach specified (structured logging with context)
- [ ] Functions kept modular (<50 lines guideline)
- [ ] Code review plan includes gofmt, goimports, go vet checks

**II. Testing Standards (NON-NEGOTIABLE)**
- [ ] Test-first approach confirmed (tests written before implementation)
- [ ] Unit test plan achieves 80%+ coverage target
- [ ] Integration tests defined for API endpoints and database operations
- [ ] Contract tests planned for external service integrations
- [ ] Test execution time within limits (unit <5s, integration <30s)

**III. User Experience Consistency**
- [ ] API responses use standard JSON structure (success/error format)
- [ ] HTTP status codes semantically correct
- [ ] Input validation comprehensive (all fields validated)
- [ ] Error messages user-friendly and actionable
- [ ] API versioning strategy defined (path-based, e.g., /api/v1/)

**IV. Performance Requirements**
- [ ] Response time targets defined (read <100ms p95, write <200ms p95)
- [ ] Scalability design supports 1,000+ concurrent requests
- [ ] Database queries optimized (indexes planned, no N+1)
- [ ] Connection pooling implemented for external services
- [ ] Context timeouts configured (max 30s)
- [ ] Pagination implemented for list endpoints (max 100 items)

**V. Documentation Language Standard (NON-NEGOTIABLE)**
- [ ] Specification written in Traditional Chinese (zh-TW)
- [ ] Implementation plan written in Traditional Chinese (zh-TW)
- [ ] User-facing documentation written in Traditional Chinese (zh-TW)
- [ ] Code comments may be in English; code identifiers must be in English

**Quality Gates**
- [ ] Pre-commit checks planned (formatting, linting, local tests)
- [ ] CI/CD pipeline includes coverage and security checks
- [ ] Performance benchmarks defined for critical paths

## Project Structure

### Documentation (this feature)

```
specs/[###-feature]/
├── plan.md              # This file (/speckit.plan command output)
├── research.md          # Phase 0 output (/speckit.plan command)
├── data-model.md        # Phase 1 output (/speckit.plan command)
├── quickstart.md        # Phase 1 output (/speckit.plan command)
├── contracts/           # Phase 1 output (/speckit.plan command)
└── tasks.md             # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### Source Code (repository root)
<!--
  ACTION REQUIRED: Replace the placeholder tree below with the concrete layout
  for this feature. Delete unused options and expand the chosen structure with
  real paths (e.g., apps/admin, packages/something). The delivered plan must
  not include Option labels.
-->

```
# [REMOVE IF UNUSED] Option 1: Single project (DEFAULT)
src/
├── models/
├── services/
├── cli/
└── lib/

tests/
├── contract/
├── integration/
└── unit/

# [REMOVE IF UNUSED] Option 2: Web application (when "frontend" + "backend" detected)
backend/
├── src/
│   ├── models/
│   ├── services/
│   └── api/
└── tests/

frontend/
├── src/
│   ├── components/
│   ├── pages/
│   └── services/
└── tests/

# [REMOVE IF UNUSED] Option 3: Mobile + API (when "iOS/Android" detected)
api/
└── [same as backend above]

ios/ or android/
└── [platform-specific structure: feature modules, UI flows, platform tests]
```

**Structure Decision**: [Document the selected structure and reference the real
directories captured above]

## Complexity Tracking

*Fill ONLY if Constitution Check has violations that must be justified*

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| [e.g., 4th project] | [current need] | [why 3 projects insufficient] |
| [e.g., Repository pattern] | [specific problem] | [why direct DB access insufficient] |

