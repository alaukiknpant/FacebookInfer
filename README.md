# FacebookInfer


### [Native-platform: Java bindings for native APIs](https://github.com/gradle/native-platform)

#### Issue 1:

```
native-platform/src/main/java/net/rubygrapefruit/platform/internal/TerminfoTerminal.java:203: warning: THREAD_SAFETY_VIOLATION
  Read/Write race. Non-private method `TerminalOutput TerminfoTerminal.bold()` indirectly reads without synchronization from `this.boldOn`. Potentially races with write in method `TerminfoTerminal.init()`.
 Reporting because a superclass `class net.rubygrapefruit.platform.terminal.TerminalOutput` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  201.       @Override
  202.       public TerminalOutput bold() {
  203. >         if (!supportsTextAttributes()) {
  204.               return this;
  205.           }
```

### Relevant Code

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

#### Relevant methods

```java
@Override
    public boolean supportsTextAttributes() {
        return boldOn != null && dim != null;
    }
```
    
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
#### Analysis

```supportsTextAttributes()``` reads whether ```boldOn``` is not null while ```write(boldOn)``` updates the value. So one thread potentially writes to the variable ```boldOn``` while another threads reads, leading to a data race.

#### Solution
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


#### Issue 2

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

#### Relevant Code

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

```java
@Override
    public boolean supportsTextAttributes() {
        return boldOn != null && dim != null;
    }
```

#### Analysis

From our first analysis, we suspect that the afformentioned method ```dim()``` also has a problem. The report points out that the method "`TerminalOutput TerminfoTerminal.dim()` indirectly reads without synchronization from `this.boldOn`". However, the problem with this particular method is that ```supportsTextAttributes()``` reads whether ```dim``` is not null while ```write(dim)``` updates the value. So one thread potentially writes to the variable ```dim``` while another threads reads, leading to a data race.

#### Solution
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

#### issue 7

#### report 

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
### relevant code

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

### relevant method

```java
@Override
public boolean supportsCursorVisibility() {
    return showCursor != null && hideCursor != null;
}
```

### analysis

### solution

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

#### Issue 8

#### report
```txt
native-platform/src/main/java/net/rubygrapefruit/platform/internal/TerminfoTerminal.java:164: warning: THREAD_SAFETY_VIOLATION
  Read/Write race. Non-private method `boolean TerminfoTerminal.supportsCursorVisibility()` reads without synchronization from `this.hideCursor`. Potentially races with write in method `TerminfoTerminal.init()`.
 Reporting because a superclass `class net.rubygrapefruit.platform.terminal.TerminalOutput` is annotated `@ThreadSafe`, so we assume that this method can run in parallel with other non-private methods in the class (including itself).
  162.       @Override
  163.       public boolean supportsCursorVisibility() {
  164. >         return showCursor != null && hideCursor != null;
  165.       }
```


### relevant code

```java
@Override
public boolean supportsCursorVisibility() {
    return showCursor != null && hideCursor != null;
}
```

#### relevant method

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
            hideCursor = TerminfoFunctions.hideCursor(result);
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

### Analysis

The ```init()``` method write to the variable hide-cursor. However, we also read hide-cursor if we call the ```supportsCursorVisibility()``` method. Since we need to acquire the lock ```lock``` for the ```init()``` method, we use the same lock for the ```supportsCursorVisibility()``` method.







#### solution

```java
    public boolean supportsCursorVisibility() {
        synchronized (lock) {
            return showCursor != null && hideCursor != null;
        }
    }
```


### [Rx Java](https://github.com/ReactiveX/RxJava)

#### Issue 3

#### Report

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

#### Revelant code

```java
        @Override
        public void cancel() {
            cancelled = true;
            cancelAll();
            drain();
        }
```


#### Relevant methods

```java
        void drain() {
            if (getAndIncrement() != 0) {
                return;
            }

            if (outputFused) {
                drainOutput();
            } else {
                drainAsync();
            }
        }
```

```java
        @SuppressWarnings("unchecked")
        void drainAsync() {
            final Subscriber<? super R> a = downstream;
            final SpscLinkedArrayQueue<Object> q = queue;
            int missed = 1;
            for (;;) {
                long r = requested.get();
                long e = 0L;
                while (e != r) {
                    ...
                    ((CombineLatestInnerSubscriber<T>)v).requestOne();
                    e++;
                }
                    ...
            }
        }
```

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


#### Analysis
The report claims that "non-private method `void FlowableCombineLatest$CombineLatestCoordinator.cancel()` indirectly writes to field `v.produced` outside of synchronization." If two threads access the `produced()` method concurrently, then the value of `produced` and `p` can be overwritten. 

#### Solution

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

#### issue 4

#### report

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

### relevant code

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
Note: Adding synchronized tag reduced the number of THREAD_SAFETY_VIOLATION from 212 to 203. This is because there are several variables (```e```, ```missed```, ```emitted```) are victims of thread violation. 

### Solution
```java
synchronized void drain() {
    ...
}
```


#### issue 5

#### report

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
 #### relevant code
 
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
#### analysis

```errorAll(a)``` changes the variable ```a``` which is not thread safe.

#### solution

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

#### Issue 6

#### Report
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

#### relevant code

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
 #### ANALYSIS
 
 #### Solution
 
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
 
note: the number of thread safety violation reduces from 202 to 187.
