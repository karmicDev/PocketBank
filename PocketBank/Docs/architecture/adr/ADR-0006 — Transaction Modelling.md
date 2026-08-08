# ADR-0006: Transaction Modelling

* **Status:** Accepted
* **Date:** 2026-08-08
* **Decision:** Model transactions as immutable financial records containing a signed `Money` amount, explicit identity, account reference, timestamp, and transaction status.

---

## Context

PocketBank needs a domain representation of financial activity associated with an account.

Transactions are particularly important because, according to ADR-0005, they are the **source of truth for account balance**.

A transaction therefore needs to answer questions such as:

* Which account does this transaction belong to?
* How much money moved?
* In which currency?
* Did money enter or leave the account?
* When did the transaction occur?
* Has the transaction settled?
* How can the transaction be uniquely identified?

There are several possible ways to model the direction of a transaction.

One option is to represent the amount as a signed `Money` value:

```swift
Money(amount: Decimal("100"), currency: .eur)
```

for money entering an account, and:

```swift
Money(amount: Decimal("-100"), currency: .eur)
```

for money leaving an account.

Another option is to keep the amount always positive and represent direction separately:

```swift
Transaction(
    amount: Money(amount: 100, currency: .eur),
    direction: .debit
)
```

The decision affects the simplicity of the domain model, balance calculation, readability, validation, and future financial operations.

---

## Decision

Transactions will be modelled as **immutable domain records**.

Each transaction will have:

* a unique `TransactionID`
* an `AccountID`
* a signed `Money` amount
* a timestamp
* an explicit transaction status

Conceptually:

```swift
public struct Transaction: Sendable, Hashable, Codable {
    public let id: TransactionID
    public let accountID: AccountID
    public let amount: Money
    public let timestamp: Date
    public let status: TransactionStatus
}
```

The exact API may evolve as additional requirements are introduced, but these concepts establish the initial transaction contract.

---

# Amount Representation

## Decision: Signed Money

Transaction amounts will use the existing `Money` value object.

The sign of the amount represents the financial direction relative to the account.

### Positive amount

A positive amount represents money entering the account:

```text
+100 EUR
```

### Negative amount

A negative amount represents money leaving the account:

```text
-25 EUR
```

Therefore:

```text
Transactions

+100 EUR
 -25 EUR
 -10 EUR
─────────
 +65 EUR
```

The resulting account balance can be calculated directly from the transaction amounts.

Conceptually:

```swift
transactions
    .map(\.amount)
    .reduce(...)
```

This keeps the balance calculation simple and directly aligned with the domain representation.

---

# Why Not Model Debit and Credit Separately?

An alternative would be:

```swift
enum TransactionDirection {
    case credit
    case debit
}
```

and:

```swift
Transaction(
    amount: Money(amount: 100, currency: .eur),
    direction: .debit
)
```

This has some advantages because the direction is explicit.

However, it also means that every operation involving transaction amounts must interpret two separate properties:

```text
amount
direction
```

For example:

```text
amount = 100 EUR
direction = debit
```

must be converted into:

```text
-100 EUR
```

before calculating a balance.

With signed `Money`, the financial effect is already represented by the value:

```text
-100 EUR
```

The representation is therefore directly composable.

---

# Domain Rule: Transaction Amount Must Not Be Zero

A transaction represents a financial movement.

Therefore:

```text
0 EUR
```

does not represent a meaningful transaction.

A zero amount transaction will not be permitted.

This differs from `Money`, where zero remains valid:

```swift
Money(
    amount: 0,
    currency: .eur
)
```

is valid.

However:

```text
Transaction
amount = 0 EUR
```

is invalid because no financial movement occurred.

This distinction belongs to the `Transaction` domain rather than to `Money`.

---

# Currency

A transaction contains a `Money` value and therefore always has a currency.

The transaction currency must match the currency of the account it belongs to.

For example:

```text
Account
Currency: EUR

Transaction
Amount: +100 EUR
```

is valid.

But:

```text
Account
Currency: EUR

Transaction
Amount: +100 USD
```

is invalid within the current PocketBank domain model.

Currency conversion is explicitly outside the responsibility of `Transaction`.

As established by the `Money` design, conversion will eventually be handled by a dedicated capability such as:

```swift
CurrencyConverter
```

---

# Transaction Identity

Every transaction will have a dedicated `TransactionID`.

A raw `UUID` will not be used throughout the domain.

Conceptually:

```swift
public struct TransactionID: Sendable, Hashable, Codable {
    public let value: UUID
}
```

This follows the same domain-modelling principle established by `AccountID`.

A transaction is an entity-like domain object with stable identity.

Two transactions with the same financial amount are still different transactions:

```text
Transaction A
+100 EUR

Transaction B
+100 EUR
```

These represent two separate financial events.

Their identities must therefore remain distinct.

---

# Account Association

Every transaction belongs to an account.

The transaction will reference the account through:

```swift
AccountID
```

rather than embedding an entire `Account`.

