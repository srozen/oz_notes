# 03/05/17

# Summary

## Shared-State concurrency (chap8)
- Mutable state : cells
- Exchange operation + semantics
- Kernel language
- Why is it difficult ?
- Managing interleaving ?
- Principle : build large atomics actions
- Programming with exchange
- Locks, monitors, transactions

# Mutable state : cells

A cell is a variable which can be updated

```
C = {NewCell 5}

C := 55

C := @C + 1

% 3 operations

```

```
% Semantics

% Access and assignements

single-assignment store
{
  c = name of the cell (can be considered as the adress of the cell)
  x=5
}

multiple assignments stores (contain cell boxes)
{
  c:x (pair of cell name + content)
}

if we do C:= 55, we change the content

single-assignment store
{
  c = name of the cell (can be considered as the adress of the cell)
  x=5
  y=55
}

multiple assignments stores (contain cell boxes)
{
  c:y (pair of cell name + content)
}

if we do Z = @C

single-assignment store
{
  c = name of the cell (can be considered as the adress of the cell)
  x=5
  y=55
  z=y
}

multiple assignments stores (contain cell boxes)
{
  c:y (pair of cell name + content)
}
```  

Access and assignements are hard to use in concurrency

## Exchange

```oz
Thread 1
C:=@C+1
-> A=@C
-> B=A+1
-> C:=B

Thread 2
C:=@C+1
-> A'=@C
-> B'=A'+1
-> C:=B'

```   

Problem with interleaving, C could be incremented only once !

**Exchange**

```oz
{Exchange C A B}

% Before
c=(name)
x
a
b

c:x

% After
c=(name)
x
a=x
b

c:b

% A is the old value
% B is the new value of the cell

```

### How the increment happens with exchange

```oz
Thread 1
{Exchange C A B}
-> B=A+1

Thread 2
{Exchange C A' B'}
-> B'=A'+1

% No problem; 2 increments independants of scheduler choices

```
All processors have an operation similar to Exchange
"Compare and swap"
"Test and set" -> test the value and change it

## Why is shared state difficult?

```

:=,@,Exchange

T1 -----a1-----a2----a3-------ak>

T2 -----b1-------b2--------bk--->

Actual execution ---a1----a2--b1--a3--a4--ak--bk--->
The scheduler chooses

How many interleavings possibles?  

(2k)
(k )

= (2k)!/k!k! = 2^2k/sqrt(pi k) -> Exponential number

```

In order to master the interleaving, we want to
-> reduce the number
-> atomic operations -> Interleaving of a small number of atomic actions

for example b..b4 will be one action, same for a..a10

-> invariants : atomic actions maintain truth



```oz
% Cells
declare
C = {NewCell 5}

{Browse @C}

% Concurrent counter
% COunter has increment and decrement operations
declare
proc {Inc} New Old in
  {Exchange C Old New}
  New = Old + 1
end

declare
proc {Dec} New Old in
  {Exchange C Old New}
  New = Old-1
end

{Inc}
{Browse @C}

thread {Inc} {Dec} end
thread {Dec} {Inc} end

for I in 1..1000 do
  thread {Delay 1} {Inc} end
  thread {Delay 1}
    if I mod 100 == 0 then {Browse @C} end
  thread {Delay 2}{Dec} end
end

% Simulating a slow network
declare
fun{SlowNet Obj D}
  proc{$ M}
    thread
      {Delay D}
      {Obj M}
    end
  end
end

declare
proc {Obj M}
  {Browse M}
end

declare
SlowObj = {SlowNet Obj 1000}

{SlowObj foo}

% Call SlowObj twice in the same thread
{SlowObj a1}
{SlowObj a2}
% a1 should appear before a2 but this is not guaranteed

% How can I guarantee the order?
% Technique called "token passing"
% Uses a cell + Exchange
```

Each call of SlowObj (knows Xi and Xi+1)
- waits for the token -> wait until Xi is bound
- gets the token
- executes
- passes the token to the next celle

```oz
declare
fun {SlowNet2 Obj D}
  C={NewCell ok}
in
  proc {$ M} Curr Next in
    % Get the variable + the next variable
    {Exchange C Curr Next}
    thread
      {Delay D}
      {Wait Curr} % Wait for token
      {Obj M}
      Next=ok % Pass token to next one
    end
  end
end

declare
SlowObj2 = {SlowNet2 Obj 1000}

{SlowObj2 a1}
{SlowObj2 a2}
```

## Locks

variables c1,c2,...,c100
shared ressources (database)

many threads want to access this ressources
```oz
thread1 lock L then
atomic actions
ens
```

The scheduler is forced to execute all the atomic actions
**critical region**
Only one thread can be executing inside the region protected by the Lock L
Effects : regions are atomic
We can implement Locks using Exchange

Lock abstraction

L = {NewLock}
lock L then \<s\> end

Without new syntax :

L = {NewLock}
{L proc {$} \<s\> end }
Using higher-order programming

```oz
% Locks

% First version of a lock

proc {NewLock L}
  Token = {NewCell ok}
in
  proc {L P}
    {Exchange Token Old New}
    {P}
    New=ok
  end
end
```

Build abstraction

Example : concurrent queue

```oz
proc {Insert Q X}
  lock L then
  end
end

proc {Delete Q X}
  lock L then
  end
end
```

Insert 2 elements together ?
**Desirable code**   

```oz
proc{Insert 2 Q X1 X2}
  lock L then
    {Insert Q X1}
    {Insert Q X2}
  end
end
```

Simple Lock is not good enough, not working

```oz
% Reentrant lock
% Smarter than SImple lock: same thread can go in again
% To implement this, thread need *identities* (names)

proc {NewReentrantLock L}
  Token = {NewCell ok}
  CurThr={NewCell none} % Identitiy of current thread
in
  proc {L P}
    if @CurThr == {Thread.this} then
      {P}
    else
      {Exchange Token Old New}
      {Wait Old}
      CurThr := {Thread.this} % CurThr assignment inside ! Otherwise buggy
      try
        {P}
      finally
        CurThr := none
        New=ok
      end
    end
  end
end
```
