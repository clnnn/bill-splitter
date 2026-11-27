---
status: accepted
---

# Money is integer minor units with an ISO 4217 code on every amount

Every amount, share and balance is a whole number of minor units of one ISO 4217 currency, and every stored amount carries its currency code. A group's currency is chosen when the group is created and never changes; each expense copies that code next to its amount, and the domain refuses an expense whose currency differs from its group's. How many minor units make one major unit (0 for JPY, 2 for EUR, 3 for KWD) comes from a table of supported currencies checked into the domain library.

Whole minor units make an equal split exact: shares add up to the amount with nothing lost to rounding. The code on every amount keeps the ledger self-describing, so entries recorded today need no migration when a later slice allows a second currency.

## Considered Options

- **Decimal or floating-point amounts in major units.** Rejected: shares would need rounding rules at every sum, and floating point cannot represent most decimal fractions exactly.
- **Currency stored on the group only.** Rejected: it is enough today, but an entry would then need its group to say what it meant, and a second currency would force a migration of every stored amount.
- **One app-wide currency, or two-decimal currencies only.** Rejected: both build in an assumption that is expensive to remove, to save work that is confined to parsing input and formatting display.
- **Taking exponents from the runtime (`Intl`) instead of a checked-in table.** Rejected: the exponent decides what a stored integer means, so it must not shift with a Node upgrade. A test compares the table with `Intl` to catch drift.
- **Letting the currency change while the ledger is empty.** Rejected: an extra screen and rule to spare someone recreating a group they have just made.

## Consequences

- An amount typed with more decimals than its currency has is refused, never rounded.
- An amount is between 1 and 100,000,000,000 minor units. The cap keeps every balance well inside JavaScript's safe integer range, so the domain uses plain numbers; storage needs a 64-bit integer.
- A refund or any other negative movement is a new kind of entry, not a negative expense.
- If ISO 4217 changes a currency's exponent, existing amounts in that currency need a deliberate migration; the table does not silently follow.
