---
status: accepted
---

# Append-only ledger of typed entries

A group's ledger is a list of entries that are only ever added, never changed or removed. Every entry has a common part (its group, its kind, one currency, who recorded it and when) plus the facts of its own kind. An expense, the only kind so far, stores its payer, amount, description and one share per member. Shares are computed once, when the expense is recorded, and stored; they are never recomputed. A member's balance is not stored anywhere: it is derived from the ledger on every read, by a rule per kind of entry.

This keeps exactly one copy of each fact, so what a group sees for an expense and what its balances are built from cannot disagree, and a member who joins later cannot change what earlier expenses meant.

## Considered Options

- **Uniform signed lines per member for every kind of entry** (Ana +45.00, Ben -45.00), with balance as a plain sum. Rejected: an expense still has to keep its payer, amount and shares to be shown, so the lines would be a second copy that can drift from them. The price of typed entries is one more balance rule per new kind.
- **Recomputing shares from current membership on read.** Rejected: a member joining would silently rewrite every earlier expense and every balance.
- **A stored running balance per member.** Rejected: a second source of truth to keep in step with the ledger, for a sum that is cheap at this scale.
- **Editing or deleting entries in place.** Rejected: balances would change with no trace of why. Expenses are add-only in the first slice, and a later correction is a new entry that refers to the one it corrects, leaving the original in place.

## Consequences

- A settlement arrives as a second kind of entry with its own facts and its own balance rule; expenses do not change.
- Edit and delete, when a slice asks for them, are built as a correction kind of entry, not as updates. Nothing is built for corrections now: no voided flag, no reference field.
- Each entry's currency must equal its group's currency for now, and a balance is one figure in that currency. Allowing entries in other currencies means redefining balance, which is a decision for that slice.
- If ledgers grow large enough that deriving balances on read is slow, a cached total can be added as a derived copy without changing the ledger.
