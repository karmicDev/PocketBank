# ADR-0004: Domain Model Design Principles

- **Status:** Accepted
- **Date:** 2026-08-07
- **Decision Makers:** Michael Karbe
- **Related Issue:** PB-006 – Bootstrap PocketBankCore

---

# Context

PocketBankCore defines the application's business domain.

As the foundation of the PocketBank ecosystem, the domain layer should remain stable, expressive and independent of presentation, networking and persistence concerns.

Without clear modelling principles, domain objects tend to become inconsistent over time. Different engineers may represent similar concepts differently, leading to unnecessary complexity and reduced maintainability.

This ADR establishes a common set of modelling principles that every domain model should follow.

---

# Decision

PocketBank adopts a domain model that emphasizes:

- explicit business concepts
- value semantics
- immutability
- strong typing
- framework independence
- protocol-oriented design

Every domain model should represent a meaningful business concept rather than a technical implementation detail.

---

# Principles

## Model Business Concepts

Domain types represent the language of the business.

Examples:

- Account
- Transaction
- Money
- Currency
- Transfer
- Customer

Technical concerns such as JSON payloads, API responses or persistence models are not part of the domain.

---

## Prefer Value Types

`struct` is the default choice for domain models.

Value semantics improve:

- predictability
- thread safety
- reasoning about state
- testability

Reference types (`class`) should only be introduced when identity or shared mutable state is explicitly required.

---

## Immutable by Default

Domain models should expose immutable state.

Properties are declared using `let` whenever possible.

State changes are represented by creating new values rather than mutating existing ones.

---

## Strongly Typed Identifiers

Primitive types such as `UUID` or `String` should not be exposed directly as identifiers.

Instead, identifiers are modelled as dedicated domain types.

Examples:

- AccountID
- TransactionID
- CustomerID

This improves readability and prevents accidental misuse.

---

## Explicit Value Objects

Concepts such as Money and Currency are modelled as dedicated value types.

Business concepts should never be represented using primitive values alone.

Example:

Preferred:

```
Money(amount: 125.50, currency: .eur)
```

Avoid:

```
Double
```

---

## Protocol Conformance

Domain models should conform to standard Swift protocols whenever appropriate.

Typical conformances include:

- Sendable
- Hashable
- Codable
- Equatable

Protocol conformance should communicate capability rather than satisfy framework requirements.

---

## Framework Independence

PocketBankCore must not depend on:

- SwiftUI
- UIKit
- URLSession
- CoreData
- SwiftData
- third-party libraries

The domain remains independent of all infrastructure and presentation technologies.

---

## Explicit Business Rules

Business rules belong inside the domain.

Validation and invariants should be expressed through the domain model rather than scattered throughout the application.

---

# Rationale

## Readability

Business concepts become immediately understandable.

The code reflects the language of the problem domain.

---

## Maintainability

Consistent modelling principles reduce ambiguity and improve long-term maintainability.

---

## Testability

Pure value types without framework dependencies are straightforward to test.

---

## Concurrency

Immutable value types naturally support Swift Concurrency and reduce shared mutable state.

---

## Stability

The domain evolves more slowly than external APIs, UI frameworks or persistence technologies.

Keeping the domain isolated protects it from infrastructure changes.

---

# Consequences

## Positive

- Consistent domain model
- Strong compiler guarantees
- Improved readability
- Better thread safety
- Easier testing
- Reduced coupling
- Stable business language
- Foundation for future provider integrations

---

## Negative

- Additional wrapper types
- More upfront modelling effort
- Slightly higher initial complexity

---

# Alternatives Considered

## Primitive Types Everywhere

Examples:

- UUID
- String
- Double

Advantages

- Less code
- Faster initial implementation

Disadvantages

- Weak domain language
- Easier misuse
- Reduced readability
- Primitive obsession

---

## Mutable Reference Types

Advantages

- Familiar object-oriented design
- Easy in-place mutation

Disadvantages

- Shared mutable state
- Harder concurrency guarantees
- Reduced predictability
- More complex reasoning about ownership

---

## Framework-Oriented Models

Using UI or persistence models as the domain.

Advantages

- Less mapping

Disadvantages

- Tight coupling
- Poor separation of concerns
- Infrastructure influences business logic

---

# Decision Summary

PocketBankCore models the business domain using immutable, strongly typed value-oriented Swift models.

The domain expresses business concepts rather than technical implementation details and remains independent of frameworks, providers and infrastructure.

These principles provide a consistent foundation for all future domain modelling within the PocketBank ecosystem.
