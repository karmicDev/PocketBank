# PocketBank Architecture

## Status

This architecture intentionally favors clarity and scalability over minimalism.

As the project evolves, architectural decisions are documented through Architecture Decision Records (ADRs).

## Overview

PocketBank is a production-inspired iOS application that demonstrates modern Apple platform development practices.

The project emphasizes maintainability, modularity, testability and scalability over rapid feature implementation.

It is intentionally structured to resemble a codebase maintained by multiple engineering teams.

---

## Architectural Principles

PocketBank follows these principles:

- Feature-first organization
- Dependency Inversion
- Protocol-Oriented Programming
- Composition over inheritance
- Swift Concurrency
- Value semantics whenever possible
- Testability by design
- Infrastructure separated from business logic

Every architectural decision should reinforce one or more of these principles.

---

# Layers

The application is divided into several layers.

┌─────────────────────────────┐
│          App                │
├─────────────────────────────┤
│         Features            │
├─────────────────────────────┤
│      Domain / Core          │
├─────────────────────────────┤
│     Infrastructure          │
└─────────────────────────────┘

Dependencies always point downward.

Features never depend directly on infrastructure.

Infrastructure never depends on features.

---

# Feature Modules

Every feature owns:

- Views
- ViewModels
- Navigation
- Local Models
- Feature-specific Services

Example:

Features/

Authentication/

Dashboard/

Transactions/

Cards/

Settings/

Features should communicate only through public interfaces.

---

# Core

Core contains reusable business abstractions.

Examples:

- BankingService
- AuthenticationService
- Analytics
- Persistence
- Networking abstractions

Core never knows about SwiftUI.

Core never depends on external APIs.

---

# Infrastructure

Infrastructure provides concrete implementations.

Examples:

- MockBank API
- TrueLayer API
- Local Mock Provider
- Keychain
- URLSession
- Logging

Infrastructure implements protocols defined by Core.

---

# Provider Abstraction

The application never depends on a specific banking provider.

Instead, Core defines interfaces.

Example:

BankingService

↓

MockBankService

TrueLayerService

LocalMockService

The active provider is selected using dependency injection.

This enables:

- testing
- local development
- provider replacement
- future expansion

without changing feature code.

---

# Dependency Direction

Allowed:

Feature
↓
Core
↓
Infrastructure

Not allowed:

Infrastructure
→ Feature

Feature
→ Feature

Core
→ SwiftUI

---

# Data Flow

User Action

↓

View

↓

ViewModel

↓

Service Protocol

↓

Infrastructure

↓

Remote API

↓

Decoded Models

↓

ViewModel

↓

View

Data flows in one direction.

---

# Dependency Injection

Dependencies are injected.

Views never instantiate services directly.

Bad:

DashboardViewModel()

creates

MockBankService()

Good:

DashboardViewModel(
    bankingService: BankingService
)

---

# Concurrency

PocketBank adopts Swift Concurrency throughout the application.

Guidelines:

- async/await instead of callbacks
- Actors for shared mutable state
- Structured Concurrency
- MainActor only for UI updates

Avoid:

DispatchQueue

Completion handlers

Shared mutable state

unless required by platform APIs.

---

# Testing Strategy

Every layer has its own testing responsibilities.

Feature

- ViewModel tests
- Navigation tests

Core

- Business logic tests

Infrastructure

- API decoding
- Networking
- Persistence

UI

- Snapshot tests
- UI automation

Testing should rely on protocols and mocks instead of concrete implementations.

---

# Future Growth

The architecture should allow:

- multiple banking providers
- offline mode
- widgets
- App Intents
- watchOS companion
- analytics providers
- feature flags

without significant restructuring.
