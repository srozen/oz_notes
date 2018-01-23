# 29/03/17

# Summary

- Port objects
- Message protocols
  - RMI
  - Asynchronous RMI
  - RMI with callback thread version continuation version
- Active Objects
- Flavius Josephus problem
  - Comparing deterministic dataflow the message passing


Deterministic dataflow not sufficient for applications like servers.
We then added WaitTwo (cumbersome, only one non-deterministic feature)
and Ports(named streams) to permit non-deterministic dataflow.   

To make it easier to use, we define port objects. They have an internal state and we
can send messages to them and it modifies their internal state.   


# Message Protocols

Message protocols is a sequence of messaging between port objects.

```oz
  client      Server  
    |           |
    | ->msg     |
    |           |
    |      rsp<-|
    |           |
```
F : Msg x State -> State
P = {NewPort S}
{Send P M}

```oz
%Port object
declare
fun{NewPortObject F I}
  % Loop is deterministic
  proc {Loop S State}
    case S of M|S2 then
      {Loop S2 {F M State}}
    end
  end
in
  thread S in
    {NewPort S P}
    {Loop S I}
  end
  P
end

% Simple Counter Object
declare
C = {NewPortObject
      fun {$ Msg State}
        case Msg of inc then State+1
        [] get(x) then X=State State
        end
      end
      0}

local X in
{Send C get(X)} {Browse X} end

{Send C inc}

for I in 1.1000 do {Send C inc} end
```

Message Protocol and Client-Server Protocol

```oz
% Message protocols

% Client-server RMI protocol

% Stateless server

% Port object with no internal state

declare
fun {NewPortObject2 Proc} P in
  thread S in
    {NewPort S P}
    for M in S do {Proc M} end
  end
  P
end

% Stateless server (Behaviour, it's the Proc to use just above !)

declare
proc {ServerProc Msg}
  case Msg
    of calc(X Y) then
      Y=X*X+2.5*X+10.0
  end
end

% Create one server

declare
Server = {NewPortObject2 ServerProc}

declare Y in
{Send Server calc(34.4 Y)} {Browse Y}

% Client that will send requests to server

declare
proc {ClientProc Msg}
  case Msg
  of work(Y) then
    {Send Server calc(10.0 Y)}
    {Wait Y}
  end
end
```

Two request, Y = Y1+Y2 in the end
RMI = Remote Message Invocation
(Looks like alternating bit protocol)

```oz
declare
proc {ClientProc Msg}
  case Msg
  of work(Y) then Y1 Y2 in
    {Send Server calc(10.0 Y1)}
    {Wait Y1}
    {Send Server calc(10.0 Y2)}
    {Wait Y2}
    Y = Y1+Y2
  end
end
Client = {NewPortObject2 ClientProc}

local Y in {Send Client work(Y)} {Browse Y} end
```

## Asynchronous RMI

We Send two messages immediately, looks like windowed tcp sending

```oz
% Client that sends request to server
% Asynchronous RMI

% The change in the code is very small but sufficient
% We just removed irrelevant wait from the previous version
% Still very simple

declare
proc {ClientProc Msg}
  case Msg
  of work(Y) then Y1 Y2 in
    {Send Server calc(10.0 Y1)}
    {Send Server calc(10.0 Y2)}
    %{Wait Y1}
    %{Wait Y2} % will do the waits
    Y = Y1+Y2
  end
end
Client = {NewPortObject2 ClientProc}
local Y in {Send Client work(Y)} {Browse Y} end
```

{Send P n}
- Non blocking
- Immediately returns
- Eventually adds M to P's stream

## RMI with callback

We have a stateless server, no particular memory of the client.
What if the server needs informations from the client to do its work?
When the client sends message to the server, the server will send the message back to the
client to get some information


```oz
  client      Server  
    |           |
    | ->msg     |
    |           |
    | needinfo<-| callback
    | -> info   |
    |           |
    |   resp<-  |
```

