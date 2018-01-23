# 08/03/17

## Summary

### Lazy deterministic dataflox
- By-need synchronization : WaitNeeded
- Lazy functions and lazy suspensions
- Simple examples : infinite lists, producer-consumer
- Hamming problem
- Three kernel languages
- What is declarative concurrency?
- Bounded buffer
- Lazy quicksort

##

```
{WaitNeeded X} waits until a thread does {Wait X}

{Wait X} waits until a thread binds X, implicit in many operations

These allow to change dataflow
```

Example

```
declare
fun lazy{Ints N}
  N|{Ints N+1}
end

% Creates an inifinite list from N

declare
L = {Ints 0}
{Browse L}
% This calls doesn't do anything, browse doesn't need the result
% How to need L, what if we want to display the first element?

{Browse L.1} % Implicit does a wait on first argument, triggering the binding
{Browse L.2.2.2.1}

% In Kernel language, every function is a procedure with one more argument which will be the output
% Ints translates to WaitNeeded :

% This one would create an infinite loop because there is no wait needed
declare
proc{Ints2 N R}
  R=N|{Ints2 N+1}
end

% The WaitNeeded would be blocking, we must add a thread
declare
proc{Ints2 N R}
  {WaitNeeded R} R =N|{Ints2 N+1}
end

% Lazy suspension, thread created in lazy function
declare
proc{Ints2 N R}
  thread {WaitNeeded R} R =N|{Ints2 N+1} end
end
```

Lazy suspension
```
R = {Ints 1}
  |
  -> R -> {Ints 1} || body of {Ints 1} does WaitNeeded on R in his own thread

% If we do
L = {Ints 1}
  |
  -> L -> {Ints 1}

L.1 / L = 1|L' / L' is a new lazy suspension {Ints 2}
L.2.1 / L = 1|2|L'' / L'' is a new waiting thread, lazy suspesion, which is {Ints 3}
```

Bigger example

Producer creating the list -> S -> Consumer

Two possibilities :

1. Eager producer
  - Producer generates elements
2. Lazy producer
  - The consumer is now in control of elements, he asks for the elements, determines how many elements are made

Producer-consumer example

```
% Eager version : producer in control

declare
fun {Prod A N}
  if N == 0 then nil
  else A|{Prod A+1 N-1} end
end

fun {Sum L Acc}
  case L of nil then Acc
  [] H|T then {Sum T Acc+H} end
end

declare L S in
thread L = {Prod 1 100} end
thread S = {Sum L 0} end
{Browse S}

% Lazy version : consumer in control

declare
fun lazy{Prod A}
  A|{Prod A+1}
end

fun {Sum L Acc N}
  if N == 0 then Acc
  else {Sum L.2 L.1+Acc N-1} end
end

declare L S in
L = {Prod 1}
thread S = {Sum L 0 100} end
{Browse S}
```

The difference between eager and lazy, basically they are the same consumer and producer.   
But who controls the process is not the same, in the lazy case, it's the consumer that decides how many
elements will be produced.    
Lazy function is doing the minimum amount of work, this is why it's **lazy**.   

Let's go one step further with producer-consumer.    

**Problems about eager or lazy**
If the eager producer go faster than consumer, there is a risk of memory overflow, but it's fast.   
Lazy version has no problem of memory overflow, but it can have a problem of slow operations, maybe it would be better
if the producer has already computed some next elements to accelerate the process.   

What if we want fast and no memory problems ?
The answer is *bounded buffer*.   

```
Producer <-> Bounded Buffer <-> Consumer

The bounder buffer has a size of N elements
Less than N elements inside, will act eager on asking elements to the producer until it's full and then stop asking

When the consumer asks elements, he asks to the buffer which will deliver these very fast.

No memory leak thanks to the buffer, up to the size N.
```

### Bounded Buffer

```
% Bounded buffer : Combines advantages of eager and lazy

% 1. First step : define producer and consumer

declare

fun lazy{Prod N}
  {Delay 500}
  N|{Prod N+1}
end

fun lazy{Cons S Acc}
  case S of H|T then Acc|{Cons T H+Acc} end
end

% Purely lazy, doesn't do anything
declare S1 S2 in
S1 = {Prod 1}
S2 = {Cons S1 0}
{Browse S1}
{Browse S2}

{Browse S2.1} will force lazy suspension to activate, and also S1 to activate
```

Explanation

```
S1 = {Prod 1}
S2 = {Cons S1 0}

S2.1 activates Cons lazy, execute body of Cons
Also activating Prod lazy
proc {Cons S Acc R}
  thread {WaitNeeded R}
  R = case S of H|T thread end end
end

Builds the thing upstream, S5.1 needs S4.1, needs S3.1 etc
```

Keeping the same producer and the same consumer, we add the buffer

