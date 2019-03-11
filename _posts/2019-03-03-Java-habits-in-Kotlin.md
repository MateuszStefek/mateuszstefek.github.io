---
layout: post
title: Java habits you should break in Kotlin
comments: true
---

I've been programming in Kotlin for a few months now, after many years of Java.
I'm observing my style evolving a lot, even though the languages are not that different.

Here is a list of my observations and tips for someone who is, like me,
jumping into the Kotlin world from Java.

# 0. Don't use java.util.Optional
I promise, this is the last obvious entry in this list.
It's included to demonstrate a point that a feature glorified in one language
can be a clear antipattern in another.

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

# 2. Fluent interfaces are suspicious

My distaste for fluent interfaces probably started when I was working on a home-cooked implementation of Abstract DAO.
It had an interface which allowed to run queries with fluent method chaining:
```
List<Product> products = dao
    .where()
    .lessOrEqual("price", 4.0)
    .limit(50)
    .query()
```

It was pretty nice to use,
but imagine what I had to do when I needed to create a caching decorator for that DAO.

Since then, I make sure that my Java classes clearly separate the actual code from fluent wrappers on that code.
With Kotlin's extension methods and scope functions the fluent wrappers can disappear completely.

## 2.1. Fluent interface example: Jackson2ObjectMapperBuilder

The Spring Framework has a lot of fluent utilities.
Let's take a look at [Jackson2ObjectMapperBuilder](https://github.com/spring-projects/spring-framework/blob/master/spring-web/src/main/java/org/springframework/http/converter/json/Jackson2ObjectMapperBuilder.java).

If you are not a Jackson guru, you might not know how to configure unknown properties,
so Spring prepared a helper method for you:

```
public Jackson2ObjectMapperBuilder failOnUnknownProperties(boolean failOnUnknownProperties) {
        this.features.put(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, failOnUnknownProperties);
        return this;
}
```

which adds a feature to a collection and later, during `build()`,
one of overloaded `configure` methods of `ObjectMapper` is called:

```
private void configureFeature(ObjectMapper objectMapper, Object feature, boolean enabled) {
    if (feature instanceof JsonParser.Feature) {
        objectMapper.configure((JsonParser.Feature) feature, enabled);
    }
    else if (feature instanceof JsonGenerator.Feature) {
        objectMapper.configure((JsonGenerator.Feature) feature, enabled);
    }
    else if (feature instanceof SerializationFeature) {
        objectMapper.configure((SerializationFeature) feature, enabled);
    }
    else if (feature instanceof DeserializationFeature) {
        objectMapper.configure((DeserializationFeature) feature, enabled);
    }
    else if (feature instanceof MapperFeature) {
        objectMapper.configure((MapperFeature) feature, enabled);
    }
    else {
        throw new FatalBeanException("Unknown feature class: " + feature.getClass().getName());
    }
}
```

In Kotlin, to add a feature to an existing class, we can simply write an extension method:

```kotlin
fun ObjectMapper.failOnUnknownProperties(failOnUnknownProperties: Boolean) {
    configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, failOnUnknownProperties)
}
```

`Jackson2ObjectMapperBuilder` has elements of The Classic builder and there
are use cases where you need exactly that.
However, you usually use it for the fluent stuff.

Most of the utils from `Jackson2ObjectMapperBuilder` can be replaced by extension methods,
some of them simple as above, some requiring a couple lines of code.

# 3. Don't ponder on method order within a class - use nested functions

If your class has multiple public methods and each public method needs a few helper methods,
you probably organize them in this order, as this is what "Clean Code" suggests:
```
class MyBoringCRUDService {
  public methodA
  private helperA1
  private helperA2

  public methodB
  private helperB1
  private helperB2

  private sharedHelperMethod
}
```

However, there are people who prefer the "public methods first" strategy:
```
class MyBoringCRUDService {
  public methodA
  public methodB

  private helperA1
  private helperA2

  private helperB1
  private helperB2

  private commonHelperMethod
}
```

You can spend a lot of time on bikeshedding, discussing which order is better,
but in both cases, the helper methods feel like a mess after a few months of heavy development.

In Kotlin, I find nested functions quite useful in that regard. Here's an example:

```kotlin
class OrderEvaluationService(
        val priceService: PriceService,
        val discountService: DiscountService
) {
    fun calculateTotalPrice(orderLines: List<OrderLine>, customer: Customer): BigDecimal {

        fun orderLinePrice(orderLine: OrderLine): BigDecimal {
            val basePrice = priceService.currentPrice(orderLine.item) * orderLine.quantity
            val discount: BigDecimal? = discountService.discountGranted(customer, orderLine.item)

            return if (discount != null) {
                basePrice * (ONE - discount)
            } else {
                basePrice
            }
        }

        return orderLines.sumBy { orderLinePrice(it) }
    }
}
```

It's obvious what is the scope of the helper function,
so there is less chance it gets lost in a large class.

The inner function uses less parameters, as it can access all variables in the scope of the outer function.
In the example, I didn't have to pass the `customer` object (explicitly, as the compiler probably does it for me).

The price for this solution is that it violates two primal instincts of a good Java programmer:
the fear of long functions and the fear of many levels of indentation.
You should expect some complaints during code review and need to
argue for not including the length of the inner function when counting the lines of the outer functions.

This technique has some limits - it works best when the main function fits in a screen (vertically).

# 4. Standard Java functional interfaces

It seems natural to use Java's `Consumer` or `Supplier`:

```
fun doStuff(callback: Consumer<Item>) {

}
```

but in Kotlin these are not needed, as you can use first-class-citizen function types:

```kotlin
fun doStuff(callback: (Item) -> Any) {

}
```

These have the advantage that the generic types have the variance properly defined - input parameters are 'in',
result types are 'out'. You don't have to write anything to simulate `Consumer<? super Item>`.

# Conclusion