```oz
% RMI with callback

declare
proc {ServerProc Msg}
  case Msg
  of calc(X Y Client) then
    {Send Client delta(D)}
    Y=X*X+125.0*D*X+X+23.3
  end
end

Server = {NewPortObject2 ServerProc}

% Buggy
declare
proc {ClientProc Msg}
  case Msg
  of work(Y) then
      {Send Server calc(10.0 Z Client)}
      % !!!! DEADLOCK Situation because it Waits for Z, but Z waits for D
      Y=Z+10.0 % waits for the Z
  [] delta(D) then
    D=0.1
  end
end

Client = {NewPortObject2 ClientProc}

local Y in {Send Client work(Y)} {Browse Y} end

% CORRECT VERSION
% Rule : port object should never wait inside a method
declare
proc {ClientProc Msg}
  case Msg
  of work(Y) then Z in
      {Send Server calc(10.0 Z Client)}
      thread Y=Z+10.0 end % waits for the Z
  [] delta(D) then
    D=0.1
  end
end


Client = {NewPortObject2 ClientProc}

% Note : create new server AND new client to make it work
local Y in {Send Client work(Y)} {Browse Y} end
```

### RMI with continuation version

```oz
% RMI with callback using a record continuation
declare
proc {ServerProc Msg}
  case Msg
  of calc(X Y Client) then Z in
    {Send Client delta(D)}
    Z=X*X+125.0*D*X+X+23.3
    {Send Client cont(Y Z)} % Rest of the work
  end
end

% Break the method into pieces
% This guarantees that methods will never wait inside

declare
proc {ClientProc Msg}
  case Msg
  of work(Y) then
      {Send Server calc(10.0 Y Client)}
  [] cont(Y Z)
      Y=Z+10.0 % waits for the Z
  [] delta(D) then
    D=0.1
  end
end
Client = {NewPortObject2 ClientProc}
local Y in {Send Client work(Y)} {Browse Y} end
```

Combination of the two

```oz
% Asynchronous RMI with callback
% This version is not completely asynchronous : delta !
% Fix: create new thread in the server

declare
proc {ServerProc Msg}
  case Msg
  of calc(X Y Client) then
    {Send Client delta(D)}
    Y=X*X+125.0*D*X+X+23.3
  end
end

% Fixed server
declare
proc {ServerProc Msg}
  case Msg
  of calc(X Y Client) then
    {Send Client delta(D)}
    thread Y=X*X+125.0*D*X+X+23.3 end
  end
end

Server = {NewPortObject2 ServerProc}

declare
proc {ClientProc Msg}
  case Msg
  of work(Y) then Z1 Z2 in
      {Send Server calc(10.0 Z1 Client)}
      {Send Server calc(20.0 Z2 Client)}
      thread Y=Z1+Z2 end % waits for the Zs
  [] delta(D) then
    D=0.1
  end
end


Client = {NewPortObject2 ClientProc}

% Note : create new server AND new client to make it work
local Y in {Send Client work(Y)} {Browse Y} end
```

# Active Objects

- Port objects defined with a class instead of a case
- Same abilities as port objects
- Introduce concept of a class

- Passive object runs in the caller thread
- Active object runs in its own thread

```oz
% Active Objects

% Class
declare
classe Counter
	attr i
	meth init
		i := 0
	end

	meth inc
		i := @i + 1
	end

	meth get(X)
		X=@i
	end
end

% A counter object
declare
C0 = {New Counter init}

% Invoking looks like calling a 1-arg procedure :
{C0 inc}

local X in {C0 get(X)} {Browse X} end

% Runs in the caller thread : (=passive object)
{C0 inc}
{C0 inc}

% What happens if a passive object is called in two threads
thread {C0 inc} end
thread {C0 inc} end
% WHat is the final result?

% We can't use passive objects in concurrent programs, we must use active objects !
% It can be one or two increments !! -> BUGGY
% Passive objects do not work in concurrent programs

% Active object: like a passive object but runs in own thread
declare
fun {NewActive Class Init}
  Obj = {New Class Init}
in
  thread
    {NewPort S P}
    for M in S do {Obj M} end
  end
  % same interface as passive object
  proc {$ M} {Send P M} end
end

declare
C1 = {NewActive Counter init}

% Works fine in multiple threads
thread {C1 inc} end
thread {C1 inc} end
local X in {C1 get(X)} {Browse X} end
```

# Flavius Josephus problem

Compare :
- deterministic dataflow
- active objects

40 soldiers in circle, k = 3

The soldiers will be agents or active objects, they receive a message, compute it and
pass the message.

### Protocol

When object receives kill(X S)
- alive and S=1 : last survivor
- alive and X mod k = 0 : becomes dead and send kill(X+1 S-1)
- alive and K mode K != 0 send kill(X+1 S)
- dead : sends kill(X+1 S) to next

kill(X S)
- S = number of live objects remaining
- X = number of objects traversed
