# ADR-0007 — Account/Transaction Aggregate Boundary

* **Status:** Accepted
* **Date:** 2026-08-08
* **Deciders:** PocketBank Engineering
* **Technical Area:** Domain Model / Architecture
* **Related ADRs:** ADR-0005 — Account and Balance Modelling, ADR-0006 — Transaction Modelling

---

## 1. Context

PocketBank models financial accounts and their transactions as separate domain concepts.

An `Account` currently contains:

```swift
AccountID
Currency
```

A `Transaction` contains:

```swift
TransactionID
AccountID
Money
Date
TransactionStatus
```

A transaction therefore identifies the account it belongs to but does not contain an `Account` reference.

This separation is intentional. `Transaction` should remain independent of `Account` and should not need to know the complete state of the account to represent a financial event.

However, some business rules require knowledge of both objects.

For example:

```text
Transaction.accountID == Account.id
```

and:

```text
Transaction.amount.currency == Account.currency
```

must hold when a transaction is associated with an account.

The transaction itself cannot enforce these rules because it only contains an `AccountID`. Introducing an `Account` reference into `Transaction` would create unnecessary coupling and potentially introduce cyclic domain relationships.

We therefore need to establish a clear aggregate boundary that defines:

1. Which object owns the relationship.
2. Where transaction validation occurs.
3. Where account-level invariants are enforced.
4. How transactions are added to an account.
5. How account balance is derived.
6. How future transaction lifecycle operations are handled.

---

# 2. Decision

`Account` will act as the **aggregate root** for account-related financial state.

`Transaction` remains an independent immutable domain entity.

The relationship is therefore:

```text
                 Account
              ┌──────────────┐
              │ AccountID    │
              │ Currency     │
              │ Transactions │
              └──────┬───────┘
                     │
                     │ owns relationship
                     ▼
              ┌──────────────┐
              │ Transaction  │
              │              │
              │ TransactionID│
              │ AccountID    │
              │ Money        │
              │ Timestamp    │
              │ Status       │
              └──────────────┘
```

The aggregate boundary is:

```text
Account Aggregate
│
├── Account
│
└── Transactions
```

`Account` is responsible for accepting transactions and enforcing invariants that require knowledge of the account.

---

# 3. Responsibilities

## 3.1 Account

`Account` is responsible for:

* Identifying the account.
* Defining the account currency.
* Managing its associated transactions.
* Enforcing account/transaction relationship invariants.
* Preventing transactions from being associated with the wrong account.
* Preventing transactions with incompatible currencies.
* Providing access to the account's transaction history.
* Providing the basis for balance calculation.

---

## 3.2 Transaction

`Transaction` remains responsible for its own local invariants and data.

A transaction is responsible for:

* Its identity.
* Its associated `AccountID`.
* Its monetary amount.
* Its timestamp.
* Its status.
* Rejecting invalid transaction-local state, such as a zero amount.

A `Transaction` must not:

* Hold an `Account` reference.
* Know the account's currency.
* Calculate account balance.
* Perform currency conversion.
* Modify its own state after creation.
* Enforce invariants that require access to the containing `Account`.

---

# 4. Account/Transaction Relationship

A transaction belongs to an account through its `AccountID`.

For example:

```swift
Transaction(
    accountID: account.id,
    amount: Money(amount: 100, currency: .eur),
    status: .completed
)
```

The transaction does not receive the complete `Account`.

This keeps the dependency direction simple:

```text
Transaction
    │
    ├── AccountID
    └── Money
```

rather than:

```text
Transaction
    │
    └── Account
         │
         └── Transaction
```

The latter would introduce unnecessary bidirectional coupling.

---

# 5. Aggregate Invariants

When a transaction is added to an account, the following invariants must be enforced.

## 5.1 Account Identity

The transaction must belong to the account receiving it.

```swift
transaction.accountID == account.id
```

A transaction belonging to another account must be rejected.

---

## 5.2 Currency Compatibility

The transaction amount must use the account's currency.

```swift
transaction.amount.currency == account.currency
```

For example:

```text
Account
    currency = EUR

Transaction
    amount = 100 EUR

→ valid
```

Whereas:

```text
Account
    currency = EUR

Transaction
    amount = 100 USD

→ rejected
```

This rule applies when the transaction is associated with the account.

The transaction itself does not enforce this invariant.

---

# 6. Why Account Is the Aggregate Root

The account is the natural consistency boundary because the invariants require account-level knowledge.

The caller should not be able to bypass the rules by manipulating a transaction collection directly.

Instead of:

```swift
account.transactions.append(transaction)
```

the API should eventually expose an operation such as:

```swift
try account.record(transaction)
```

or:

```swift
try account.add(transaction)
```

The exact API will be decided during implementation.

The important architectural rule is:

> Transactions enter the Account aggregate through the aggregate root.

This ensures that all account-level invariants are enforced consistently.

---

# 7. Transaction Collection Encapsulation

The account's transaction collection must not be freely mutable by external callers.

