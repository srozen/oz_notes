# 08/02/17
_Sections 1.1 to 1.9_

Theme : concurrent programming

Programming paradigms :

- Functionnal programming (ML, Scheme, Scala)
  - [+thread]Deterministic dataflow(Stream-based) - [by-need synchro]Lazy deterministric dataflow(Haskell)
  - [+cell]Sequential OOP(Java,Scala,Python) - [+thread]Shared state concurrency(Java,Scala) - Active objects
  - [+port]Event-based programming(Event Loop) - [+thread]Message Passing Concurrenct(Erlang) - Active Objects (Go, Akka)

## Functionnal programming
  - no state
  - pure functions (no objects with memory)
  - foundation of all paradigms
  - higher-order programming
  - data abstractions buildable with high-order programming

Adding concepts leads to extra expressiveness, but makes it harder to reason

## Introduction to Oz and functional programming

{Browse 100} -> display 100, {} is a procedure call

* Variables and identifier
* lists
* Computing with lists
* Functions
* Conditional 'if'
* Recursive functions
* Pattern Matching

### variables and identifier

declare
X = 100*100

X is an indentifier in the program - Start with **uppercase**
x is the variable in memory

Variables are **single-assignement** in functional programming

### Data structures

#### List
Pairing construction
x y -> x|y

```
<List> ::= nil
        | <E> '|' <List>
```

Example :

```
declare
X = 1|2|3|nil

declare
X = (1|nil)|(2|nil)|nil
```

Graphical representation as a binary tree

##### Computing with lists

```
declare
L = 1|2|3|4|5|nil

{Browse L.1} -> 1 (Head)
{Browse L.2} -> 2|3|4|5|nil (Tail)
{Browse L.2.2.2.1} -> 4
```

#### Function

* Same computation, repeat with many values
* Define new functions

fun declares the function

```
fun {Four L}
  L.2.2.2.1
end

{Browse {Four L}}
```

Functions are values in memory

#### Conditional

```
if X then Y
else Z
end
```

### Recursive functions

Two rules :
- Function follows type
- Recursive call is last operation, invariant programming -> define the invariant (Tail recursion)
 - define an invariant (Relationship between variables)
 - with code : all the part that change are arguments

#### Bad Recursive

This one is bad because not terminal recursive, the '+' is the last operation done
```
fun {BadLen L}
  if L == nil then 0
  else 1 + {BadLen L.2} end
end
```

'A' is the accumulator, passed as argument
```
fun {GoodLen L A}
  if L == nil then A
  else {GoodLen L.2 A+1} end
end
```

Works by invariant programming
* Both L & A change at each recursive call
* Relationship doesn't change between L&A

length(Linitiation) = length(L) + A

### Pattern matching

Pattern = Invisible representation of a data structure

List representable with H|T

```
fun {GoodLen L A}
  case L of H|T then {GoodLen T A+1}
  [] nil then A
  end
end

fun {GoodLen L A}
  case L of H|T then {GoodLen T A+1}
  [] H1|H2|T then {GoodLen T A+2}
  [] nil then A
  end
end
```

Pattern can be switched if independant, patterns should always be independant

### High-order programming
- functions are first-class entities, they can be passed as arguments

```
declare
fun {Map L F}
  case L of nil then nil
  [] H|T then {F H} | {Map T F}
  end
end
```

- Functions can be passed anonymous, using a '$' as a name (**Lambda Expression**)
