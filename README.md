# FacebookInfer


### [Native-platform: Java bindings for native APIs](https://github.com/gradle/native-platform)

#### Issue:

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
    
```
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