The preferred model is:

```swift
public private(set) var transactions: [Transaction]
```

or an equivalent encapsulation strategy.

External code should be able to inspect transactions but should not be able to bypass aggregate validation.

For example, this should not be possible:

```swift
account.transactions.append(transaction)
```

if doing so would bypass currency or ownership validation.

Instead:

```swift
try account.record(transaction)
```

should be the controlled mutation boundary.

---

# 8. Transaction Immutability

Transactions remain immutable after creation.

A transaction's:

```text
ID
AccountID
Amount
Timestamp
Status
```

must not be directly mutable.

This means an account does not modify an existing transaction to change its financial meaning.

Future lifecycle operations such as:

```text
pending → completed
pending → failed
completed → reversed
```

will require explicit domain modelling.

Such transitions are outside the scope of this ADR.

---

# 9. Balance Calculation

The account balance is derived from its transactions.

Conceptually:

```text
Balance = Σ Transaction Amounts
```

For example:

```text
+100 EUR
-25 EUR
+50 EUR
─────────
 125 EUR
```

The exact treatment of transaction status will be defined separately.

In particular, this ADR does not yet decide whether:

```text
pending
completed
failed
```

transactions contribute to the balance in the same way.

That decision belongs to the balance calculation design.

---

# 10. Currency Conversion

Currency conversion is explicitly **not part of the Account aggregate boundary**.

If an external transaction is received in another currency, the system must not silently convert it simply to satisfy the account currency invariant.

For example:

```text
Account:       EUR
Transaction:   USD
```

must not automatically become:

```text
Transaction:   EUR
```

by invoking a converter during transaction insertion.

Currency conversion may become necessary for future functionality, but it represents a separate domain concern.

A future ADR will define:

* Exchange rates.
* Conversion providers.
* Conversion timing.
* Rounding.
* Historical rates.
* Original vs converted amounts.
* Provider-specific currency information.

---

# 11. Provider Independence

The aggregate must remain independent of external banking providers.

The following concepts must not appear inside the aggregate:

```text
PlaidTransaction
OpenBankingTransaction
ProviderTransactionID
BankAPIResponse
ProviderCurrency
```

Provider-specific models belong outside `PocketBankCore` and must be mapped into the domain model.

Conceptually:

```text
Provider
   │
   ▼
Provider Adapter
   │
   ▼
Domain Transaction
   │
   ▼
Account Aggregate
```

This preserves the independence of the core domain.

---

# 12. Persistence Boundary

This ADR does not prescribe a persistence technology.

The aggregate must not depend directly on:

```text
Core Data
SwiftData
Realm
SQLite
REST
GraphQL
Firebase
```

Persistence mechanisms belong outside the domain layer.

A future repository abstraction may provide persistence:

```swift
protocol AccountRepository {
    func load(id: AccountID) async throws -> Account
    func save(_ account: Account) async throws
}
```

The exact repository architecture will be addressed separately.

---

# 13. Concurrency and Sendability

`Account` and `Transaction` remain `Sendable` domain models.

The aggregate should be manipulated through controlled operations rather than exposing mutable state across concurrency boundaries.

Concurrency coordination, persistence synchronization, and actor isolation will be addressed at the application/infrastructure layer unless a concrete domain requirement emerges.

The domain model itself should remain free of unnecessary concurrency infrastructure.

---

# 14. Alternatives Considered

## 14.1 Transaction Owns an Account

Example:

```swift
struct Transaction {
    let account: Account
}
```

### Rejected

This creates bidirectional coupling:

```text
Account → Transaction
Transaction → Account
```

It also duplicates account information and makes the transaction dependent on the entire account aggregate.

A transaction only needs the account identity.

---

## 14.2 Transaction Stores Account Currency

Example:

```swift
struct Transaction {
    let accountID: AccountID
    let accountCurrency: Currency
    let amount: Money
}
```

### Rejected

This duplicates information already owned by `Account`.

It creates multiple sources of truth:

```text
Account.currency
Transaction.accountCurrency
```

These values could become inconsistent.

---

## 14.3 Universal `BankingID`

Example:

```swift
protocol BankingID {
    var value: UUID { get }
}
```

### Rejected

Strongly typed domain IDs provide important compiler-level guarantees.

These should remain distinct:

```text
AccountID
TransactionID
CustomerID
TransferID
```

Even if they share the same underlying representation.

Implementation reuse may be introduced later if justified, but public domain semantics should remain strongly typed.

---

## 14.4 Automatic Currency Conversion

Example:

```text
Account EUR
Transaction USD
       │
       ▼
CurrencyConverter
       │
       ▼
Transaction EUR
```

### Rejected

Currency conversion is not the same concern as currency validation.

Automatic conversion could also destroy information about the original transaction currency and amount.

Currency conversion will be addressed independently if required.

---

## 14.5 Free-Mutable Transaction Collection

Example:

```swift
account.transactions.append(transaction)
```

### Rejected

