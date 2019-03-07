---
layout: post
title: Java habits you should break in Kotlin
---

# Lessons from the history: Classic OO patterns partially obsoleted by Java 8

About 10 years ago, a typical Java repository was full of classes with patterns in their names:
- Factory
- Command
- Chain (of command)
- Strategy
- Adapter
- Decorator

This trend resulted in code exemplified by this piece of artwork:
```java
public class ObjectFactoryCreatingFactoryBean extends AbstractFactoryBean<ObjectFactory<Object>> {
    @Nullable
    private String targetBeanName;
```
This class is not a joke, but a core element of the Spring framework.

But in 2014, Java 8 was released and since then,
we have been observing less and less
`CommandFactoryStrategyAdapter` nonsense in our repositories.
It still happens, but this style isn't that trendy anymore.

Basically, the functional elements of Java 8 demonstrated
what functional programming advocates had been saying for years.
Many OO design patterns are just workarounds for inconveniences of C++ or Java:

Factory is just a function returning an object:
```java
var fruitFactory = () -> new Orange()
```

Command is just a partial function application:
```
command = receiver -> receiver.method(parameter)
```

Decorator is often a simple function composition:
```
enhanced = x -> enhance(delegate.f())
```

Adapter might be trivial:
```
adapter = (param1, param2) -> adaptee.someMethod(param2, param1, null)
```

Chain Of Responsibility can be replaced by folding:
```
compositeHandler = input -> handlers.foldRight(input, (h, acc) -> h.handle(acc))
```

What I'm trying to say is, when you switch a programming language,
you need to check your habits and patterns to evaluate
if they are still relevant.
Often the old style is not only old-school, but also harmful.

# Kotlin
I've been programming in Kotlin for a few months now, after many years of Java.
I'm observing my style evolving a lot, even though the languages are not that different.

Here is a list of my observations and tips for someone who is, like me, jumping into the Kotlin world.

# 0. Don't use java.util.Optional
I promise, this is the last obvious entry.

This is how Java coders protect themselves from the dreaded `NullPointerException`:
```java
/** @author me */
public Set<String> allPrivileges() {
    return ofNullable(groups).orElse(emptySet())
            .stream()
            .flatMap(g -> ofNullable(g.getPrivileges()).orElse(emptySet()).stream())
            .map(Privilege::getKey)
            .collect(toSet());
}
```
Kotlin reaction:

<img src="/images/laughing.gif"/>

These poor people,
they are traumatized by `NullPointerException` so much,
that when they want to read a variable,
they wrap the value in an object,
call a static function in a synthetic class,
and then unwrap the result of that function.
Yet, it still doesn't work:
```java
// perfectly valid Java code
List<Optional<Product>> list = new ArrayList<>();
list.add(null);
```

More seriously, if you find yourself using `Optional` in Kotlin, for example like this:
```kotlin
interface ProductRepository {
  fun findById(id: Long): Optional<Product>
}
```
then there's definitely something wrong and you should go back to the tutorials.

# 1. Fluent builder considered harmful

The Builder pattern comes in two flavours:

The Classic:
```
CarBuilder builder = new CarBuilder();
builder.setDoors(4);
builder.setHorsePower(100);
Car car = builder.build();
```

The Fluent:
```
Car car = new CarBuilder()
  .withDoors(4)
  .withHorsePower(100)
  .build();
```

The builder pattern allows you to hold a reference to a partially constructed object.
The decisions about different properties can be implemented in different methods and
the final act of building an object can be deferred somewhere else.

The Fluent is a variation on The Classic,
making it is easier to chain setter calls, hence "fluent" in the name.
It is so popular, that many programmers think that the method chaining ability
is the defining feature of the Builder pattern.

The Classic and The Fluent solve different problems and should be considered distinct patterns.

In Kotlin, **The Fluent is no longer needed** and I would consider it an antipattern.

What inconveniences of Java does The Fluent solve?

## 1.1 Explicit parameter naming of Fluent Builder

If your object has many properties and/or multiple constructor arguments,
we need some way to explicitly name them in the code.
```
Car car1 = new Car(4, 100);
Car car2 = new CarBuilder()
  .withDoors(4)
  .withHorsePower(100)
  .build();
```
Arguably, the construction of car2 is easier to read.

Kotlin's solution to the explicit naming problem are
[Named Arguments](http://kotlinlang.org/docs/reference/functions.html#named-arguments):
```kotlin
val car = Car(doors = 4, horsePower = 4)
```
This has a nice property that when someone adds a new field to the class,
this code will no longer compile.
Most implementations of The Fluent will not complain.

## 1.2 Method chaining of Fluent Builder

Method chaining is convenient, when you need to create an object inside a short expression:

```
assertThat(service.estimatePrice(exampleCarBuilder().withDoors(4).build()))
  .returns(whatever);
```

In Kotlin, if you have an expression you want to keep in a single statement, you
can use one of the scope functions:
[let](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/let.html),
[apply](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html),
[run](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/run.html),
[with](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/with.html),
[also](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/also.html)

```kotlin
val car = classicBuilder().run {
    doors = 4
    horsePower = 100
    build()
}

val car = classicBuilder().also {
    doors = 4
    horsePower = 100
}.build()

val car = with(classicBuilder()) {
    doors = 4
    horsePower = 100
    build()
}
```

If you are new to Kotlin, this code might look ugly to you,
but believe me, this is even described as a common
[idiom](https://kotlinlang.org/docs/reference/idioms.html#calling-multiple-methods-on-an-object-instance-with).
There's no performance penalty for the lambdas - the scope functions are inline.

Here's another example from [Kotlin Expertise Blog](https://kotlinexpertise.com/coping-with-kotlins-scope-functions/),
proving that the good old `StringBuilder` hadn't had to be fluent:
```kotlin
val s: String = with(StringBuilder("init")) {
        append("some")
        append("thing")
        println("current value: $this")
        toString()
    }
```


By the way, many Kotlin DSLs use a trick which wraps
`classicBuilder().also { ... }.build()` in a helper function like `newCar` below,
so that you can use the builder pattern in a code which looks very declarative:

```
val car = newCar {
    doors = 4
    horsePower = 100
}
```
Use this technique with judgement. The amount of magic can be overwhelming.

# 2. Fluent interfaces in general