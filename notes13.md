# Summary

Transactions
- Motivation and ACID properties
- Properties
  - Serializability
  - Avoiding cascading abort
  - Deadlock and wait-for
- Concurrency control
" Optimistic concurrency control with strict two-phase locking and deadlock avoidance"


# Transactions

Database, simple model
Array of cells
These cells are all the bank accounts
We want to update these accounts with transactions, how to work
efficiently with this lot of transactions?

1. By far the simplest way is to have one lock for the database, but
one lock is very inefficient. One lock for a huge database is not
scalable.

2. We have to be smarter. We put one lock for each piece of data.
Each lock protect its piece of data.
Database have to be stored on stable storage, in case of crash.
But we update that disk, so the transactions has to be atomic.
Atomic udate on disk is implemented using switch (before / after), and a buffer.

Properties of transactions ACID :
- Atomic : intermediate states are invisibles,
- Consistent : states have invariant properties (bank : total amount of money is constant)
- Isolation : Several concurrent transaction don't interfere
- Durability : peristance of the data, stable sorage


We want to implement ACID.

## Concurreny Control
Algorithm that implements ACID.

### Atomicity

#### Serializability
We have two transactions

```
T1 -------|c1-(100-20)|------|c2|-----> time
T2 ---------|c1-(80-20)|--|c2|-------> time (FASTER !)

Transaction from c1 to c2, after the transaction is finished
the money is transfered.
But we can have problem of transaction, what if T2 wants to transfer also
and he's faster than T1
When transfering T2 sees the intermediate state (not atomic !)


T1 has to keep the locks of c1 and c2 until it's transaction is finished
T1 -------|c1         c2|-----> time
T2 -------------------------|c1         c2|-------> time

```

Serializability
- All transaction executes "as if" they were sequential, in some order
- This guarantees atomicity

Two-phase locking
- Atomic
- "T1 and T2 execute in a sequential order"
- Most used, simple way to guarantee atomic transaction

#### Cascading abort
Shrinking phase : T1 progressively release locks not needed anymore,
and T2 can use them already.

What if T1 crash and abort, as it never happened but T2 used changed
data ?
T2 is already done and commited ! T2 has to abort too because it
used the new values of T1.
We have to do a cascading abort, necessary for correctness !
Complicated implementation
Interaction with external world harder

We want to avoid cascading abort
**Strict 2 Phase Locking**
Still a growing phase, but the locks are all released at once !!
The cascading abort is then avoided, no shrinking phase.

#### Deadlock

```
T1 -------|c1         c2
T2 -------|c1         c2
This is a deadlock
```   

T1 and T2 locks in the same time
Anywhere we have users and ressources, general problem in society


#### Wait-for graph

- Transactions
- Locks (ressources)

```
    T1
  /     \
C1      C2
  \     /
    T2

Deadlock :
Cycle in wait-for graph
- Large system
- 100's active transaction
- 1000's of locks

```

We have to lock at the whole system to detect a deadlock
We prefer avoid than cure, **avoidance** required !


## COncurrency control

* Transaction requests lock
- asks transaction manager
- Yes; no; please wait; restart(all locks released)
* Optimistic/pessimistic
- T asks for a lock
Pessimistic won't give the lock if any risk of deadlock (tends to say no because fixing might be hard,expensive)
example : trains on train tracks
Optimistic would tend to give the lock (fixing the deadlock is easy)
example : airplane seats reservation

## Priority

- Each transaction has priority number
- Early transaction have higher priority

Transaction asks for a lock

- if cell unlocked, give it
- if already locked by same transaction, continue
- if cell is locked by transaction by higher priority : wait
- if celle locked by transaction with lower priority : restart lower priority, give the lock


- commits: releases all locks
- aborts : releases + restart state

LJAPO1100,LINGI2172,LFSA2202,LINGI2399,LINGI1123,LINGI1122,LINGI1131,LSMG2004
