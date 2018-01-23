# 22/02/17

# Deterministic Dataflow

- single-assignement variables
- partial values
- simple dataflow
- concurrency and threads
- semantics : executions and interleaving
- execution tree
- streams and agents

## single-assignement variables

```
declare X in
{Browse X}

X=10 % Feeded after
```

**Semantics**

Declare is like local for the whole program
Browse creates an activity which will wait until X is bound, when X is bound then the display is updated

```
%Partial values
declare T U in
T = a|b|c|U|nil
{Browse T}

```

1. Creates parts of the result ` [a b c U] `
2. Create another part of the result `U=...`
3. U is the shared variable

U can be assigned later
Single assignment allows to split a program into independant pieces, and it is still functionnal programming
Change completely the organisation of the program -> Consequences on concurrency


## Simple Dataflox

```
% activity 1
declare X Y in
X = Y + 1
{Browse X}

% activity 2
Y = 11*11
```

Y is not defined yet, the dataflow idea is to wait on the X assignment until Y is bound to some value.
The execution will wait till Y is bound

Shared variables causes synchronization between 2 activities, 1 will be finished after 2
These independant activities are threads

```
local X Y in
  X=Y+1
  {Browse X}
end
```

No way to bind Y, the Y is unaccessable because of the local scope. Deadlock situation.
Garbage collector will remove it, GC remove an activity that is unable to continue.


## Threads

- Functionnal programming + threads = deterministic dataflow
- thread < S > end

```
declare A B C D in
thread B = A+1 {Browse B} end
thread C = B+1 {Browse C} end
thread D = C+1 {Browse D} end
A = 100
```

### Thread semantics

<!--- SEE p61 --->

**What is an execution?**
*Instructions s - Memory o*
Execution = sequence of computation steps   
(s,o) -> Step1 -> (s', o')   

With one activity   

```
|X=1|                              |Y=2|
|Y=2|           -> 1 Step(X=1) ->  |Z=3|
|Z=3|,{x,y,z}                      |   |,{x=1,y,z}
```

with two activities   

```
|X=3||Y=2|             -> 1 Step(Y=2) ->  |X=3||Z=3|
|W=5||Z=3|,{w,x,y,z}                      |W=5||   |,{x,w,y=3,z}
```

With two activity, **choose** an activity to execute first ! Which one?    
The choice is controlled by the system, the programmer has not so much of a choice between the activities, but the choice is not arbitrary -> **Nondeterminism**.    
These activities, threads, are independant.    
WHith the principle of **fairness**, it is ensured that every activity will eventually get chosen to do a step.    

The scheduler in the system is responsible for making these choices.   


Execution with multiple activities     
(MS, o) -> 1 step -> (MS', o')     

Multiset of stacks (MS)    

({S1,S2,S3},o) -> ({S1,S2,S3'},o')    

This is **interleaving**   

### Remarks on efficiency of threads
Creating threads takes time   
` thread A=1 end `    
A thread is a "unit of waiting|work", delegate waiting to thread, tasks    

The efficiency depends on the language, Java threads are expensive, Erlang threads are cheap because it is concurrency-oriented    

Exemple of cheap threads   

```
declare
fun{Fibo N}
  if N==1 orelse N==2 then 1
  else {Fibo N-1} + {Fibo N-2} end
end
```

Concurrent Fibonacci

```
declare
fun{CFibo N}
  {Delay 500}
  if N==1 orelse N==2 then 1
  else thread {Fibo N-1} end + {Fibo N-2} end
end
```

## Execution Tree

Execution = path in the tree starting from root

```
thread A=1 end
thread B=2 end
thread C=A+B end
```

node = exec.state


One path within the tree is an execution.   
Schedule chooses, Nondeterminism   
Result the same, no observable nondeterminism   


Programming with deterministic dataflow -> Streams and agents

## Stream and agents

- stream = communication channel
- agent = activity with communication channels
------------
- stream = list with unbound tail, list under construction

```
L = a|b|c|nil
L = a|b|c|L2
  L2 = d|e|L3
    L3 = f|nil
```

Activity 1 creates the stream (list under construction)

```
declare
fun {Interval L H}
  if L>H then nil
  else L|{Interval L+1 H} end
end

{Browse {Interval 10 20}}
```

Activity 2 read a list, return accumulated sum

```
declare
fun {Sum L A}
  case L of H|T then H+A|{Sum T H+A}
  [] nil then nil
  end
end

{Browse {Sum [1 2 3 4 5] 0}}
{Browse {Sum {Interval 10 20} 0}}
```

Now turn Sum into an agent, input list is now a stream   
Browse in another agent

```
declare S T in
thread T={Sum S 0} end
thread {Browse T} end

S=1|2|3|4|5|_
```

Any list function which takes 0 or more list as input, O or more list as output can be
converted into an agent.
- inside a thread
- give it streams instead of complete lists
