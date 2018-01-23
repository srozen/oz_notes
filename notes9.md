# 19/04/17

# Summary

- Multi-agent programming
  - Comparing deterministic dataflow with port objects (Flavius Josephus)
  - Program design for concurrency
  - Example : Lift control system

# Flavius Josephus

N objects in circle
Remove each Kth object
Number of last remaining objects?

Message kill(X S)
```oz
X : Number traversed
S : Number live remaining
```

Protocol
```oz
receive kill(X S)
alive
  S > 1
  x mod k=0 -> dead; kill(X+1 S-1) to successor
  x mod k!=0 -> alive; kill(X+1 S) to successor

  S=1: done I is answer
dead
  kill(X S) to successor
```

Active objects uses classes

```oz
% Flavius Josephus Problem

declare
class Victim
  attr ident k last succ alive
  meth init(I K Last)
    ident := I
    k := K
    last := Last
    alive := true
  end
  meth setSucc(S) succ := S end
  meth kill(X S)
    if @alive then
      if S==1 then
        @last=@ident
      else
        if X mod @k == 0 then
          alive := false
          {@succ kill(X+1 S-1)}
        else
          {@succ kill(X+1 S)}
        end
      end
    else
      {@succ kill(X S)}
    end
  end
end

declare
fun{Josephus N K}
  A={NewArray 1 N null}
  Last
in
  for I in 1..N do
    1.I := {NewActive Victim init(I K Last)}
  end
  for I in 1..N do
    {A.I setSucc(A.(I+1))}
  end
  {A.N setSucc(A.1)}
  {A.1 kill(1 N)}
  Last
end
```

Flavius Josephus Short-circuited

```oz
% Flavius Josephus Problem

declare
class Victim
  attr ident k last succ alive pred
  meth init(I K Last)
    ident := I
    k := K
    last := Last
    alive := true
  end
  meth setSucc(S) succ := S end
  meth setPred(S) pred := S end %SSP
  meth kill(X S)
    if @alive then
      if S==1 then
        @last=@ident
      else
        if X mod @k == 0 then
          alive := false
          {@pred setSucc()} %SSP
          {@succ setPred()} %SSP
          {@succ kill(X+1 S-1)}
        else
          {@succ kill(X+1 S)}
        end
      end
    else
      {@succ kill(X S)}
    end
  end
end

declare
fun{Josephus N K}
  A={NewArray 1 N null}
  Last
in
  for I in 1..N do
    1.I := {NewActive Victim init(I K Last)}
  end
  for I in 1..N do
    {A.I setSucc(A.(I+1))}
  end
  {A.N setSucc(A.1)}
  for I in 1..N do
    {A.I setPred(A.(I-1))}
  end % SSP
  {A.1 setPred(A.N)}% SSP
  {A.1 kill(1 N)}
  Last
end
```

Solution in Deterministic Dataflow

The Victim is now an agent with 1 output and 1 input stream

```oz
% Solution in deterministic dataflow

declare
fun{Josephus2 N K}
  Last Xs Zs
  fun{Victim Xs I}
    case Xs of kill(X S)|Xr then
      if S == 1 then
        Last = I
        nil
      else
        if X mod K == 0 then
          kill(X+1 S-1)|Xr
        else
          kill(X+1 S)|{Victim Xr I}
        end
      end
    [] nil then nil end
  end
  fun{Pipe Xs L H}
    if L>H then Xs
    else Ys in
      Ys={Pipe Xs L H-1}
      thread {Victim Ys H} end
    end
  end
in
  Zs={Pipe Xs 1 N}
  Xs=kill(1 N)|Zs
  Last
end

{Browse {Josephus2 5 2}}
{Browse {Josephus2 40 3}}
```

# Multi-agent programming

## Methodology

Program with concurrent components agents (for exemple, active objects as implementation)

### Discipline
1. Architecture
-> System made of concurrent compononents
- Instantiation
Class with attributes
- Composition
Components composed of sub-components etc
- Linking
Link can be streams etc
Multi-shot link is when you can send mutiple messages : **ports**
Sometimes we need one message on the link, nothing more : **One-shot links**
- Restriction
Visibility, hide links inside

## Architecture
I,C,L,R

## Design

1. Informal specification
2. Kinds of components we need
3. Message protocols
4. State diagrams of components
5. Implement (coding)
Schedule(fair)
6. Test and iterate
6. Invariants

##

Initial state :
- receive a message
- change state
- serve a message

---------

- State machine : connected to the world
- Transitions
