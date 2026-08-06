# ADR-0001: Layered, Feature-Oriented Architecture

- **Status:** Accepted
- **Date:** 2026-08-06
- **Decision Makers:** Michael Karbe
- **Related Issue:** PB-002 – Define Application Architecture

---

# Context

PocketBank is intended to be a production-quality reference application that demonstrates modern iOS engineering practices.

The project has several goals:

- Demonstrate scalable software architecture
- Support long-term maintainability
- Encourage clear separation of concerns
- Make features independently testable
- Allow infrastructure to evolve without affecting business logic
- Provide an architecture suitable for collaboration across multiple engineering teams

A traditional MVVM application often becomes organized around screens and UI concerns. As the application grows, networking, persistence, authentication, analytics, and business logic frequently become tightly coupled to presentation code.

While MVVM remains an excellent presentation pattern, it does not define the overall architecture of a large application.

---

# Decision

PocketBank will adopt a **layered, feature-oriented architecture**.

The application will be organized into independent feature modules that communicate through abstractions defined in the Core layer.

MVVM will be used **within individual features** as the presentation pattern, but **MVVM is not considered the application's architecture**.

The architecture consists of the following layers:

```
App
│
├── Features
│
├── Core
│
└── Infrastructure
```

Each layer has clearly defined responsibilities and dependency rules.

Dependencies always point downward.

```
Features
    ↓
Core
    ↓
Infrastructure
```

No layer may depend on a higher layer.

---

## Project Structure

PocketBank uses an Xcode Workspace combined with Swift Package Manager modules.

The workspace is responsible for composing the application from independent modules.

Example:

PocketBank.xcworkspace

├── PocketBankApp
│
├── Features
│
├── Core
│
├── Infrastructure
│
└── DesignSystem

---

# Rationale

## Separation of Concerns

Each layer owns a single responsibility.

- Features focus on user experience.
- Core defines business capabilities and contracts.
- Infrastructure provides technical implementations.

This separation improves maintainability and readability.

---

## Scalability

As the application grows, additional features can be added without significantly affecting existing modules.

Examples:

- Investments
- Savings
- Notifications
- Widgets
- Apple Watch
- App Intents

Each feature can evolve independently.

---

## Testability

Business logic depends on protocols rather than concrete implementations.

This enables:

- Mock implementations
- Fast unit tests
- Isolated feature testing
- Predictable behavior

without requiring live services.

---

## Replaceable Infrastructure

Networking providers, persistence frameworks, analytics platforms, or authentication mechanisms may change over time.

Because Features depend only on Core abstractions, these implementations can be replaced without modifying presentation code.

Examples:

- MockBank
- TrueLayer
- Local Mock Provider

All implement the same contracts defined by Core.

---

## Team Collaboration

Feature-oriented organization reduces merge conflicts and ownership ambiguity.

Engineers can work independently on different features while sharing common infrastructure.

This mirrors the structure commonly found in large production applications.

---

# Consequences

## Positive

- Clear architectural boundaries
- High testability
- Improved maintainability
- Easier onboarding
- Better scalability
- Infrastructure can evolve independently
- Supports dependency injection naturally
- Encourages protocol-oriented design
- Easier feature ownership
- Well suited for long-term projects

---

## Negative

- More files and folders
- Higher initial complexity
- More abstractions than a small application requires
- Slightly slower initial development
- Requires discipline to maintain architectural boundaries

---

# Alternatives Considered

## Traditional MVVM Application

Advantages

- Simple
- Easy to understand
- Fast initial development

Disadvantages

- Presentation layer often becomes responsible for networking and business logic
- Increasing coupling as the application grows
- Difficult to scale across multiple teams
- Lower separation of concerns

---

## VIPER

Advantages

- Excellent separation of responsibilities
- Highly testable

Disadvantages

- Significant boilerplate
- Higher cognitive overhead
- Less suitable for this project's educational goals

---

## The Composable Architecture (TCA)

Advantages

- Strong unidirectional data flow
- Excellent testability
- Powerful tooling

Disadvantages

- Additional framework dependency
- Steeper learning curve
- More ceremony than required for this project

---

# Decision Summary

PocketBank adopts a layered, feature-oriented architecture because it provides clear architectural boundaries, promotes maintainability, enables dependency inversion, and scales well as the application grows.

MVVM remains the presentation pattern within individual features, but it is intentionally not treated as the application's overall architecture.

This decision establishes the architectural foundation for all future development within the project.