Conceptually:

```swift
Transaction(
    id: transactionID,
    accountID: accountID,
    amount: money,
    timestamp: timestamp,
    status: status
)
```

This prevents the transaction from creating a recursive object graph:

```text
Account
 └── Transaction
      └── Account
           └── Transaction
```

Instead:

```text
Account
   │
   └── AccountID
          ▲
          │
     Transaction
```

The relationship is expressed through identity.

---

# Timestamp

Every transaction will have a timestamp.

The timestamp represents when the transaction occurred or was recorded by the domain.

The initial model will use:

```swift
Date
```

rather than introducing a custom date abstraction.

The domain does not yet need to distinguish between:

* initiated at
* authorized at
* booked at
* settled at

Those distinctions can be introduced later if the provider and product requirements require them.

---

# Transaction Status

Transactions may exist in different lifecycle states.

The initial model will therefore introduce a transaction status.

Conceptually:

```swift
public enum TransactionStatus: Sendable, Hashable, Codable {
    case pending
    case completed
    case failed
}
```

The exact status set may evolve.

The important distinction is that a transaction's lifecycle is separate from its financial amount.

For example:

```text
Transaction
Amount: -100 EUR
Status: pending
```

is different from:

```text
Transaction
Amount: -100 EUR
Status: completed
```

---

# Pending Transactions

Pending transactions require particular care because they must not automatically be treated as settled financial activity.

For example:

```text
Current balance:      1,000 EUR

Pending transaction:   -100 EUR

Available balance:       900 EUR
```

This ADR intentionally does not define the final semantics of:

* current balance
* available balance
* reserved balance
* pending balance

Those concepts will be addressed when account balance and transaction lifecycle behavior are implemented.

The important rule is:

> Transaction status must be represented explicitly rather than inferred from the amount.

---

# Balance Calculation

According to ADR-0005, transactions are the source of truth for account balance.

For settled transactions, the signed amounts can be summed directly.

Example:

```text
+1,000 EUR
  -50 EUR
  -25 EUR
──────────
  +925 EUR
```

Conceptually:

```swift
balance = transactions
    .filter { $0.status == .completed }
    .map(\.amount)
    .reduce(...)
```

The exact implementation will be introduced with the `Account` balance behavior.

Pending and failed transactions will not automatically contribute to the settled balance.

---

# Immutability

Transactions will be immutable.

Once created:

```swift
transaction.amount
transaction.accountID
transaction.timestamp
transaction.status
```

will not be directly mutated.

A transaction represents a historical financial record.

If its state needs to change, such as moving from:

```text
pending
```

to:

```text
completed
```

the domain should model that transition explicitly rather than allowing arbitrary property mutation.

This prevents accidental modification of financial history.

---

# Transaction Corrections, Reversals, and Refunds

This ADR does not introduce mutable transaction correction.

A completed transaction should not simply be edited from:

```text
-100 EUR
```

to:

```text
-50 EUR
```

because doing so destroys historical information.

Future transaction correction mechanisms should instead model concepts such as:

* reversal
* refund
* compensating transaction
* adjustment

These will be represented as new financial activity.

For example:

```text
Original transaction:
-100 EUR

Refund:
+100 EUR
```

rather than modifying the original transaction.

---

# Transaction Ordering

Transactions will contain a timestamp, but timestamp alone is not considered a globally unique ordering mechanism.

Two transactions may have the same timestamp.

The transaction ID therefore remains the identity mechanism.

If deterministic ordering becomes necessary, a later domain or persistence decision may introduce additional sequencing information.

This is intentionally deferred.

---

# Provider Independence

The core transaction model must not depend on any banking provider.

Provider-specific transaction representations such as:

```text
Open Banking transaction
Plaid transaction
Bank API transaction
```

must be mapped into the PocketBank domain model.

Conceptually:

```text
Provider API
     │
     ▼
Provider DTO
     │
     ▼
Transaction
     │
     ▼
PocketBank Domain
```

The `Transaction` domain object must not contain:

* provider-specific IDs as its primary identity
* API response objects
* networking types
* JSON-specific implementation details
* provider-specific status values

Provider mapping belongs outside `PocketBankCore`.

---

# Alternatives Considered

## Option A — Signed Money

```text
+100 EUR
-25 EUR
```

### Advantages

* Simple balance calculation
* Directly represents financial effect
* Uses existing `Money` abstraction
* No separate direction enum
* Easy arithmetic
* Minimal domain model

### Disadvantages

* Sign must be interpreted correctly
* The meaning of positive and negative values is contextual
* Some provider APIs represent direction separately

**Accepted.**

---

## Option B — Positive Money + TransactionDirection

```swift
Transaction(
    amount: Money(amount: 100, currency: .eur),
    direction: .debit
)
```

### Advantages

* Direction is explicit
* Maps naturally to providers that expose debit/credit separately
* Prevents negative `Money` from representing financial movement

### Disadvantages

