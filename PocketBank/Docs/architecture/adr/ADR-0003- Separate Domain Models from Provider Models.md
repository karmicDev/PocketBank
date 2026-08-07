# ADR-0003: Separate Domain Models from Provider Models

- **Status:** Accepted
- **Date:** 2026-08-07
- **Decision Makers:** Michael Karbe
- **Related Issue:** PB-006 – Bootstrap PocketBankCore

---

# Context

PocketBank aims to support multiple banking providers over time.

Examples include:

- MockBank
- Plaid
- TrueLayer
- Nordigen
- Open Banking APIs
- Future proprietary providers

Each provider exposes its own API, naming conventions, data formats and business semantics.

Examples:

```json
{
    "account_id": "123",
    "current_balance": 1520.45
}
```

Another provider might return:

```json
{
    "id": 42,
    "balance": 1520.45,
    "currency": "EUR"
}
```

Although these payloads represent similar concepts, they are implementation details of the provider and must not become part of PocketBank's domain model.

Without clear separation, external APIs begin to shape the application's internal architecture, making future provider integrations increasingly difficult.

---

# Decision

PocketBank distinguishes between **Domain Models** and **Provider Models**.

The Core package owns the application's ubiquitous language and defines all domain concepts.

Provider packages own their transport models (DTOs), API clients and mapping logic.

External models are never exposed outside their provider package.

All translation into the PocketBank domain occurs inside the provider package.

---

# Architecture

```
                PocketBank App

                      │

                      ▼

               PocketBankCore

        Account
        Transaction
        Money
        Currency

                      ▲

                      │

         BankingProvider Protocol

      ┌───────────────┼────────────────┐
      │               │                │
      ▼               ▼                ▼

 PocketBankMockBank  PocketBankPlaid  PocketBankTrueLayer

      │               │                │

      ▼               ▼                ▼

  DTOs            DTOs            DTOs

      │               │                │

      ▼               ▼                ▼

   JSON             JSON             JSON
```

---

# Domain Models

Domain models represent business concepts used throughout the application.

Examples:

- Account
- Transaction
- Money
- Currency
- AccountIdentifier
- TransactionIdentifier

Domain models:

- are provider independent
- contain business meaning
- are stable over time
- define the language of the application

---

# Provider Models

Provider models represent external API contracts.

Examples:

- PlaidAccountDTO
- MockBankTransactionDTO
- TrueLayerBalanceResponse

Provider models:

- mirror external APIs
- may change whenever an API changes
- are implementation details
- are never exposed to the application

---

# Mapping

Each provider package is responsible for translating provider models into domain models.

Example:

```
JSON

↓

PlaidAccountDTO

↓

PlaidAccountMapper

↓

Account
```

The application only interacts with domain models.

---

# Rationale

## Stable Domain

External APIs change frequently.

The PocketBank domain should remain stable regardless of provider implementation details.

---

## Multiple Providers

Supporting additional providers should require only a new provider package.

The application itself should not require architectural changes.

---

## Encapsulation

Provider-specific implementation details remain isolated.

Knowledge of external APIs does not leak into the rest of the application.

---

## Testability

Mapping logic can be tested independently.

Domain logic can be tested without networking or API dependencies.

---

## Maintainability

Changes introduced by one provider remain localized to that provider package.

The risk of regressions across unrelated features is reduced.

---

# Consequences

## Positive

- Stable domain language
- Clean separation of concerns
- Easier onboarding of new providers
- Reduced coupling to external APIs
- Independent testing of mapping logic
- Better long-term maintainability
- Consistent architecture across providers

---

## Negative

- Additional mapping code
- More types to maintain
- Initial implementation requires more structure

---

# Alternatives Considered

## Use Provider Models Throughout the Application

Advantages

- Less code
- Faster initial development

Disadvantages

- Tight coupling to external APIs
- Provider terminology leaks into business logic
- Difficult provider replacement
- Frequent breaking changes across the application

---

## Shared DTO Layer

Advantages

- Centralized transport models

Disadvantages

- Blurs ownership
- Different providers expose different semantics
- Encourages accidental coupling between providers

---

# Decision Summary

PocketBank owns its domain model.

Every provider owns its own transport models and mapping layer.

The application communicates exclusively through provider-independent domain models defined in PocketBankCore.

This establishes a stable business language while allowing provider implementations to evolve independently.
