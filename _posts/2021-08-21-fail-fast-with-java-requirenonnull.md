---
layout: post
title: Fail-fast with Java's requireNonNull method
tags: Java
---

Null pointer exceptions can be a tricky bug because they can occur far from where the root cause of the problem is. A `null` value may be passed into one method, and that method may pass the argument on to another method, and so on. Then, when that value is finally used, an exception will be thrown. To debug the issue, you need to trace the source of the bad data through your program.

To avoid this problem, if we don't want a certain value to be `null`, we should explicitly say so, as soon as possible. That's where Java's [`Objects.requireNonNull()`](https://docs.oracle.com/javase/8/docs/api/java/util/Objects.html#requireNonNull-T-) method comes in. It's a concise way of throwing a 
`NullPointerException` if a given value is `null`:

```java
public void doSomething(String arg1, Integer arg2) {
	requireNonNull(arg1);
	requireNonNull(arg2);

	// Rest of method...
}
```

Advantages:
- Concise - a short one-liner
- Self-documenting - makes it clear that `null` arguments are not welcome
- [Fail-fast](https://en.wikipedia.org/wiki/Fail-fast) - don't allow `null` values to propagate through your program


## Where to use it?

In short, I'd recommend using `requireNonNull()` to validate the arguments in **all public methods**, including constructors. In order to properly decouple classes in your program, different classes should not make assumptions about how they will be called.

`requireNonNull()` can be particularly beneficial when used in a constructor, in combination with `final` fields. Consider the example below:
```java
public class ExampleClass {
	private final Integer myNumber;

	public ExampleClass(Integer number) {
		this.myNumber = Objects.requireNonNull(number);
	}
}
```

The key benefit here is that `myNumber` is now *guaranteed* to be non-null, and so any other code in this class can safely omit any further null checks. It's also nice that `requireNonNull()` returns the value passed to it, allowing our code to be even more concise.


## What's wrong with `if (arg == null)` checks?

You may be used to solving this problem by writing code such as:
```java
public class ExampleClass {
	private final Integer myNumber;

	public ExampleClass(Integer number) {
		if (number == null) {
			throw new NullPointerException();
		}
		this.myNumber = number;
	}
}
```

While this is perfectly valid, the key disadvantage is that it's more verbose: 4 lines vs. 1. This may seem insignificant, but it means extending the size of all of your public constructors by 3 lines, and other public methods by 2 lines, for each argument. This adds up, and not only will it make your code noticeably longer, but there will be a tendency to leave these checks out. By making these `null` checks as simple and easy as possible, they are much more likely to be done.


## Conclusion

While it would be nice if we didn't have to worry about littering our code with `null` checks, the reality is this is just something we have to live with in Java. `requireNonNull` makes makes this issue a little less painful, helping us to avoid wasted time debugging null pointer exceptions. Other languages solve this problem in more elegant ways, such as [Kotlin's choice](https://kotlinlang.org/docs/null-safety.html#safe-calls) to not allow `null` values to be stored in a variable by default, and requiring the programmer to explicitly mark a variable with `?` in order to allow `null` values.


## References

- [Oracle Java Docs: Objects requireNonNull](https://docs.oracle.com/javase/8/docs/api/java/util/Objects.html#requireNonNull-T-)
- [Wikipedia: Fail Fast](https://en.wikipedia.org/wiki/Fail-fast)
- [Kotlin Lang: Null Safety](https://kotlinlang.org/docs/null-safety.html#safe-calls)
- [Stack Overflow: Why should one use Objects.requireNonNull()?](https://stackoverflow.com/questions/45632920/why-should-one-use-objects-requirenonnull)
