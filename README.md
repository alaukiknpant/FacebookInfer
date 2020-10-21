# Using Facebook Infer to Detect Data Races in Java

## Introduction

In this report, we explore the use of a state-of-the-art tool called [Facebook Infer](https://fbinfer.com) for finding concurrency bugs in large Java codebases.
Facebook's Infer is based on [this]() paper which describes RacerD - a
"static program analysis for detecting data races in Java programs which is fast,
can scale to large code, and has proven effective in an industrial software
engineering scenario."

RacerD has flagged over 2500 issues after being deployed for a year at Facebook.[[1]](https://ilyasergey.net/papers/racerd-oopsla18.pdf)
RacerD is used to detect data races, a concurrency bug. A data race in concurrent Java code occurs when two or more threads in a single process access the same memory location concurrently, and
at least one of the accesses is for writing, and the threads are not using any exclusive
locks to control their accesses to that memory.[[2]](https://ilyasergey.net/YSC3248/week-11-races.html)

RacerD's distinguishing feature is that it is compositional, i.e. it does not do a whole program
analysis. It does so by reasoning sequentially about memory accesses, locks and threads. This report
highlights the issues we found on popular non-trivial Java codebases using this tool.
___
#### Issues 1-4 refer to the following repo: [Native-platform: Java bindings for native APIs](https://github.com/gradle/native-platform)
___

## Issue 1

#### <ins> Error Report from Infer

```txt
native-platform/src/main/java/net/rubygrapefruit/platform/internal/TerminfoTerminal.java:203: warning: THREAD_SAFETY_VIOLATION
  Read/Write race. Non-private method `TerminalOutput TerminfoTerminal.bold()` indirectly reads without synchronization from `this.boldOn`. Potentially races with write in method `TerminfoTerminal.init()`.
 Reporting because a superclass `class net.rubygrapefruit.platform.terminal.TerminalOutput` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  201.       @Override
  202.       public TerminalOutput bold() {
  203. >         if (!supportsTextAttributes()) {
  204.               return this;
  205.           }
```

#### <ins> Relevant Code

Based on the error report generated by Infer, we know that the error is associated with the following method: `bold()`.

```java
@Override
public TerminalOutput bold() {
    if (!supportsTextAttributes()) {
        return this;
    }

    synchronized (lock) {
        write(boldOn);
    }
    return this;
}
```

Here, the class `TerminfoTerminal` that contains the method `bold()` operates in a multi-threaded environment. We know this because its superclass `terminal.TerminalOutput` is annotated as @ThreadSafe. The error description by Infer points to line 203 of the file `TerminfoTerminal.java`, which calls the method `supportsTextAttributes()`. Hence, to see if there is indeed a thread safety violation, we looked at the method `supportsTextAttributes()`.

```java
@Override
public boolean supportsTextAttributes() {
    return boldOn != null && dim != null;
}
```

We noticed that the method `supportsTextAttributes()` reads both `this.boldOn` and `this.dim`. We then examined where the field `this.boldOn` is read and written besides `supportsTextAttributes()`. It turns out that there are two such instances in the class `TerminfoTerminal`. First, the method `bold()` has a read access to `this.boldOn` when it calls `write(boldOn)`, where the `write(byte)` method is used to write a single byte to the Java OutputStream. Note that this is fine with the previous read access as two simultanous reads do not lead to a data race. Second, the method `init()` (see attached code below) has a write access to `this.boldOn` in the code `boldOn = TerminfoFunctions.boldOn(result)` (implementation of `init()` attached below).

```java
@Override
protected void init() {
    synchronized (lock) {
        FunctionResult result = new FunctionResult();
        TerminfoFunctions.initTerminal(output.ordinal(), capabilities, result);
        ...
        boldOn = TerminfoFunctions.boldOn(result);
        if (result.isFailed()) {
            throw new NativeException(String.format("Could not determine bold on control sequence %s: %s", getOutputDisplay(), result.getMessage()));
        }
        dim = TerminfoFunctions.dimOn(result);
        ...
    }
}
```

Since `supportsTextAttributes()` is not synchronized, while one thread is reading whether ```this.boldOn``` is not null in `supportsTextAttributes()`, another thread might write ```this.boldOn``` in init(), leading to a data race.

#### <ins> Solution

We can implement the following solution to resolve the data race described above:

```java
@Override
public TerminalOutput bold() {
    synchronized (lock) {
        if (!supportsTextAttributes()) {
            return this;
        }
    }

    synchronized (lock) {
        write(boldOn);
    }
    return this;
}
```
We implemented the above solution because the method `init()` is synchronized around the object called "lock". Recall that this definiton of lock is as follows:

                                              private final Object lock = new Object();

To avoid data race, we need to use insert "synchronized (lock)" around the if-statement with `supportsTextAttributes()` where it is currently missing. In this way, we use the same lock for all read and write access of `this.boldOn`. Hence the reading and writing the variable must occur in a mutually-exclusive manner, and the issue regarding this data race is resolved.

___

## Issue 2

Seeing that `supportsTextAttributes()` also reads `dim`, we suspect that the `dim()` method, which is implemented right below `bold()` in the same class `TerminfoTerminal`, might also have data race problems. Indeed, looking through the bug report generated by Infer, we found the following error identified by Infer.

#### <ins> Error Report from Infer

```txt
native-platform/src/main/java/net/rubygrapefruit/platform/internal/TerminfoTerminal.java:230: warning: THREAD_SAFETY_VIOLATION
  Read/Write race. Non-private method `TerminalOutput TerminfoTerminal.dim()` indirectly reads without synchronization from `this.boldOn`. Potentially races with write in method `TerminfoTerminal.init()`.
 Reporting because a superclass `class net.rubygrapefruit.platform.terminal.TerminalOutput` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  228.       @Override
  229.       public TerminalOutput dim() throws NativeException {
  230. >         if (!supportsTextAttributes()) {
  231.               return this;
  232.           }
```

#### <ins> Relevant Code

```java
@Override
public TerminalOutput dim() throws NativeException {
    if (!supportsTextAttributes()) {
        return this;
    }

    synchronized (lock) {
        write(dim);
        if (bright && foreground != null) {
            write(getColor(foreground, false));
        }
        bright = false;
    }
    return this;
}
```

Not surprisingly, this error is also caused by the under-synchronization of `supportsTextAttributes()`, which we have attached below.

```java
@Override
public boolean supportsTextAttributes() {
    return boldOn != null && dim != null;
}
```

Same as the previous thread safety violation, since `supportsTextAttributes()` is not synchronized, when one thread is reading `this.boldOn` by calling the method `supportsTextAttributes()`, another thread could potentially write `this.boldOn` via the method `init()`. In fact, there are two potential violations here: the two fields `this.boldOn` and `this.dim` are in the same situation, so a data race can also occur when two concurrent accesses to the same memory location `this.dim` where one of them is a write. In either case, we know that `init()`, which is the only method that contains a write access to either `this.boldOn` or `this.dim`, is synchronized with "lock", and hence to resolve the data race, we need to add synchronization on the method that contains a read access to these memory locations.

#### <ins> Solution

The fix of this data race is similar to the previous one. We simply implemented synchronization in the following way so that the reads and writes of `this.boldOn` or `this.dim` are accessed in a mutually-exclusive manner.

```java
@Override
public TerminalOutput dim() throws NativeException {
    synchronized (lock) {
        if (!supportsTextAttributes()) {
            return this;
        }
    }

    synchronized (lock) {
        write(dim);
        if (bright && foreground != null) {
            write(getColor(foreground, false));
        }
        bright = false;
    }
    return this;
}
```

Given that the two bugs in Issue 1 and Issue 2 are effectively the same, we figured that there might be better ways to implement synchronization. Instead of adding synchronization around the if-statement with “supportsTextAttributes()” in the `dim()` method, our second solution to solve the data race problem is to use synchronization directly on `supportsTextAttributes()`:

```java
public boolean supportsTextAttributes() {
    synchronized (lock) {
        return boldOn != null && dim != null;
    }
}
```

Again, we need to synchronize around the object called `lock` because we synchronize `init()` around the same `lock` object. This change would eliminate the error we had previously as well. In fact, we noticed that the second fix helps reducing the number of errors in the Infer report by 2, and we suspect that this is because another method that calls `supportsTextAttributes()` also had a similar data race problem previously and is now fixed through synchronization.

Just to double check that implementing synchronization in the `supportsTextAttributes()` method directly could resolve the data race in both Issue 1 and Issue 2, we tried removing the synchronziation lock we implemented in the solution of Issue 1 and the number of errors indeed did not increase.

___

## Issue 3 and 4

#### Error Report from Infer

_Error 1:_
```txt
native-platform/src/main/java/net/rubygrapefruit/platform/internal/TerminfoTerminal.java:315: warning: THREAD_SAFETY_VIOLATION
  Read/Write race. Non-private method `TerminalOutput TerminfoTerminal.hideCursor()` indirectly reads without synchronization from `this.hideCursor`. Potentially races with write in method `TerminfoTerminal.init()`.
 Reporting because a superclass `class net.rubygrapefruit.platform.terminal.TerminalOutput` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  313.       @Override
  314.       public TerminalOutput hideCursor() throws NativeException {
  315. >         if (!supportsCursorVisibility()) {
  316.               return this;
  317.           }
 ```

_Error 2:_

```txt
native-platform/src/main/java/net/rubygrapefruit/platform/internal/TerminfoTerminal.java:164: warning: THREAD_SAFETY_VIOLATION
  Read/Write race. Non-private method `boolean TerminfoTerminal.supportsCursorVisibility()` reads without synchronization from `this.hideCursor`. Potentially races with write in method `TerminfoTerminal.init()`.
 Reporting because a superclass `class net.rubygrapefruit.platform.terminal.TerminalOutput` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  162.       @Override
  163.       public boolean supportsCursorVisibility() {
  164. >         return showCursor != null && hideCursor != null;
  165.       }
```


##### Method 1

```java
@Override
public TerminalOutput hideCursor() throws NativeException {
    if (!supportsCursorVisibility()) {
        return this;
    }

    synchronized (lock) {
        write(hideCursor);
    }
    return this;
}
```

##### Method 2

```java
@Override
public boolean supportsCursorVisibility() {
    return showCursor != null && hideCursor != null;
}
```

##### Method 3

```java
@Override
protected void init() {
    synchronized (lock) {
        FunctionResult result = new FunctionResult();
        TerminfoFunctions.initTerminal(output.ordinal(), capabilities, result);
        if (result.isFailed()) {
            throw new NativeException(String.format("Could not open terminal for %s: %s", getOutputDisplay(), result.getMessage()));
        }
        ansiTerminal = isAnsiTerminal();
        hideCursor = TerminfoFunctions.hideCursor(result);             //****** hideCursor refereced here (Write operation)******
        if (result.isFailed()) {
            throw new NativeException(String.format("Could not determine hide cursor control sequence for %s: %s", getOutputDisplay(), result.getMessage()));
        }
        showCursor = TerminfoFunctions.showCursor(result);
        if (result.isFailed()) {
            throw new NativeException(String.format("Could not determine show cursor control sequence for %s: %s", getOutputDisplay(), result.getMessage()));
        }
        ...
    }
}
```

#### Analysis


Here, the class `TerminfoTerminal` that contains the afformentioned methods operates in a multi-threaded environment. We know this because
its superclass `terminal.TerminalOutput` is annotated as `@ThreadSafe` and also by the fact that the methods ```hideCursor()``` and ```init()``` have the synchronized modifiers in them. ```TerminfoTerminal``` has several data races and one such data race happens in a memory location ````this.hideCursor````.

 We claim that a data race occurs in this location because of the potential of two concurrent accesses to the same memory location where one of them is a write.
 One place where a thread can write to the ````hideCursor```` field happens in the method `TerminfoTerminal.init()`(referred in the error report 2) which is shown in Method 3 above. Notice that this method is already synchronized around the object called lock. Hence, we infer that subsequent blocks of code that order reads and writes to the memory location ````this.hideCursor```` must be synchronized on the object called ```lock```.
 
 There might be several other places where we read or write from this afformentioned memory location. We will point out two locations based on the error reports we got. First, we see a read of the ````hideCursor```` field in the```hideCursor()``` method (referring to code - `write(hideCursor)` - in Method 1). Another method where we read this memory location is in the ```supportsCursorVisibility()``` method (see Method 2 above), which is called by the ```hideCursor()``` method.

 To prevent a data race, we must synchronize each call to ```supportsCursorVisibility() ```. This can be done in two ways.  We can synchronize a call to this method within the ```hideCursor()``` method on the object called ```lock``` as a quick fix. However, a better solution that fixes both the errors would be to synchronize the entire ```supportsCursorVisibility()``` method around the object called ```lock```. This ensures that if other methods try to access the memory location ````hideCursor```` through ```supportsCursorVisibility()```, they have to synchronize on the object called ```lock```.


#### Solution


Quick fix for Error 1:

```java
@Override
public TerminalOutput hideCursor() throws NativeException {
    synchronized (lock) {
        if (!supportsCursorVisibility()) {
            return this;
        }
    }

    synchronized (lock) {
        write(hideCursor);
    }
    return this;
}
```


Better Solution:

```java
public boolean supportsCursorVisibility() {
    synchronized (lock) {
        return showCursor != null && hideCursor != null;
    }
}
```

## Issue 5

#### <ins> Error Report from Infer

```txt
native-platform/src/main/java/net/rubygrapefruit/platform/internal/AnsiTerminal.java:126: warning: THREAD_SAFETY_VIOLATION
  Unprotected write. Non-private method `TerminalOutput AnsiTerminal.bright()` writes to field `this.bright` outside of synchronization.
 Reporting because a superclass `class net.rubygrapefruit.platform.terminal.TerminalOutput` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  124.       public TerminalOutput bright() throws NativeException {
  125.           try {
  126. >             bright = true;
  127.               if (foreground != null) {
  128.                   outputStream.write(BRIGHT_FOREGROUND.get(foreground.ordinal()));
```

#### Relevant Code

```java
@Override
public TerminalOutput bright() throws NativeException {
    try {
        bright = true;
        if (foreground != null) {
            outputStream.write(BRIGHT_FOREGROUND.get(foreground.ordinal()));
        }
    } catch (IOException e) {
        throw new NativeException(String.format("Could not set foreground color on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

Here, the class `AnsiTerminal` operates in a multi-threaded environment. We know this because
its superclass `terminal.TerminalOutput` is annotated as `@ThreadSafe`. We claim that there exists one data race in the memory location `this.bright`.

In the above method `bright()`, one thread can write `this.bright` as `true` without synchronization. However, there are several other read and write access to `this.bright` in the class `AnsiTerminal`. For instance, the following method, `foreground(color)` has read access to `this.bright`.

```java
public TerminalOutput foreground(Color color) throws NativeException {
    try {
        if (bright) {
            outputStream.write(BRIGHT_FOREGROUND.get(color.ordinal()));
        } else {
            outputStream.write(FOREGROUND.get(color.ordinal()));
        }
        foreground = color;
    } catch (IOException e) {
        throw new NativeException(String.format("Could not set foreground color on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

Hence, we claim that a data race occurs in this location because of the potential of two concurrent accesses to the same memory location where one of them is a write. While one thread is writing to `this.bright` in `bright()` method, another thread might potentially be reading the variable `this.bright` in `foreground(color)`.

To prevent a data race, we can synchronize the call to `bright()` on the current instance (obtain lock on the current instance). We also synchronize other methods that read or write to the `bright` field on the current instance.

#### Solution

```java
@Override
synchronized public TerminalOutput bright() throws NativeException {
    try {
        bright = true;
        if (foreground != null) {
            outputStream.write(BRIGHT_FOREGROUND.get(foreground.ordinal()));
        }
    } catch (IOException e) {
        throw new NativeException(String.format("Could not set foreground color on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

```java
synchronized public TerminalOutput foreground(Color color) throws NativeException {
    try {
        if (bright) {
            outputStream.write(BRIGHT_FOREGROUND.get(color.ordinal()));
        } else {
            outputStream.write(FOREGROUND.get(color.ordinal()));
        }
        foreground = color;
    } catch (IOException e) {
        throw new NativeException(String.format("Could not set foreground color on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

Additionally, through a bit of searching, we noticed that `bright()` is not the sole method in the class `AnsiTerminal` without synchronization. We suspect that the following method, `dim()`, `normal()`, and `reset()`, might have a data race problem for the same reason.

```java
@Override
public TerminalOutput dim() throws NativeException {
    try {
        outputStream.write(DIM_ON);
        bright = false;
    } catch (IOException e) {
        throw new NativeException(String.format("Could not set foreground color on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

```java
public TerminalOutput normal() throws NativeException {
    try {
        outputStream.write(NORMAL_INTENSITY);
        if (foreground != null && bright) {
            outputStream.write(FOREGROUND.get(foreground.ordinal()));
        }
        bright = false;
    } catch (IOException e) {
        throw new NativeException(String.format("Could not switch to normal output on %s.", getOutputDisplay()), e);
    }
    return this;
}

public TerminalOutput reset() throws NativeException {
    try {
        outputStream.write(RESET);
        foreground = null;
        bright = false;
    } catch (IOException e) {
        throw new NativeException(String.format("Could not reset output on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

Hence, to resolve such race problems, we synchronize the call to the abovementioned methods on the current instance (obtain lock on the current instance). By fixing this issue with under-synchronization of the call to methods that contains the write access to `this.bright`, we were able to reduce the number of errors in the Infer analysis by 5.

## Issue 6

#### <ins> Error Report from Infer

```txt
native-platform/src/main/java/net/rubygrapefruit/platform/internal/AnsiTerminal.java:127: warning: THREAD_SAFETY_VIOLATION
  Read/Write race. Non-private method `TerminalOutput AnsiTerminal.bright()` reads with synchronization from `this.foreground`. Potentially races with unsynchronized write in method `AnsiTerminal.foreground(...)`.
 Reporting because a superclass `class net.rubygrapefruit.platform.terminal.TerminalOutput` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  125.           try {
  126.               bright = true;
  127. >             if (foreground != null) {
  128.                   outputStream.write(BRIGHT_FOREGROUND.get(foreground.ordinal()));
  129.               }
```

#### Relevant Code

```java
@Override
synchronized public TerminalOutput bright() throws NativeException {
    try {
        bright = true;
        if (foreground != null) {
            outputStream.write(BRIGHT_FOREGROUND.get(foreground.ordinal()));
        }
    } catch (IOException e) {
        throw new NativeException(String.format("Could not set foreground color on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

After fixing the previous bug, Infer is still complaining about the method `bright()`, but this time the problem is with something else. Notice that the `bright()` method checks whether `this.foreground` is not null, so this is a read access of the variable `this.foregorund`. Again, the class `AnsiTerminal` operates in a multi-threaded environment. We know this because
its superclass `terminal.TerminalOutput` is annotated as `@ThreadSafe`. Though the method `bright()` is now synchronized, there are under-synchronized methods with write access to `this.foreground` in the class `AnsiTerminal`, which leads to a data race in the memory location `this.foreground`.

Through a bit of research, we discovered that the following two methods, `foreground(color)` and `defaultForeground()` respectively, have write access to `this.foreground` and is currently not synchronized.

```java
public TerminalOutput foreground(Color color) throws NativeException {
    try {
        if (bright) {
            outputStream.write(BRIGHT_FOREGROUND.get(color.ordinal()));
        } else {
            outputStream.write(FOREGROUND.get(color.ordinal()));
        }
        foreground = color;
    } catch (IOException e) {
        throw new NativeException(String.format("Could not set foreground color on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

```java
@Override
public TerminalOutput defaultForeground() throws NativeException {
    try {
        outputStream.write(DEFAULT_FG);
        foreground = null;
    } catch (IOException e) {
        throw new NativeException(String.format("Could not switch to bold output on %s.", getOutputDisplay()), e);
    }
    return this;
}
```

Hence, to resolve such race problems, we synchronize the call to the abovementioned methods on the current instance (obtain lock on the current instance). By fixing this issue with under-synchronization of the call to methods that contains the write access to `this.foreground`, we were able to reduce the number of errors in the Infer analysis by 5 like in Issue 5.

## Issue 7

#### <ins> Error Report from Infer

```txt
native-platform/src/main/java/net/rubygrapefruit/platform/internal/FileStat.java:41: warning: THREAD_SAFETY_VIOLATION
  Unprotected write. Non-private method `void FileStat.details(int,int,int,int,long,long,int)` writes to field `this.modificationTime` outside of synchronization.
 Reporting because a superclass `class net.rubygrapefruit.platform.file.PosixFileInfo` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  39.           this.gid = gid;
  40.           this.size = size;
  41. >         this.modificationTime = modificationTime;
  42.           this.blockSize = blockSize;
  43.       }
```

#### Relevant Code









___



#### Issues 8-12 refer to the following repo: [Rx Java](https://github.com/ReactiveX/RxJava)

___
## Issue 8

#### <ins> Error Report from Infer

```txt
src/main/java/io/reactivex/rxjava3/internal/operators/flowable/FlowableCombineLatest.java:178: warning: THREAD_SAFETY_VIOLATION
  Unprotected write. Non-private method `void FlowableCombineLatest$CombineLatestCoordinator.cancel()` indirectly writes to field `v.produced` outside of synchronization.
 Reporting because another access to the same memory occurs on a background thread, although this access may not.
  176.               cancelled = true;
  177.               cancelAll();
  178. >             drain();
  179.           }
  180.
```

#### <ins> Revelant Methods

###### Method 1
```java
@Override
public void cancel() {
    cancelled = true;
    cancelAll();
    drain();                // ***** Method 2 *****
}
```

###### Method 2

```java
void drain() {
    if (getAndIncrement() != 0) {
        return;
    }

    if (outputFused) {
        drainOutput();
    } else {
        drainAsync();       // ***** Method 3 *****
    }
}
```

##### Method 3

```java
void drainAsync() {
    final Subscriber<? super R> a = downstream;
    final SpscLinkedArrayQueue<Object> q = queue;
    int missed = 1;
    for (;;) {
        long r = requested.get();
        long e = 0L;
        while (e != r) {
            ...
            ((CombineLatestInnerSubscriber<T>)v).requestOne();   // ***** Method 4 referenced here *****
            e++;
        }
            ...
    }
}
```

##### Method 4

```java
public void requestOne() {
    int p = produced + 1;
    if (p == limit) {
        produced = 0;
        get().request(p);
    } else {
        produced = p;
    }
}
```

#### <ins> Analysis


From the error report, we understand that the ```drain()``` method which is called by the ```cancel()```
method indirectly writes to the field ```v.produced```. We also understand that the memory location
occupied by the field ```produced``` can potentially be accessed by background threads concurrently.
Hence, we infer the potential of a data race, i.e. two or more concurrent accesses to this memory location where one of them is a write.

As seen in in the methods referenced above, we search several methods that are transitively called by method 1 to search for the potential of
a write to the field ```produced``` (eg. method 1 transitively calls method 3 because it calls method 2 which calls 3).
We find out that we write to the field ```produced``` in the method called ```requestOne()```, which means that two threads could potentially write to the field `produced` by calling method `requestOne()` concurrently, leading to a data race.

Infact, we also discover that if two threads call the `requestOne()` method concurrently, then not only can there be a data race in
the memory location occupied by the variable `produced`.

#### <ins> Solution


To solve this issue, we synchronize the call to the method ``requestOne()`` on the current instance (obtain lock on the current instance).


```java
synchronized public void requestOne() {
    int p = produced + 1;
    if (p == limit) {
        produced = 0;
        get().request(p);
    } else {
        produced = p;
    }
}
```

### Issue 9

#### <ins> Error Report from Infer

```txt
src/main/java/io/reactivex/rxjava3/internal/operators/flowable/FlowableBufferBoundary.java:302: warning: THREAD_SAFETY_VIOLATION
  Unprotected write. Non-private method `void FlowableBufferBoundary$BufferBoundarySubscriber.drain()` writes to field `this.emitted` outside of synchronization.
 Reporting because another access to the same memory occurs on a background thread, although this access may not.
  300.                   }
  301.
  302. >                 emitted = e;
  303.                   missed = addAndGet(-missed);
  304.                   if (missed == 0) {
```

#### <ins> Relevant Code

```java
void drain() {
    if (getAndIncrement() != 0) {
        return;
    }

    int missed = 1;
    long e = emitted;
    Subscriber<? super C> a = downstream;
    SpscLinkedArrayQueue<C> q = queue;

    for (;;) {
        long r = requested.get();

        while (e != r) {
            if (cancelled) {
                q.clear();
                return;
            }

            boolean d = done;
            if (d && errors.get() != null) {
                q.clear();
                errors.tryTerminateConsumer(a);
                return;
            }

            C v = q.poll();
            boolean empty = v == null;

            if (d && empty) {
                a.onComplete();
                return;
            }

            if (empty) {
                break;
            }

            a.onNext(v);
            e++;
        }

        if (e == r) {
            if (cancelled) {
                q.clear();
                return;
            }

            if (done) {
                if (errors.get() != null) {
                    q.clear();
                    errors.tryTerminateConsumer(a);
                    return;
                } else if (q.isEmpty()) {
                    a.onComplete();
                    return;
                }
            }
        }

        emitted = e;
        missed = addAndGet(-missed);
        if (missed == 0) {
            break;
        }
    }
}
```

#### <ins> Analysis

From the error report, we understand that the ```drain()``` method (note that this method is different from the one referenced in Issue 5)
writes to the feild ```emitted```. We also understand that the memory location
occupied by this feild can potentially be accessed by background threads concurrently.
Hence, we infer the potential of a data race, i.e. two or more concurrent accesses to this memory location where one of them is a write.

In particular, we argue that if two methods call this ```drain()``` method concurrently then the
feild ```emitted``` can be a victim to two or more concurrent accesses where atleast one of them is a
```write```.


#### <ins> Solution


To solve this issue, we synchronize the call to the method ``drain()`` on the current instance (obtain lock on the current instance).
Adding synchronized modifier reduced the number of "THREAD_SAFETY_VIOLATIONS" from 212 to 203.
Such a high decrease could have hapenned because there might be several variables like ```emitted``` that are victims of data races.


```java
synchronized void drain() {
    ...
}
```

### Issue 10

#### <ins> Error Report from Infer

```txt
src/main/java/io/reactivex/rxjava3/internal/operators/flowable/FlowableGroupJoin.java:209: warning: THREAD_SAFETY_VIOLATION
  Unprotected write. Non-private method `void FlowableGroupJoin$GroupJoinSubscription.drain()` indirectly writes to field `up.error` outside of synchronization.
 Reporting because another access to the same memory occurs on a background thread, although this access may not.
  207.                           q.clear();
  208.                           cancelAll();
  209. >                         errorAll(a);
  210.                           return;
  211.                       }
 ```

 #### <ins> Relevant Code

 ```java
void drain() {
    if (getAndIncrement() != 0) {
        return;
    }
    ...
            if (mode == LEFT_VALUE) {
                ...
                ex = error.get();
                if (ex != null) {
                    q.clear();
                    cancelAll();
                    errorAll(a);
                    return;
                }
                ...
            }
            ...
    }
}
```

In this example, Infer suggests that there is a potential data race problem with `errorAll(a)` on line 209. To see if this is indeed the case, we looked into the the `errorAll()` method.

#### <ins> Relevant Methods

```java
void errorAll(Subscriber<?> a) {
    Throwable ex = ExceptionHelper.terminate(error);

    for (UnicastProcessor<TRight> up : lefts.values()) {
        up.onError(ex);
    }

    lefts.clear();
    rights.clear();

    a.onError(ex);
}
```
#### <ins> Analysis

```errorAll(a)``` changes the variable ```a``` which is not thread safe.

#### <ins> Solution

```java
synchronized void errorAll(Subscriber<?> a) {
    Throwable ex = ExceptionHelper.terminate(error);

    for (UnicastProcessor<TRight> up : lefts.values()) {
        up.onError(ex);
    }

    lefts.clear();
    rights.clear();

    a.onError(ex);
}
```

### Issue 8

#### <ins> Error Report from Infer

``` txt
src/main/java/io/reactivex/rxjava3/internal/operators/observable/ObservableGroupJoin.java:205: warning: THREAD_SAFETY_VIOLATION
  Unprotected write. Non-private method `void ObservableGroupJoin$GroupJoinDisposable.drain()` indirectly writes to field `up.error` outside of synchronization.
 Reporting because another access to the same memory occurs on a background thread, although this access may not.
  203.                           q.clear();
  204.                           cancelAll();
  205. >                         errorAll(a);
  206.                           return;
  207.                       }
```

#### <ins> Relevant code

```java
void drain() {
    if (getAndIncrement() != 0) {
        return;
    }

    int missed = 1;
    SpscLinkedArrayQueue<Object> q = queue;
    Observer<? super R> a = downstream;

    for (;;) {
        for (;;) {
            ...

            Throwable ex = error.get();
            if (ex != null) {
                q.clear();
                cancelAll();
                errorAll(a);
                return;
            }
     ...
    }
}
 ```

 #### <ins> Analysis

 #### <ins> Solution

 ```java
synchronized void drain() {
    if (getAndIncrement() != 0) {
        return;
    }

    int missed = 1;
    SpscLinkedArrayQueue<Object> q = queue;
    Observer<? super R> a = downstream;

    for (;;) {
        for (;;) {
            ...

            Throwable ex = error.get();
            if (ex != null) {
                q.clear();
                cancelAll();
                errorAll(a);
                return;
            }
     ...
    }
}
 ```

#### <ins> Note
the number of thread safety violation reduces from 202 to 187.

## Conclusion
