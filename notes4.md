# 01/03/17

## Summary

### Deterministic dataflow
- Concurrency for dummies
- Digital logic + Nbits adder

## Lazy deterministic dataflow
- Wait Needed operation
- Three kernel languages
- Lazy functions, lazy suspensions
- Infinite list
- Bounded buffer : combination of eager and lazy

## Deterministic dataflox

### Concurrency for dummies

```
% Concurrency for dummies
% Filter : takes a list and a function, returns elements for which function is true

declare
fun{Filter L F}
  case L of H|T then
    if {F H} then
      H|{Filter T F}
    else
      {Filter T F}
    end
  [] nil then nil
  end
end

{Browse {Filter [1 2 3 4 5] fun{$ X} X mod 2 == 0 end}}

% Turn the function into agent with S2 as output channel and S1 as input channel
% This agent could run in unexpected ways

declare S1 S2 in
thread S1 = {Filter S2 fun{$ X} X mod 2 == 0 end} end
{Browse S1}
{Browse S2}

S2 = 1|2|3|4|5|6|_

% Exemple where elements are unbound
% Will block waiting on the first unbound values

declare S1 S2 in
thread S1 = {Filter S2 fun{$ X} X mod 2 == 0 end} end
{Browse S1}
{Browse S2}
S2 = 1|2|3|_|_|6|7|8|_

% How to filter and continue over unboud values ?

% New version of Filter with extra thread, will block on the if

declare
fun{CFilter L F}
  case L of H|T then
    if thread {F H} end then
      H|{CFilter T F}
    else
      {CFilter T F}
    end
  [] nil then nil
  end
end

declare S1 S2 in
thread S1 = {Filter S2 fun{$ X} X mod 2 == 0 end} end
{Browse S1}
{Browse S2}

% Result is like before
% Because we need to know the value of {F H}
% To construct the list

%%%%%%%%%%%%%%%%%%%%%%%%%%%%%

% Map function
declare
fun {Map L F}
  case L of H|T then {F H}|{Map T F}
  [] nil then nil
  end
end

declare S1 S2 in
thread S1 = {Map S2 fun{$ X} X*X end} end
{Browse S1}
{Browse S2}

S2 = 1|2|3|4|5|_

% Map agent is blocked at the unbound variable
S2 = 1|2|3|4|5|_|_|8|9|10|_

% New version of Map

declare
fun {CMap L F}
  case L of H|T then thread {F H} end|{CMap T F}
  [] nil then nil
  end
end

declare S1 S2 in
thread S1 = {Map S2 fun{$ X} X*X end} end
{Browse S1}
{Browse S2}

S2 = 1|2|3|4|5|_|_|8|9|10|_

% Dependencies must be taken in account for threading

% CMap goes farther than Map
% It gives results for 8 9 10

% But Map will also give these results when the two vars are bound !

% Adding threads can make programs more
% incremental -> do more with the same input, but in never adds a bug

```

### Digital logic simulation

Input signal -> Output signal (eletric)   
Model this as a stream, which will 0 or 1 values    
Each values are time instants    

Assumption : Delay between input and output is smaller than delta(t)    
Means that logic gate is instantaneous, delay is negligible    

Conditionnal logic    
- Computes boolean logic
- Delay is not important

Sequential logic    
- logic of memory  
- based on delay : output depends on previous output

```
% Digital logic simulation

% Boolean function And

declare
fun{And A B}
  if A == 1 andthen B == 1 then 1 else 0 end
end

declare
fun{AndLoop S1 S2}
  case S1|S2 of (A|T1)|(B|T2) then
    {And A B}|{AndLoop T1 T2}
  end
end

declare S1 S2 S3 in
thread S3 = {AndLoop S1 S2} end

{Browse S1}
{Browse S2}
{Browse S3}

S1 = 0|1|1|_
S2 = 1|1|0|_

% Building many different kinds of gates
% Code for And gate will be made generic

declare
fun{GateMaker F}
  fun{$ S1 S2} % Calling this creates a gate
    % Recursive loop and thread are here
      fun{GateLoop S1 S2}
        case S1|S2 of (A|T1)|(B|T2) then
          {F A B}|{GateLoop T1 T2}
        end
  in
    thread {GateLoop S1 S2}
  end
end

% Create an And gate

declare AndG in
AndG = {GateMaker And}

% One And gate
declare S1 S2 S3 in
S3 = {AndG S1 S2}
{Browse S1}
{Browse S2}
{Browse S3}
```

### Nbits adder

Carry bit must be taken in account   
Remember Bouterfa
**TODO : Add NBIT ADDER schema if nothing better to do**

```
% Or gate

declare OrG in
OrG = {GateMaker
        fun{$ X Y}
          if X == 1 orelse Y == 1 then 1 else 0 end
        end}
% XOR Gate

declare XorG =
XorG = {GateMaker
          fun {$ X Y}
            if X == 1 andthen Y == 0 then 1
            elseif X == 0 andthen Y == 1 then 1
            else 0 end
          end}

% Full Adder
declare
proc{FullAdder X Y Ci S Co}
  A B C D E
in
  A = {AndG X Y}
  B = {AndG X Ci}
  C = {AndG Y Ci}
  E = {AndG A B}
  Co= {AndG E C}
  D = {XorG X Y}
  S = {XorG D Ci}
end

declare X Y Ci S Co in
{FullAdder X Y Ci S Co}
{Browse X}
{Browse Y}
{Browse Ci}
{Browse S}
{Browse Co}

X = 1|1|1|_
Y = 1|1|0|_
Ci = 1|0|0|_

% N-bit adder
declare
% Builds an N-bit adder
% Note : order of building is not important
% Also : can use adder while being built

proc {Adder Xs Ys Ci Ss Co}
  case Xs|Ys of (X|Xr)|(Y|Yr) then
    {FullAdder X Y Cm S Co}
    {Adder Xr Yr Ci Sr Cm}
    Ss = S|Sr
  [] nil|nil then % 0-bit adder
    Co = Ci
  end
end

% Testing the Adder
% Testing a 4-bit version
declare Y3 Y2 Y1 Y0 X3 X2 X1 X0 S3 S2 S1 S0 Ci Co in
{Adder [X3 X2 X1 X0] [Y3 Y2 Y1 Y0] Ci [S3 S2 S1 S0] Co}

```


## Lazy Deterministic Dataflow
- One new concept `{WaitNeeded X}`   
By-need synchronization    

- Introduce it by explaining dataflow implementation  

```

C = A + B

{Wait A}
{Wait B}
local X in
  {PrimAdd A B X} % Machine instruction
  {Bind C X}
end

% Sync between
% {Wait}
% {Wait}
% {Bind}
```

- {Wait X} waits until somebody does {Bind X}
- {WaitNeeded X} waits until somebody does {Wait X}

- {Wait X} -> "I need X"
- {WaitNeeded X} -> Wait until somebody needs X

```
% Example of WaitNeeded
declare X in
{WaitNeeded X}
X = 100*100
{Browse X} % Browse doesn't need X, will display _
{Browse X+1} % This addition will need X and cause X to be computed !


% With WaitNeeded we can define lazy functions

% Example of an eager function (normal like we know)
declare
fun{F1 X} X*X end
{Browse {F1 100}}

% Example of a lazy function
declare
fun lazy {F2 X} X*X end
declare B in
B = {F2 100}
{Browse B}
{Browse B+0}

% Lazy functions are defined with WaitNeeded
% Here is the definition of F2 :
declare
proc {F2 X R}
  thread {WaitNeeded R} R = X*X end
end
```