* Requires additional domain concept
* Balance calculation requires interpreting direction
* Creates two pieces of state representing one financial effect
* More complex arithmetic
* Duplicates information that can already be represented by the sign

**Rejected for the initial domain model.**

---

## Option C — Separate Credit and Debit Transaction Types

For example:

```swift
CreditTransaction
DebitTransaction
```

### Advantages

* Very explicit type-level modelling
* Impossible to confuse direction at the type level

### Disadvantages

* Creates unnecessary type proliferation
* Makes generic transaction handling more complicated
* Complicates APIs and persistence
* Does not provide enough value for PocketBank's current scope

**Rejected.**

---

## Option D — Event Sourcing

Represent every transaction as a domain event and reconstruct account state by replaying events.

### Advantages

* Strong auditability
* Complete history
* State can be reconstructed
* Powerful for complex financial systems

### Disadvantages

* Significant architectural complexity
* Requires event storage
* Beyond PocketBank's current scope
* Would introduce infrastructure concerns too early

**Rejected for now.**

This decision does not prevent a future event-sourced architecture.

---

# Consequences

## Positive

* Transactions have a clear and minimal domain representation.
* Financial direction is directly represented by the signed amount.
* Balance calculation becomes straightforward.
* Transaction history remains immutable.
* Transaction identity is strongly typed.
* Account relationships use domain identifiers.
* Currency remains explicit.
* Provider-specific representations remain outside the domain.
* The model is suitable for concurrency and Codable persistence.
* Future reversal and refund modelling can preserve financial history.

## Negative

* Consumers must correctly interpret the sign of transaction amounts.
* Some provider APIs will require mapping debit/credit semantics into signed amounts.
* Pending transaction semantics require additional domain decisions.
* Transaction status adds lifecycle complexity.
* Balance calculation may eventually require projections or caching for performance.

---

# Architectural Rules

The following rules apply to the PocketBank transaction model:

1. **Every transaction has a unique `TransactionID`.**

2. **Every transaction belongs to an `AccountID`.**

3. **Transaction amounts are represented using `Money`.**

4. **Positive transaction amounts represent money entering the account.**

5. **Negative transaction amounts represent money leaving the account.**

6. **A transaction amount must not be zero.**

7. **A transaction's currency must match the account currency.**

8. **Transactions are immutable financial records.**

9. **Completed transactions must not be modified to correct financial history.**

10. **Corrections, refunds, and reversals are represented as new financial activity.**

11. **Transaction status is independent of transaction amount.**

12. **Pending and failed transactions do not automatically contribute to the settled account balance.**

13. **Transactions are the source of truth for account financial activity.**

14. **Account balance is derived from applicable transactions.**

15. **Transaction domain objects must not depend on banking providers.**

16. **Provider-specific transaction data must be mapped into the PocketBank domain model outside `PocketBankCore`.**

17. **Currency conversion is outside the responsibility of `Transaction`.**

---

# Resulting Domain Model

The domain now has the following conceptual structure:

```text
                         ┌──────────────┐
                         │   Account    │
                         │              │
                         │  AccountID   │
                         │  Currency    │
                         └──────┬───────┘
                                │
                                │ AccountID
                                ▼
                       ┌─────────────────┐
                       │   Transaction   │
                       │                 │
                       │ TransactionID   │
                       │ AccountID       │
                       │ Money           │
                       │ Timestamp       │
                       │ Status          │
                       └────────┬────────┘
                                │
                                │ contains
                                ▼
                           ┌──────────┐
                           │  Money   │
                           │          │
                           │ Decimal  │
                           │ Currency │
                           └────┬─────┘
                                │
                                ▼
                           ┌──────────┐
                           │ Currency │
                           └──────────┘
```

The resulting financial flow is:

```text
Transaction
     │
     │ signed Money
     ▼
Account balance
```

For example:

```text
Account: EUR

Transactions:

+1,500 EUR  completed
  -50 EUR   completed
  -25 EUR   completed
 -100 EUR   pending
           ──────────
Balance:    1,425 EUR
```

The pending transaction may affect an eventual available-balance calculation, but it does not change the settled balance defined by this ADR.

---

# Future Decisions

The following topics are intentionally deferred:

* Transaction creation rules
* Transaction validation API
* Account balance calculation API
* Available balance
* Pending balance
* Transaction status transitions
* Transaction reversal
* Refunds
* Recurring transactions
* Transfer modelling
* Transaction metadata
* Merchant information
* Transaction categories
* Provider transaction IDs
* Transaction synchronization
* Pagination
* Transaction ordering
* Balance caching
* Persistence strategy
* Event sourcing

These decisions will be made when the corresponding domain requirements arise.

---

# References

* ADR-0001: Layered Feature Architecture
* ADR-0002: Use Independent Swift Package Repositories for Shared Foundations
* ADR-0003: Provider Separation
* ADR-0004: Domain Modelling Rules
* ADR-0005: Account and Balance Modelling
* `Currency` value object
* `Money` value object
* `AccountID` value object
* `Account` entity

