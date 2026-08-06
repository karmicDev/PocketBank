# ADR-0002: Use Independently Versioned Swift Packages for Shared Foundations

- **Status:** Accepted
- **Date:** 2026-08-06
- **Decision Makers:** Michael Karbe
- **Related Issue:** PB-005 – Create Swift Package Module Foundation

---

# Context

PocketBank is intended to demonstrate production-quality software architecture and engineering practices rather than simply deliver application features.

The project follows a layered, feature-oriented architecture (ADR-0001), where shared functionality is separated from product-specific features.

An important architectural decision is how these shared foundations should be organized and distributed.

There are two common approaches:

1. A monorepo containing the application and all packages.
2. Independent Swift packages maintained in their own repositories.

The chosen approach should reinforce architectural boundaries, ownership, maintainability and long-term scalability.

---

# Decision

Shared foundational components will be developed as **independently versioned Swift Packages**, each maintained in its own repository.

The PocketBank application consumes these packages as external dependencies using Swift Package Manager.

Each package is responsible for its own:

- source code
- tests
- documentation
- semantic versioning
- release history
- CI/CD pipeline

The PocketBank application repository remains responsible for:

- application composition
- product features
- documentation
- architectural decisions

---

# Initial Package Structure

```
karmicDev/

├── PocketBank
│
├── PocketBankCore
│
├── PocketBankNetworking
│
├── PocketBankDesignSystem
│
└── PocketBankTestingSupport
```

---

# Responsibilities

## PocketBankCore

Contains domain models and business contracts.

Examples:

- Account
- Transaction
- Money
- BankingService
- AuthenticationService
- Domain errors

This package contains no UI or networking implementation.

---

## PocketBankNetworking

Contains reusable networking infrastructure.

Examples:

- HTTPClient
- Endpoint
- Request Builder
- Request Execution
- Error Mapping

This package contains no banking-specific logic.

---

## PocketBankDesignSystem

Contains reusable UI components and design tokens.

Examples:

- Buttons
- Cards
- Typography
- Colors
- Icons
- Loading Views

Business logic is intentionally excluded.

---

## PocketBankTestingSupport

Provides reusable testing infrastructure.

Examples:

- Test Factories
- Mock Services
- Fixtures
- Shared Assertions
- Test Utilities

This package is consumed only by test targets.

---

# Rationale

## Explicit Module Boundaries

Swift Package Manager enforces dependencies at the compiler level.

Modules expose only their public APIs, reducing accidental coupling.

---

## Independent Versioning

Each package can evolve independently using Semantic Versioning.

Breaking changes become explicit rather than accidental.

---

## Clear Ownership

Each package represents a clearly defined architectural responsibility.

This mirrors platform-team ownership commonly found in larger engineering organizations.

---

## Reusability

Shared foundations are intentionally designed to be reusable outside the PocketBank application.

Future projects may reuse these packages without modification.

---

## Improved Testing

Each package owns its own test suite.

Business logic, networking and UI components can be validated independently from the application.

---

## Separation of Product and Platform

The PocketBank repository focuses on delivering banking features.

Shared packages focus on platform capabilities.

This separation encourages clean architectural boundaries.

---

# Consequences

## Positive

- Strong compiler-enforced module boundaries
- Independent semantic versioning
- Smaller repositories with focused responsibilities
- Better package reusability
- Clear ownership model
- Independent testing
- Easier long-term maintenance
- Demonstrates production-oriented architecture

---

## Negative

- Increased repository management
- Additional CI/CD configuration
- Coordinated dependency updates
- More release management
- Higher initial setup cost

---

# Alternatives Considered

## Monorepo with Local Packages

Advantages

- Simpler development workflow
- Atomic commits across packages
- Easier onboarding
- Single repository

Disadvantages

- Weaker ownership boundaries
- Shared version history
- Packages are less independently reusable

---

## Single Xcode Project

Advantages

- Fast initial development
- Minimal setup
- Suitable for small applications

Disadvantages

- No compiler-enforced module boundaries
- Increased coupling
- Limited scalability
- Poor representation of production architecture

---

# Decision Summary

PocketBank adopts independently versioned Swift Packages for all shared foundational components.

This decision reinforces architectural boundaries, improves modularity, supports independent evolution, and better reflects the engineering practices used in scalable production systems.

Feature development remains within the PocketBank application repository, while reusable platform capabilities are developed and maintained as independent packages.
