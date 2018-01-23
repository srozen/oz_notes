# 10/05/17

# Summary

## Monitors and transactions
### Monitors
- Motivation
- Definition + semantics
- Bounded buffer example
- Programming pattern
- Implementation
### Transactions


# Monitors and transactions
Lock protects differents parts of the code
- Simple lock not good enough
- Reentrant lock which protects, if the thread is already inside (building abstractions)
Built with **Exchange** (read-write atomics)

Reentrant lock not good enough, protecting code is not good enough.

What if we build a bounded buffer :
- We put things into the buffer
- We get things out of the buffer

- Many threads put and get on the bounder buffer

```
N=3   
          push(a) | get(x) X=a will free a slot
          push(b) |
          push(c) |
Wait      push(d) |
```  

There is a coordination in this.

The bounded buffer is now an array of cells.  
-> Lock
Lock: waits for exclusive access
Buffer full: put must wait until not full, a *get* makes the buffer not full   
There is a coordination between the threads
Cannot be achieved with Reentrant locks.  


**Monitor** was invented to achieve concurrency on shared data structures.
Monitor is actually a lock + wait set (set of waiting threads)
The wait set is like a parking lot.

Going in the wait set : **wait**
Leaving the wait set : **notify/signal**

## Monitors
Reentrant lock

Operation **wait**

- Current thread is suspended
- then it is placed in the wait set (outside of the lock (lock is inside the BB)).
- Release the lock

We have to release the lock, otherwise it's a deadlock !

Operation **notify**  

- Remove a (*arbitrary*) thread T from the wait set
- Then T runs like normal, T proceeds to get the lock
- T resumes where it stopped
- (Kind of tricky operation)

Operation **notify all** (much easier)
- Does notify for all threads in the wait set

Example with a lot of printers BW and Color that wants to print,
wake up all the threads, would be stupid to wake up a color print
if no Color printer is available.  

But simple **notify** is more efficient but harder, **notify all** is simplier
but less efficient.

## Bounded buffer example

### Algorithm

Array from 0 to n-1
First and last position
Wrap around when we reach the end of the array !

```oz
% Bounded buffer
declare
class Buffer
  attr
    buf first last n

  meth init(N)
    buf := {NewArray 0 N-1 null}
    first := 0 last := 0 n:=N
    {NewMonitor @lockm @waitm @notifym @notifyallm}
  end

  meth put(X)
    {@lockm
      proc {$}
        % Wait until buffer is not full(@i<@n)
        if @i==@n then
          {@waitm}
          % Condition might become false here
          {self put(X)} % test condition again
        else
          % Now add one element
          @buf.@last:=X
          last:=(@last+1) mod @n
          i := @i+1
          {@notifyallm}
        end
    end}

  end

  meth get(X)
    {@lockm
      proc{$}
        % Wait until buffer is not empty(@i>0)
        if @i==0 then
          {@waitm}
          % Condition might become false here
          {self get(X)}
        else
          % Now remove one element
          X=@buf.@first
          first := (@first+1) mod @n
          i:=@i-1
          {@notifyallm}
        end
    end}
  end
end

% Example execution

declare
BB = {New Buffer init(3)}

{BB put(a)}

local X in {BB get(X)} {Browse X} end

{BB put(a)}
{BB put(b)}
{BB put(c)}
{Browse 'try fourth'}
{BB put(d)}
{Browse 'end fourth'}

local X in {BB get(X)} {Browse X} end

local X in {BB get(X)} {Browse X} end
{Browse 'after get'}

%%%%%%%%%%%%%%%%



```

### Monitor

{NewMonitor LockM WaitM NotifyM NotifyAllM}

{Lock proc {$}
  <s>
  {WaitM}
  {NotifyM}
  {NotifyAllM}
end }
