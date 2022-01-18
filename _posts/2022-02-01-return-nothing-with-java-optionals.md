---
layout: post
title: Return nothing, with Java Optionals
tags: Java, Methods, Null, NullPointerException, Optional
---

In Java, it is common for a method to return `null`, or throw an exception, when it cannot return a value. In this post I aim to convince you to reach for the [Optional class](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Optional.html) instead. This will make your code more readable, reliable and succinct.


## Three ways to return nothing

In Java, there are three common ways to indicate that a method cannot return a value:

**1. Return `null`.** This may seem like the simplest option, however, it's not obvious that you need to write special code to handle the `null` return value. Not checking will likely result in a `NullPointerException` when the value is accessed, crashing your program. This may happen straight away, or much later on, in a distant class. The method's documentation could state that it returns `null` when it cannot find a value, but this is easy to miss.

**2. Throw an exception.** Exceptions should be reserved for exceptional circumstances. Perhaps your method should only be called when a value is present, and the absence of a value represents a failure of your program. Then an exception is reasonable. But if you're simply checking a database for a value, and it's not there, that's probably not exceptional. Throwing exceptions is expensive, and should only be done when a genuine fault has occurred.

**3. Return an empty Optional.** In the next section I will explain what this is, and why this is usually the right choice.


## What is an Optional?

An Optional is an immutable container which has two possible states: empty or full. When it is empty, there is no value contained within. When it is full, it contains a single object. Let's see an example.

```java
public Optional<String> fetchValue(int myId) {
    if (myDataStore.contains(myId) {
        return Optional.of(myDataStore.get(myId));
    }
    return Optional.empty();
}
```

When we return an Optional we are explicitly telling the user that a value might not be present. It's not possible for them to miss that part of the method's documentation, as the return type itself contains this information. It's also much cheaper than throwing an exception.


## Now that I have an Optional, what do I do with it?

Optionals are a bit like checked exceptions. With checked exceptions, the compiler requires you to add `throws ExceptionName` to the method declaration. If the user is determined to ignore it, they can swallow the exception, but they've had to explicitly choose to do so.

Similarly with Optionals, you should always handle the case where the value isn't present. If you don't, you will trigger a `NoSuchElementException` when accessing the value via the `get()` method. One way to do this is by first checking that the value is present:

```java
if (myOptional.isPresent()) {
    doSomething(myOptional.get());
}
```

However, there are more concise options, depending on what you're trying to achieve. Here's a handful selected from the [Java Optional documentation](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Optional.html):

- Returns value inside Optional, or returns default value if Optional is empty: `myOptional.orElse("default-value")`
- Returns value inside Optional, or throws given exception if Optional is empty: `myOptional.orElseThrow(() -> new NotFoundException())`
- Returns value inside Optional, or computes a value if Optional is empty: `myOptional.orElseGet(() -> getFromDB(myId))`
- Performs some action, but only if Optional is not empty: `myOptional.ifPresent(myValue -> doSomething(myValue))`



## When are Optionals not the best option?

If your method returns a collection or a map, there is no need to wrap it in an Optional. You already have a way to show that no value was found: just return an empty data structure.

Creating an Optional object will take longer than simply returning the unwrapped value. However, this is only relevant in performance-critical code, once you have actually measured a difference in speed between the two approaches.


## Conclusion

Optionals are an effective mechanism to communicate that a method might return nothing. They're clearer than returning `null`, and typically more appropriate than throwing an exception. The [Optional class](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Optional.html) has lots of useful methods for constructing Optionals and extracting the value within. There are even specific [Optional classes for primitive types](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/OptionalInt.html).




## References

- Effective Java, by Joshua Bloch
- [Java 11 Docs: java.util.Optional](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/Optional.html)
- [Java 11 Docs: java.util.OptionalInt](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/OptionalInt.html)