```
% 2. Building the bounded buffer - step by step

declare

fun lazy{Prod N}
  {Delay 500}
  N|{Prod N+1}
end

fun lazy{Cons S Acc}
  case S of H|T then Acc|{Cons T H+Acc} end
end

declare
proc {BounderBuffer S1 S2 N}
  ...
end

% 3. First copy the input to the output

proc {BounderBuffer S1 S2 N}
  fun lazy {Loop S1}
    case S1 of H1|T1 then
      H1|{Loop T1}
    end
  end
in
  S2 = {Loop S1}
end

% 4. At startup : ask for N elements
% whenever consumer asks, ask for one more
% asking for elements should not block main code

proc {BounderBuffer S1 S2 N}
  fun lazy {Loop S1 End}
    case S1 of H1|T1 then
      H1|{Loop T1 thread End.2 end}
    end
  end
in
  thread End = {List.drop S1 N} end
  S2 = {Loop S1 End}
end

% 5. Example execution of bounded buffer

declare S1 S2 S3 in
S1 = {Prod 1}
{BoundedBuffer S1 S2 3}
S3 = {Cons S2 0}
{Browse S1}
{Browse S2}
{Browse S3}
{Browse S3.1} % Get it right away because already inside the buffer
{Browse S3.2.1}
{Browse S3.2.2.2.1} % Buffer will have to ask the producer
```

## Three paradigms
- Functionnal programming
- Deterministic dataflow (more)
- Lazy deterministic dataflow (more)

## Kernel language

```
% Functionnal programming
<s> := skip
    | <s>1 <s>2
    | <x> = <v>
    | <x>1 = <x>2
    | local <x> in <s> end
    | {<x> <x>1 ... <x>n}
    | if <x> then <s>1 else <s>2 end
    | case <x> of <p> then <s>1 else <s>2 end
    % Extension with deterministic dataflow
    | thread <s> end
    % Extension with lazy deterministic dataflow
    | {WaitNeeded <x>}
```

They are all **declarative**, all the executions will give the same result.

## Hamming problem
Typical lazy stream problem.   

```
n = 2^a 3^b 5^c

Problem :
- Arrange in increasing order
- Incremental : you don't know how many in advance

Laziness is the key !
```

```
H = 1|2|3|4|5|6|...

2*H = 2|4|..
3*H = 3|...
5*H = 5|...

Evaluating always with the smallest number in the lists



/* HAMMING PROBLEM */

declare
fun lazy{Times S N}
  case L of H|T then N*H|{Times T N} end
end

declare
fun lazy{Merge S1 S2}
  case S1|S2 of (H1|T1)|(H2|T2) then
    if H1<H2 then H1|{Merge T1 S2}
    elseif H1>H2 then H2|{Merge S1 T2}
    else /* H1 == H2 */ H1|{Merge T1 T2} end
  end
end

declare
H = 1|{Merge {Times H 2}
        {Merge {Times H 3}
          {Times H 5}}}

{Browse H}
{Browse H.2.1}

% On way to force lazy to create element :

declare
proc {Touch L N}
  if N == 0 then skip
  else {Touch L.2 N-1} end
end

{Touch H 50}
{Touch H 100}
{Touch H 1000}

```

```
H = 1|H'
H' -> {Merge A M} | A -> {Times H 2}
                  | M -> {Merge B C} | B -> {Times H 3}
                                     | C -> {Times H 5}

H'.1 Activate a lazy suspension, and create a new lazy suspension : H' = 2|H''
```



## Lazy quicksort

```
% Eager quicksort
declare
proc {Partition L X L1 L2}
  case L of Y|M then
    if Y =< X then
      L1 = Y|M1
      {Partition M X M1 L2}
    else M2 in
      L2 = Y|M2
      {Partition M X L1 M2}
  end
  [] nil then L1 = nil L2 = nil end
end

declare
fun {Append L1 L2}
  case L1 of X|M1 then X|{Append M1 L2}
  [] nil then L2 end
end

declare
fun {Quicksort L}
  case L of X|M then L1 L2 S1 S2 in
    {Partition M X L1 L2}
    S1 = {Quicksort L1}
    S2 = {Quicksort L2}
    {Append S1 X|S2}
  [] nil then nil
  end
end

{Browse {Quicksort [4 6 2 7 6 5 6 4]}}

% Lazy quicksort

declare
fun lazy {Append L1 L2}
  case L1 of X|M1 then X|{Append M1 L2}
  [] nil then L2 end
end

declare
fun lazy {Quicksort L}
  case L of X|M then L1 L2 S1 S2 in
    {Partition M X L1 L2}
    S1 = {Quicksort L1}
    S2 = {Quicksort L2}
    {Append S1 X|S2}
  [] nil then nil
  end
end

declare
S = {Quicksort [3 8 7 6 9 2 1 0 8 9 4]}
{Browse S.1}

% Lazy quicksort becomes O(n) instead of eager O(n log n)
% Smallest k elements = O(n + k log k)
% K is not known in advance
% Lazy is inventing a new algorithm in some way
```
