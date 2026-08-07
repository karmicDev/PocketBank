# ADR-0005: Account and Balance Modelling

* **Status:** Accepted
* **Date:** 2026-08-07
* **Decision:** Model account balance as derived domain state, with transactions as the source of truth.

---

## Context

A bank account has a monetary balance that changes over time as money moves into and out of the account.

A simple model could represent an account as:

```swift
struct Account {
    let id: AccountID
    let balance: Money
}
```

This is straightforward, but it introduces an important question:

> Is the balance the source of truth, or is it a projection of the account's financial activity?

In a banking domain, transactions represent the actual movement of money. A balance is therefore conceptually the result of applying those transactions to an account.

The decision affects:

* Domain modelling
* Data consistency
* Persistence
* Synchronization
* Offline behavior
* Transaction history
* Error recovery
* Testing
* Future extensibility

PocketBank is not intended to implement a production banking ledger. However, its domain model should reflect sound financial and software-engineering principles while remaining appropriately simple for the scope of the project.

---

## Decision

We will model an `Account` as an entity whose balance is **derived from its financial transactions**.

Transactions are the source of truth for account activity.

Conceptually:

```text
Transactions
     │
     ▼
  Account
     │
     ▼
  Balance
```

The balance is therefore a projection of the account's transaction history rather than an independently authoritative value.

---

## Account Identity

An account is an entity and therefore has identity independent of its current state.

The identity will be represented by the existing:

```swift
AccountID
```

The account's identity remains stable even when its balance or other properties change.

Conceptually:

```swift
struct Account {
    let id: AccountID
    // additional account properties
}
```

`AccountID` is therefore preferred over using a raw `UUID` throughout the domain.

---

## Balance Representation

The balance will be represented using the existing `Money` value object.

```swift
Money(
    amount: Decimal,
    currency: Currency
)
```

An account has a single account currency for the scope of PocketBank.

For example:

```text
Account
    ID: 123...
    Currency: EUR

Transactions
    +100 EUR
    -25 EUR
    -10 EUR

Balance
    65 EUR
```

A balance must never combine incompatible currencies.

---

## Transactions as the Source of Truth

Transactions represent financial activity associated with an account.

For example:

```text
Transaction 1: +100 EUR
Transaction 2:  -25 EUR
Transaction 3:  -10 EUR
```

The resulting balance is:

```text
100 - 25 - 10 = 65 EUR
```

Conceptually:

```swift
balance =
    transactions
        .map(\.amount)
        .reduce(.zero, +)
```

The exact implementation will be decided when the `Transaction` domain model is introduced.

The important architectural rule is:

> The balance must not become an independently authoritative representation of the same financial information.

---

## Why Not Store Balance as Independent State?

A model such as:

```swift
struct Account {
    let id: AccountID
    let balance: Money
}
```

is attractive because it is simple and efficient.

However, treating this value as the source of truth creates the possibility of inconsistency.

For example:

```text
Transaction history:
+100 EUR
-25 EUR

Expected balance:
75 EUR

Stored balance:
80 EUR
```

There would be no reliable way for the domain to determine which value is correct.

With transactions as the source of truth:

```text
Transactions
     │
     ▼
Calculated balance
```

the balance can always be reconstructed.

---

## Balance as a Projection

Although the balance is derived conceptually, the architecture does not prohibit caching or persisting a calculated balance for performance.

For example, a future persistence layer may store:

```text
Transactions
Cached balance
```

However, such a cached balance is a **projection** and not the authoritative source.

If the cached value conflicts with the transaction history, the transaction history wins.

Conceptually:

```text
              ┌─────────────────┐
              │   Transactions  │
              │  Source of Truth│
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │ Balance         │
              │ Projection      │
              └─────────────────┘
```

This allows future optimizations without changing the domain model.

---

## Offline and Synchronization Considerations

A mobile application may temporarily operate with incomplete or stale transaction data.

Therefore, the UI may display a balance that represents the most recently synchronized transaction state.

The domain should not assume that locally available data is necessarily the complete financial history.

Synchronization concerns belong outside the `Account` entity.

The architecture should distinguish between:

```text
Domain truth
```

and:

```text
Locally available projection
```

This keeps networking, synchronization, and persistence concerns outside the core domain model.

---

## Pending Transactions

PocketBank will eventually need to distinguish between transactions that have been settled and transactions that are pending.

For example:

```text
Current balance:       1,000 EUR
Pending transaction:    -100 EUR
Available balance:       900 EUR
```

These are different concepts.

Therefore, this ADR does **not** define the final semantics of:

* Current balance
* Available balance
* Pending transactions
* Reserved amounts

Those decisions will be made when the transaction and account-availability models are introduced.

The important rule is that pending financial activity must not be silently treated as settled transaction history.

---

