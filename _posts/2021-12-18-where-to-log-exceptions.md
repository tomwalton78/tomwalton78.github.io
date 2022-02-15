---
layout: post
title: Where to Log Exceptions
tags: Java, Logging, Exceptions
---

Logging exceptions that occur is important for understanding why a program, or part of it, failed. However, bad 
logging practices can make this debugging task needlessly difficult, with either missing information, or the same 
information repeated many times. In this post I discuss how to avoid these problems, and so quickly get to the root 
cause of issues.


## Logging exceptions correctly

To begin with, let's recap how to properly log an exception in the first place, regardless of where in the 
application this is done.

```java
try {
  processFile(fileName);
} catch (IOException ex) {
  // Handle exception...
  LOGGER.error("Failed to process data source: {}", dataSource, ex);
}
```

The key thing to call out here is we pass the exception object itself to the logger, thereby allowing the logger to 
log the full stack trace of the exception, so no information is lost. Additionally, we provide extra information that is available to us in the current context, such as the data source name. This ensures we get a nice, useful stack trace like so:

```
2021-12-18T13:51:57.886 [main] ERROR loader.DataLoader - Failed to process data source: company A
java.io.FileNotFoundException: my_data.csv (No such file or directory)
	at java.io.FileInputStream.open0(Native Method) ~[?:?]
	at java.io.FileInputStream.open(FileInputStream.java:211) ~[?:?]
	at java.io.FileInputStream.<init>(FileInputStream.java:153) ~[?:?]
	at java.io.FileInputStream.<init>(FileInputStream.java:108) ~[?:?]
	at java.io.FileReader.<init>(FileReader.java:60) ~[?:?]
	at loader.DataLoader.processFile(DataLoader.java:68) ~[main/:?]
	at loader.DataLoader.main(DataLoader.java:23) [main/:?]
```

## Log-throw is an anti-pattern

Logging exceptions, and then continuing to throw that exception, or a new exception that wraps it, results in redundant 
log statements. These clutter 
up the log files, making it harder to understand what a program is doing, and pinpoint the root cause of an issue. 
The caller of a given function cannot know whether that function has already logged the exception or not, so it can 
either log it again (redundant), or even worse, not log it at all. Even if you are catching an exception and 
throwing a different one, you should pass the original exception into the constructor of the new one, so the stack 
trace is retained:

```java
try {
  doSomething();
} catch (IOException ex) {
  throw new MyCustomException(ex);
}
```

## Log exceptions where they are handled

If we shouldn't log exceptions and then throw them again, where do we log them? After all, we need to make they are
logged somewhere. Exceptions should be logged where they are handled. The handling code is the part of your program that
knows what the exception means in the context of the program (hence why it is handling it), and so can provide useful
additional context in the log statement. The exception must be logged here, or its information will be lost, as the
handling code does not rethrow it. Furthermore, here you have the maximum context: the full stack trace. Of course, you won't have access to the same
context that you would have had further down the call stack, and so it's those components' responsibility to include any
useful debugging information in the stack trace.

This practice ensures that there is only a single point in the code where the exception is logged, and logging can be
centralized, with only the higher level components needing to deal with logging.


## Exceptions to this rule

While the above advice works well most of the time, there are cases where you may want to log error information, and 
then continue to throw another exception. One example is if you're at the edge of the application layer, and you 
don't want to expose the full error information to the client 
  (e.g. API caller). In this case you may log the full stack trace, and then throw a more generic, high-level 
  exception that the client will see. This could be for security reasons, or simply readability.


## Conclusion

This post shows how to  avoid redundant log statements when logging exceptions in Java, although is more generally 
applicable to other languages that use exceptions. Following these practices should result in cleaner, easier to 
read application logs, making debugging issues less painful.



## References

- [Stack Overflow: Why is "log and throw" considered an anti-pattern?](https://stackoverflow.com/questions/6639963/why-is-log-and-throw-considered-an-anti-pattern)
- [Loggly: Logging](https://www.loggly.com/blog/logging-exceptions-in-java/)