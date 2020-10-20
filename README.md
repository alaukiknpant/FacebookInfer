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

Error 1: From our first analysis, we suspect that the afformentioned method ```dim()``` also has a problem. The report points out that the method "`TerminalOutput TerminfoTerminal.dim()` indirectly reads without synchronization from `this.boldOn`". However, the problem with this particular method is that ```supportsTextAttributes()``` reads whether ```dim``` is not null while ```write(dim)``` updates the value. So one thread potentially writes to the variable ```dim``` while another threads reads, leading to a data race.

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
