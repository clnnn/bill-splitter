# Bill Splitter

People who share costs record what each of them paid, and see what each of them owes or is owed overall.

## Language

### People and groups

**User**:
A person with an account, identified by their email address.
_Avoid_: Account, person

**Display name**:
The name a user gives for themselves, shown to the other members of every group they are in. Their email address is never shown to other members.
_Avoid_: Username, nickname, full name

**Sign-in link**:
A link emailed to an address that, when the person holding the mailbox confirms it, signs them in as the user with that address, creating the user if there is none. It works once and for a short time.
_Avoid_: Magic link, login link, verification link

**Session**:
One browser's signed-in state for one user, begun by a sign-in link and ended by signing out or by a long time unused.
_Avoid_: Login, token

**Group**:
A set of members who share expenses in one currency. It has a name, one invite link and one ledger, and no owner.
_Avoid_: Team, circle, household

**Member**:
A user's membership of one group. One user in two groups is two members.
_Avoid_: Participant, friend

**Join order**:
The order in which a group's members joined it, the user who created the group first. It never changes as the group grows.
_Avoid_: Member order, seniority

**Invite link**:
The one link belonging to a group that lets any user who opens it become a member.
_Avoid_: Invitation, join link, share link

### Money

**Currency**:
The one ISO 4217 currency that all of a group's amounts are in, chosen when the group is created and never changed.
_Avoid_: Denomination, money type

**Minor unit**:
The smallest unit of a currency (a cent, a yen, a fils). Every amount, share and balance is a whole number of minor units.
_Avoid_: Cent, subunit, penny

### Ledger

**Ledger**:
The ordered list of a group's entries. Entries are only ever added, never changed or removed.
_Avoid_: History, log, activity

**Entry**:
One record in a ledger. An expense is the one kind of entry so far.
_Avoid_: Transaction, record, item

**Expense**:
An entry recording that one member paid an amount on behalf of the group, with a description and one share per member.
_Avoid_: Bill, purchase, cost

**Payer**:
The one member who paid an expense. Not necessarily the member who recorded it.
_Avoid_: Buyer, owner

**Amount**:
The total a payer paid for an expense.
_Avoid_: Total, price, cost

**Share**:
The part of an expense's amount that one member is responsible for, fixed when the expense is recorded. The payer has a share too, and an expense's shares add up to exactly its amount.
_Avoid_: Split, portion, cut

**Equal split**:
The way an expense's amount is divided into shares that differ by at most one minor unit. Every member gets the amount divided by the number of members, rounded down; the minor units left over go one each to the payer first, then to the other members in join order.
_Avoid_: Even split, fair split

**Balance**:
What a member has paid minus the total of their shares, across the whole ledger. Positive means the group owes them, negative means they owe the group, and a group's balances add up to zero.
_Avoid_: Debt, net, total
