<!--
Sync Impact Report
- Version change: 0.0.0 (template placeholders) -> 1.0.0
- Modified principles: five placeholder principles replaced with Library-First
  Architecture, Secure-by-Default, Test-First Quality, Contract and Integration
  Testing, and Observable Simplicity.
- Added sections: Product and Technical Constraints; Development Workflow and
  Quality Gates.
- Removed sections: none.
- Follow-up TODOs: none.
-->

# ContosoDashboard Constitution

## Core Principles

### I. Library-First Architecture
Application capabilities MUST be organized behind focused, independently
testable services or components with explicit contracts. New business behavior
MUST be kept out of UI markup and infrastructure-specific code when a domain
service can own it. Public contracts MUST document inputs, outputs, failure
behavior, and authorization assumptions. This keeps the training application
replaceable, testable, and suitable for the planned storage-provider
abstraction.

### II. Secure-by-Default
Every feature MUST enforce authorization at the server boundary and MUST
validate all untrusted input before persistence or processing. Files MUST be
stored outside `wwwroot`, use generated identifiers rather than user-supplied
paths, pass allow-list and malware checks, and be served only through an
authorized endpoint. Secrets and connection strings MUST come from
configuration or environment-specific secret storage, never source control.
Security is a functional requirement because the dashboard handles employee
and project data.

### III. Test-First Quality
A change MUST have acceptance criteria and automated tests before it is
considered complete. Tests MUST cover successful behavior, validation failures,
authorization boundaries, and relevant persistence or file-system failures.
Implementation SHOULD follow a red-green-refactor cycle, and a defect fix MUST
include a regression test. This makes requirements executable and preserves
the instructional value of the repository.

### IV. Contract and Integration Testing
Changes to service interfaces, database models, storage behavior, identity
claims, or external integrations MUST include integration or contract tests
covering the changed boundary. Tests MUST verify that the local file-storage
implementation can be replaced by another implementation without changing
business behavior. Shared schemas and authorization flows MUST be tested
through the application boundary, not only through isolated mocks.

### V. Observable Simplicity
Solutions MUST use the simplest design that satisfies the approved
requirements; speculative abstractions and unrelated refactors are
prohibited. User-visible operations MUST provide actionable success and error
feedback, and server failures MUST produce structured, privacy-preserving
logs with enough context to diagnose the issue. Performance requirements MUST
be measured at the relevant boundary, including the documented two-second
search target. Simplicity and observability keep this training codebase
understandable and maintainable.

## Product and Technical Constraints

ContosoDashboard is a training-purpose ASP.NET Core application targeting
.NET 10. Features MUST respect the stakeholder requirements for employee,
team-lead, project-manager, and administrator roles. Document uploads MUST
enforce the supported-type allow-list and 25 MB per-file limit, preserve
required metadata, and maintain the sequence of generating a unique path,
saving the file, and then saving metadata. Search and download results MUST
be filtered by the caller's permissions. The application MUST remain suitable
for local development without requiring production-only infrastructure, while
storage access MUST be isolated behind an interface so Azure Blob Storage can
be introduced without changing business logic.

## Development Workflow and Quality Gates

Every feature MUST begin with an approved specification or an explicitly
recorded maintenance rationale. The implementation plan MUST identify affected
contracts, data migrations, security controls, and tests. A pull request MUST
include automated test results and a concise description of authorization,
data-handling, and operational impacts. Before merge, the project MUST build
cleanly, applicable tests MUST pass, and reviewers MUST verify compliance with
this constitution. Database or file-format changes MUST include a migration or
rollback note. Documentation MUST be updated when behavior, configuration, or
operator steps change.

## Governance

This constitution is the highest-level project guidance for design and review;
feature specifications, plans, and implementation details MUST conform to it.
When a conflict is found, the contributor MUST document the conflict and
resolve it in favor of this constitution or propose an amendment before
merging the change.

Amendments MUST be made through the constitution workflow, include a Sync
Impact Report, state the affected principles and migration implications, and
receive review from the project owner. Any amendment that changes behavior
MUST identify affected specifications and implementation work. Reviewers MUST
perform a compliance check for every change that affects security, data
handling, contracts, or quality gates.

Constitution versions use semantic versioning. A MAJOR increment is required
for removing or redefining a principle in a backward-incompatible way. A
MINOR increment is required for a new principle or materially expanded
governance requirement. A PATCH increment is used for clarifications, wording,
and non-semantic corrections. The constitution MUST record the amendment date
and retain the original ratification date.

**Version**: 1.0.0 | **Ratified**: 2026-09-15 | **Last Amended**: 2026-09-15
