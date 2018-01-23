# 22/03/17

## Summary

### From declarative concurrency to message passing
- What is declarative concurrency ? (4.1.4)
- Limitations of declarative concurrency (4.8)
- Message-passing concurrency (chapter 5)
- Port objects


# From declarative concurrency to message passing

## What is declarative concurrency ?
- functionnal programming, y = f(x), x and y being typically values
- Deterministic dataflow  
Input can be a stream growing indefinitely or unbound variables?
What about unbound variables?
- Lazy deterministic dataflow

### Definition
**Partial Termination** : Give program an input, and eventually terminate.
What with a infinite stream? When the input stops growing, the program will
compute with this stream, and eventually the output will also stops growing.
This is the partial termination, not completely terminate, but the program
is suspended. Finite time of computation.

**Logical equivalence**
Store : {x=1, y=x, z}

Information as set of constraints : logical relation between variables :
- x < y
- a = b + 1
- x = 1
- l = x|l'

Any relation can be a constraint, but in a practical language we will have only
certain forms of constraints, typically equality constraints (bindings)

```oz
sigma  = c1 & c2 ...

sigma is the store (also a constraint)

c1 is a constraint (primitive constraint), an equality constraint
x = 1
x = y
x = record

```

If two stores have the same informations, they are equivalents.
They can have different bindings, but still have the same informations, so they
are equivalents.

Deterministic : The results are equivalents, maybe the computations are made in
different order.

#### Equivalent of constraints

```oz
C1 and C2 are equivalent
  if
  (1) same variables vars(C1) == vars(C2)
  (2) same information for each vars x :
    values(x,C1) == values(x,C2)
```


Another example

Sigma1 = {x = a|x', x', a = 1}
Sigma2 = {x = 1|x', a = 1, x'}

#### Logical equivalence

Two stores S1 and S2 are logically equivalents if they constain the same
information. The bindings may be differents.

With a program error, like assigning twice a variable, the program will no
longer be declarative. Still defined tho.
### Declarative Concurrency

A concurrent program is declarative if for all possible inputs, all executions
have following result.

- All executions do not terminate
- All executions event reach partial termination and results are logically
equivalent

#### Execution tree

Each branch is a choice of the scheduler,
it eventually reach a leaf, but they all have the same result.  


## Limitations of declarative concurrency

### Client-Server example

A server has to accept the two commands stream from the clients
C1 with stream S1 and C2 stream S2.
Impossible with the serveur, what if client 1 makes a command
and 2 makes a command ? We don't know who will comes first, it no longer
depends on the scheduler.    
The order of handling commands at server is not defined.  

This is observable nondeterminism.

Let's try writing it :

```oz
proc {Server S1 S2}
  case S1|S2 of (M1|T1)|(M2|T2) then
    do M1
    do M2
    {Server T1 T2}
    end
  end
end
```

Possible that client2 disconnect and never makes a command again, so server
will be blocked waiting for S2, and client1 will not be able to submit
commands.    

```oz
proc {Server S1 S2}
  case S1|S2 of (M1|T1)|S2 then do M1
  [] S1|(M2|T2) then do M2 end
end
```

**The case statement is deterministic**, it waits for something precise, for
one thing.

To program client/server, we need to extend the deterministic dataflow
with a new language concept. What is it??

### Extended Kernel Language   

F = {WaitTwo X Y} will wait on either X either Y is bound
  - x bound : returns 1
  - y bound : returns 2
  - X and y bound : returns 1 or 2
{Wait X} is deterministic

With WaitTwo, we can create the server :

```oz
% {WaitTwo X Y} waits until X or Y are bound
declare
fun{WaitTwo X Y}
  {Record.waitOr X#Y}
end

%Example

declare X Y in
{Browse {WaitTwo X Y}}

X=a
Y=a

% Expose nondeterminism of scheduler
thread {Delay 100} X=a end
thread Y=a end

% Server using WaitTwo

declare
proc{Server S1 S1}
  F={WaitTwo S1 S2}
in
  case F|S1|S2 of 1|(M1|T1)|S2 then
    %Do M1
    {Server T1 S2}
  [] 2|S1|(M2|T2) then
    %Do m2
    {Server S1 T2}
  end
end

```

To program client/server, we need to extend the deterministic dataflow
with a new language concept. Solution of **WaitTwo**

**Problem WaitTwo** : What about a million of clients on WaitTwo?
WaitTwo only accepts two possibilities, need to build a tree of WaitTwo

#### Second Solution : Port

Port : Named stream
S = a|b|c|S'

Name = a value inside the program that represents the stream

```oz

P = {NewPort S}

% P is the name of the stream
% S is the stream

{Send P M}

% M is the message

C1 : {Send P M1}
C2 : {Send P M2}

S = M1|LM2 or S = M2|M1
```

#### Semantics

Store : {p = §, s, p : s}

§ is a constant
s is a stream
: is association, p knows about S

If we do a {Send P M}

Store : {p = §, s=M|S', p : s'}


```oz

% Port = named stream
% P = {NewPort S}
% {Send P X}

% Client Server Example with ports
declare P S in
P = {NewPort S}

% Client 1
thread {Send P M1} end

% Client 2
thread {Send P M2} end

% Server

declare
proc {Server S}
  case S of M|T then
    % .. do M
    {Server T}
  end
end
```

#### Port objects

* Put port inside a thread with a recursive
* Similar to agents
* Initial state I P = {NPO F I}
* State transition have F , F : State x Msg -> State

```oz

% Port object

declare
fun {NewPortObject F I}
  proc {Loop S State}
    case S of M|T then
      {Loop T {F State M}}
    end
  end
in
  thread S in
    P = {NewPort S}
    {Loop S I}
  end
  P
end
```

```oz

% Example : simple counter

declare
Ctr = {NewPortObject
      fun {$ C M} {Browse C} C+1 end
      0}

{Send Ctr a}
thread {Send Ctr b} end
for I in 10..20 do {Send Ctr x} end

% Bigger example : playing ball

declare
fun {NewPlayer Pa Pb}
  {NewPortObject
  fun {$ State Msg}
    case State of state(Pa Pb C) then
      case Msg of ball then
        {Send State.(({OS.rand} mod2)+1 ) ball}
      [] get(X) then
        X = code
        State
      end
    end
  state(Pa Pb 0)}
end

% Create the game

declare Pa Pb Pc in
Pa = {NewPlayer Pb Pc}
Pb = {NewPlayer Pa Pc}
Pc = {NewPlayer Pa Pb}

{Browse {Send Pa get($)}}

% Start game
{Send Pa ball}
```
