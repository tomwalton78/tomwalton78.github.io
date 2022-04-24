---
layout: post
title: Why you should use immutable objects
tags: Immutability, Immutable, Objects, Java, Object Oriented Programming
---

Using immutable objects in your codebase has numerous benefits, especially when it comes to [value objects](https://martinfowler.com/bliki/ValueObject.html). It makes your code easier to reason about, with less scope for side effects and subtle bugs. There are a few pitfalls to consider when designing immutable objects, and so using a 3rd party library to construct them can make your life much easier.

## What are immutable objects?

Immutable objects are objects whose state cannot be modified after they are first created. If you need to "modify" the object, you would have to create a new object. Importantly, this isn't just a practice that the developer follows when using the object, but the design of the object itself makes it impossible to modify the object's state. Below is a simple example in Java:

```java
final public class ImmutableRectangle() {
    final private int length;
    final private int width;

    public ImmutableRectangle(int length, int width) {
        this.length = length;
        this.width = width;
    }
    
    public int getLength() {
        return this.length;
    }
    
    public int getWidth() {
        return this.width;
    }
}
```

Key features that prevent the developer from modifying state post-creation:

- Class is marked as `final`. This avoids a subclass overriding methods so that they change the object's state.
- All fields are `final`, so values cannot be changed later on.
- All fields are `private`, so they cannot be access from outside of the class.
- No setter methods, only getters.
- Values returned are primitive types. This ensures a copy of the value is returned, rather than a reference to the memory location where the value is stored.

## Simplify your codebase

Some of the most confusing bugs occur when one piece of code interacts with another piece of unrelated code far away in the system. One way this can happen is by modifying an object that is shared by both classes. As the object is passed around it is easy to forget that the object is used in other places as well, and strange, unintended behaviour can occur.

The simple example below shows how this can go wrong. In practice, the codebase will be much larger, and the bug more subtle.

```java
public class ExampleClass() {

	public void exampleMethod() {
		Rectangle rectangle = new Rectangle(5, 10);
        
        Rectangle square = makeSquare(rectangle);
        
        int area = rectangle.computeArea();
        /*
        area equals 100, but we expected it to equal 50. What happened?
        The makeSquare method modified its arguments, a bad practice.
        */
	}

	private Rectangle makeSquare(Rectangle rectangle) {
		int longestSide = Math.max(rectangle.getLength(), rectangle.getWidth());
        if (longestSide == rectangle.getLength()) {
            rectangle.setWidth(longestSide);
        } else {
            rectangle.setLength(longestSide);
        }
        return rectangle;
	}

}
```

A common solution to the problem above is defensive copying of objects passed to functions. However, wouldn't it be nice if we were forced to make a copy of an object if we want to change it. We could never forget to copy it. When we pass an object to an unfamiliar function, we could be certain that the object won't be modified by that function. More generally, we can be confident about the state of our program, and can more easily reason about what the code should do. Different parts of the codebase are more effectively isolated from one another. Poor quality code in another part of the system cannot cause bugs within your own code so easily. This is what immutable objects give us.

## Thread safety

Sharing mutable objects between threads adds yet another layer of complexity. We have to worry about which thread is allowed to modify the object, using various synchronisation mechanisms to control access. Getting this wrong can lead to subtle bugs which are hard to reproduce, with your program producing different results from run to run. You can make all these problems go away by using immutable objects. It doesn't matter if multiple threads are sharing the same object, because they cannot modify that object.

Admittedly, this isn't always possible. Sometimes threads need to communicate with each other via the state of an object, or a 3rd party library doesn't use immutable objects. The important thing is to use immutable objects wherever you can. This lets you focus on the few remaining cases where inter-thread communication is required, making it more likely you get these cases right. Or at least know where to look if things go wrong.

## Avoid invalid state

When an object is created it should be in a valid state, as ensured by validation applied during construction. However, if the object is mutable there is a chance that it reaches an invalid state later on. Perhaps another piece of code directly sets a property of the object to an invalid value. Or maybe you miss a key piece of validation in one of the object's methods; an edge case that slipped through the cracks. Now the object is in an invalid state and your program may not behave as intended, leading to bugs, crashes or a poor user experience. Using immutable objects means you only have to worry about validating the object's state during its creation. After creation, you can be confident it will always be in a valid state.

## Limitations

While I would advocate for using immutable object by default, there are cases where they don't make sense. For some applications the performance cost of creating copies of large objects to change a single field can be prohibitive. However, ensure there is a measurable performance benefit that's worth the increased code complexity. One example could be changing the position of a video game character, where it's impractical to create a new, complex character object every time the character moves. Another would be modifying a large, in-memory dataset.

## Use a 3rd party library

While it is possible to create your own immutable objects, as per the example earlier on, there are several pitfalls. It's also quite verbose to write so much code for all of your objects, and easy to miss an edge case that permits mutability. Using a 3rd party library, such as [Java's Immutables](https://immutables.github.io/immutable.html), means you can use simple annotations to define your value objects, writing as little code as possible. Furthermore, you get other useful features for free. These include [builder methods](https://refactoring.guru/design-patterns/builder), equality comparison and the ability to copy an existing object but with changes to a few fields.

## Conclusion

In many situations using immutable objects is a no-brainer. They provide numerous benefits which help you avoid bugs and more easily reason about what your code is doing. Furthermore, if using a 3rd party library, it can be easier than creating mutable objects by hand. Of course, there are cases where mutability may be required, such as when performance is critical. But the key message I'd like to get across is that you should reach for immutable objects by default, and think twice when using mutable objects. Not the other way around.

## References

- [Martin Fowler: Value Objects](https://martinfowler.com/bliki/ValueObject.html)
- [Java Immutables library](https://immutables.github.io/immutable.html)
- [Java Docs: A Strategy for Defining Immutable Objects](https://docs.oracle.com/javase/tutorial/essential/concurrency/imstrat.html)
- [Hackernoon: 5 Benefits of Immutable Objects Worth Considering for Your Next Project](https://hackernoon.com/5-benefits-of-immutable-objects-worth-considering-for-your-next-project-f98e7e85b6ac)
- [Refactoring Guru: Builder Design Pattern](https://refactoring.guru/design-patterns/builder)