This allows callers to bypass aggregate invariants.

All mutations affecting the aggregate must pass through the aggregate root.

---

## 14.6 Transaction Repository as Aggregate Boundary

An alternative would be to treat transactions independently and let a repository enforce account relationships.

### Rejected for the initial domain model

The relationship between an account and its transactions is part of the account's domain consistency.

The aggregate root provides a more explicit and testable boundary.

Persistence and repository concerns remain separate.

---

# 15. Consequences

## Positive

### Strong invariant enforcement

All account-level transaction rules have one authoritative entry point.

### No duplicated account state

Transactions don't store account currency or the complete account.

### Strong type safety

`AccountID` and `TransactionID` remain distinct domain types.

### Clear dependency direction

```text
Account
   │
   └── Transaction

Transaction
   ├── AccountID
   └── Money
```

### Provider independence

The domain remains independent of banking providers.

### Testability

Account-level rules can be tested without networking, persistence, or external services.

### Future extensibility

The model leaves room for:

* Currency conversion
* Provider adapters
* Persistence
* Transaction lifecycle
* Balance policies
* Multiple transaction sources

without prematurely coupling those concerns to the domain.

---

## Negative

### Account becomes more complex

The aggregate root now owns transaction management and validation.

### Transaction access is indirect

Callers cannot simply mutate a transaction collection.

### Future transaction lifecycle operations require additional modelling

Changing transaction status will require explicit domain decisions.

### Large transaction histories may require additional architecture

An in-memory transaction collection may eventually become unsuitable for very large accounts.

Persistence and loading strategies will need to address this later.

---

# 16. Testing Strategy

The following behaviors must eventually be covered.

## Valid Transaction

```text
Account EUR
Transaction EUR
→ accepted
```

## Wrong Account

```text
Account A
Transaction belongs to Account B
→ rejected
```

## Currency Mismatch

```text
Account EUR
Transaction USD
→ rejected
```

## Zero Amount

```text
Transaction 0 EUR
→ rejected
```

## Multiple Transactions

```text
Account
    +100 EUR
    -20 EUR
    +50 EUR

→ all valid transactions retained
```

## Encapsulation

Verify that callers cannot bypass aggregate validation by directly mutating the transaction collection.

## Balance

Balance behavior will be tested separately once the balance policy has been defined.

---

# 17. Implementation Guidance

The implementation should introduce a controlled operation on `Account`, for example:

```swift
public mutating func record(
    _ transaction: Transaction
) throws
```

The exact method name is intentionally left open.

Conceptually:

```swift
public mutating func record(
    _ transaction: Transaction
) throws {
    guard transaction.accountID == id else {
        throw AccountError.transactionBelongsToDifferentAccount
    }

    guard transaction.amount.currency == currency else {
        throw AccountError.currencyMismatch
    }

    transactions.append(transaction)
}
```

The implementation should preserve the following invariant:

```text
Every transaction contained by an Account:
    transaction.accountID == account.id
    transaction.amount.currency == account.currency
```

The exact `AccountError` cases and collection representation will be determined during implementation.

---

# 18. Architectural Rules

The following rules are established by this ADR:

1. `Account` is the aggregate root for account transactions.
2. `Transaction` is an independent immutable domain entity.
3. `Transaction` references an account through `AccountID`, not `Account`.
4. Transactions enter an account through controlled aggregate operations.
5. Account-level invariants must not be bypassable through public mutable collections.
6. Account identity and transaction identity remain strongly typed and distinct.
7. Account/transaction currency compatibility is enforced when the relationship is established.
8. Currency conversion is not performed automatically by the aggregate.
9. Provider-specific concepts do not enter the core domain.
10. Persistence technology does not belong inside the aggregate.
11. Transaction lifecycle transitions require explicit domain modelling.
12. Balance calculation is derived from transactions but requires a separate policy decision.

---

# 19. Future Work

This ADR intentionally leaves several topics for future milestones:

* Transaction lifecycle/state transitions
* Balance calculation policy
* Pending transaction handling
* Reversals
* Refunds
* Currency conversion
* Foreign-currency transactions
* Transaction pagination
* Persistence
* Repository abstractions
* Provider adapters
* Open Banking integration
* Plaid integration
* Account synchronization
* Transaction deduplication
* Transaction ordering

These concerns should be introduced only when their domain requirements are understood.

---

# 20. Decision Summary

PocketBank will model `Account` as the aggregate root and `Transaction` as an immutable entity associated with the account through `AccountID`.

The account is responsible for enforcing invariants that require knowledge of both the account and transaction.

Therefore:

```text
Transaction
    knows → AccountID

Account
    knows → Transactions

Account
    enforces → Account/Transaction invariants
```

The critical invariant is:

```text
transaction.accountID == account.id
AND
transaction.amount.currency == account.currency
```

This design avoids bidirectional coupling, duplicated account state, premature currency conversion, and provider-specific dependencies while providing a clear consistency boundary for future financial functionality.

**Decision: Accepted.**

