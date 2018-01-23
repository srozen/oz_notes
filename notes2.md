# 15/02/17

## Summary

- Programming techniques
  - Reverse with invariants
  - Merge sort
- Trees and accumulators
  - DFS Traversal
- Higher-order programming
  - Genericity, instantiation, composition fold, encapsulation
- Kernel language
  - Single assignement
  - Tail-recursive append

## Programming techniques

### Reverse with invariants

```
[1] Initial list L0
L = [a0 a1 ... aN-1]
L = a0|a1|...|aN-1|nil
```

Reversing the list with invariant which represents the intermediate result

```
[2] Reduced list L
ai|ai+1|...|aN-1|nil
ai-1|ai-2|...|a1|a0
[3] Accumulated result A
```

reverse(L0) = append(reverse(L), A) -> Mathematical formula

The reverse(L0) is constant, but the append can change (L and A)
With this invariant, we can write the program. L and A will be arguments.


```
fun{Reverse L A}
  case L
  of H|T then {Reverse T H|A}
  [] nil then A
  end
end
```

Main problem is boundary conditions, which is solved by the invariant.

### Merge sort


L = [a0 ... aN-1]
Split -> Merge Sort -> Merge
S : premutations of L elements in ascending order


L = [3 1 4 1 5 9 2 6 5]
_Split N complexity_
[3 4 5 2 5] [1 1 9 6]
_Sort_
[2 3 4 5 5] [1 1 6 9]
_Merge LogN complexity_
[1 1 2 3 4 5 5 6 9 ]

```
% Buggy - Infinite loop because not following 1st rule
declare
fun {MergeSort L}
  L1 L2 S1 S2
in
  {Split L L1 L2}
  S1 = {MergeSort L1}
  S2 = {MergeSort L2}
  {Merge S1 S2}
end

% Also buggy because not using the right type neither
declare
fun {MergeSort L}
  case L of H|T then
    L1 L2 S1 S2
  in
    {Split L L1 L2}
    S1 = {MergeSort L1}
    S2 = {MergeSort L2}
    {Merge S1 S2}
  [] nil then
    nil
  end
end

% True fixed : using correct type!
declare
fun {MergeSort L}
  case L of H1|H2|T then
    L1 L2 S1 S2
  in
    {Split L L1 L2}
    S1 = {MergeSort L1}
    S2 = {MergeSort L2}
    {Merge S1 S2}
  [] H|nil then
    L
  [] nil then
    nil
  end
end

declare
proc {Split L L1 L2}
  case L of H1|H2|T then T1 T2 in
    % H1 into L1, H2 into L2
    L1=H1|T1
    L2=H2|T2
    {Split T T1 T2}
  [] H|nil then
    L1=nil L2 = H|nil
  [] nil then
    L1=nil L2=nil
  end
end

%test Split
declare L1 L2 in
{Split [3 1 4 1 5 9 2 6 5] L1 L2}
{Browse L1}
{Browse L2}

declare
fun {Merge L1 L2}
  case L1|L2 of
    (H1|T1)|(H2|T2) then
    if H1<H2 then H1|{Merge T1 L2}
    elseif H1>H2 then H2|{Merge L1 T2}
    else /* H1==H2 */ H1|H2|{Merge T1 T2}
    end
  [] nil|(H2|T2) then
    L2
  [] (H1|T1)|nil then
    L1
  [] nil|nil then
    nil
  end
end
```

The merge is tail-recursive because of the single-assignement.

## Higher-order programming

The order of :
  - Base value : 0
  - Function returning value : 1
  - Function taking a function of order N as argument : N+1

Contextual environnement : Values used to build the function.

```
% Fold - Hiding an accumulator

declare
fun {FoldL L F U}
  case L of H|T then {FoldL T F {F U H}}
  [] nil then U
  end
end
```

Hide a value inside a function by encapsulation

```
declare
fun {Zero} 0 end
fun {Inc H} N in
N = {H} + 1
  fun {$} N end
end

 Three = {Inc{Inc{Inc Zero}}}
```

## Trees and accumulators

```
<Tree> ::= leaf
        |  tree(<Int> <Int> <Tree> <Tree>)  
```


Tree traversal - DFS
```
declare
proc{Traverse T}
  case T of tree(K V T1 T2) then
    {Browse K|V}
    {Traverse T1}
    {Traverse T2}
  [] leaf then
    skip
  end
end

declare
T = tree(10 100
         tree(9 81
              tree(4 16 leaf leaf)
              leaf)
         tree(11 121
              tree(14 196 leaf leaf)
              tree(13 169 leaf leaf)))


```

Gathering all the nodes into a list
Convert the tree into a list
Achievable by using an accumulator

```
% Bad because not using accumulator, the last call is not the recursion
declare
fun{Tree2List T}
  case T of tree(K V T1 T2) then L1 L2 in
    L1 = {Tree2List T1}
    L2 = {Tree2List T2}
    {Append L1 (K|VL)|2}
  [] leaf then
    nil
  end
end

% Accumulator version
declaire
proc {Tree2ListAcc T S1 Sn}
  case T of tree(K V T1 T2) then S2 S3 in
    S1 = (K|V)|S2
    {Tree2ListAcc T1 S2 S3}
    {Tree2ListAcc T2 S3 Sn}
  [] leaf then
    S1=Sn
  end
end

declare S1 in
{Tree2ListAcc T S1 nil}
{Browse S1}
```