## Multi-Currency Considerations

An account has one currency within the scope of PocketBank.

For example:

```text
EUR Account
```

contains:

```text
+100 EUR
-25 EUR
```

but not:

```text
+100 EUR
+50 USD
```

Currency conversion is explicitly outside the responsibility of `Account` and `Money`.

Currency conversion will later be represented through a separate capability such as:

```swift
CurrencyConverter
```

as established during the `Money` design.

---

## Alternatives Considered

### Option A — Store Balance Directly

```swift
struct Account {
    let id: AccountID
    let balance: Money
}
```

#### Advantages

* Simple
* Fast access
* Easy to serialize
* Easy to display

#### Disadvantages

* Balance can become inconsistent with transactions
* Transaction history becomes secondary
* Recovery is more difficult
* Auditing becomes harder
* Synchronization conflicts become more difficult to reason about

**Rejected** as the authoritative domain representation.

---

### Option B — Derive Balance from Transactions

```text
Transactions
     ↓
Balance
```

#### Advantages

* Transactions are the source of truth
* Balance can be reconstructed
* Strong auditability
* Easier consistency reasoning
* Naturally represents financial history
* Allows cached balance projections later

#### Disadvantages

* Requires transaction modelling
* Potentially more expensive to calculate
* Requires consideration of pending transactions and synchronization

**Accepted.**

---

### Option C — Event Sourcing

Represent every account change as an immutable domain event and rebuild account state by replaying events.

For example:

```text
AccountOpened
MoneyDeposited
MoneyWithdrawn
MoneyTransferred
```

#### Advantages

* Complete history
* Excellent auditability
* State can be reconstructed
* Natural fit for financial systems

#### Disadvantages

* Significant complexity
* Requires event-store architecture
* More infrastructure than PocketBank currently needs
* Would distract from the mobile engineering objectives of the project

**Rejected for now.**

The decision to derive balance from transactions does not prevent a future event-sourced architecture.

---

## Consequences

### Positive

* Transactions remain the authoritative financial information.
* Account balance has a clear semantic meaning.
* The domain avoids duplicate sources of truth.
* Balance can be reconstructed.
* Cached balances can be introduced later without changing the domain concept.
* The architecture naturally supports transaction history.
* The model remains independent from persistence and networking.

### Negative

* `Account` cannot be fully implemented until the transaction model exists.
* Calculating a balance may require processing multiple transactions.
* Synchronization and pending-transaction semantics require additional modelling.
* A production implementation would likely need balance projections for performance.

---

## Architectural Rules

The following rules apply to the PocketBank domain:

1. **Account identity is represented by `AccountID`, not `UUID`.**

2. **Money values are represented by `Money`, not primitive numeric values.**

3. **An account has one currency within the current PocketBank domain model.**

4. **Transactions are the authoritative representation of financial activity.**

5. **Account balance is a projection of transaction activity.**

6. **A cached or persisted balance may be used for performance, but it is never more authoritative than the transaction history.**

7. **`Money` does not perform currency conversion.**

8. **`Account` does not perform currency conversion.**

9. **Networking, persistence, synchronization, and caching concerns remain outside the core domain model.**

10. **Pending transactions and available-balance semantics will be modelled explicitly rather than being implicitly mixed with settled transactions.**

11. **The domain should remain independent of any specific banking provider.**

---

## Resulting Domain Direction

The domain will evolve toward the following relationship:

```text
                    ┌──────────────┐
                    │   Account    │
                    │   AccountID  │
                    └──────┬───────┘
                           │
                           │ owns / references
                           ▼
                    ┌──────────────┐
                    │ Transactions │
                    └──────┬───────┘
                           │
                           │ composed of
                           ▼
                       ┌───────┐
                       │ Money │
                       └───┬───┘
                           │
                           │ denominated in
                           ▼
                       ┌────────┐
                       │Currency│
                       └────────┘
```

This gives us a clear domain hierarchy:

```text
Currency
   ↓
Money
   ↓
Transaction
   ↓
Account
```

while `AccountID` provides stable entity identity.

---

## Future Decisions

The following topics are intentionally deferred:

* Transaction domain model
* Transaction identity
* Credit vs debit representation
* Pending transactions
* Available balance
* Account status
* Account types
* Transaction ordering
* Reversals and refunds
* Balance caching
* Persistence strategy
* Synchronization strategy
* Currency conversion
* Multi-currency accounts
* Event sourcing

These will be addressed by subsequent ADRs or domain decisions when they become relevant.

---

## References

* ADR-0001: Layered Feature Architecture
* ADR-0002: Use Independent Swift Package Repositories for Shared Foundations
* ADR-0003: Provider Separation
* ADR-0004: Domain Modelling Rules
* `Currency` value object
* `Money` value object
* `AccountID` value object
