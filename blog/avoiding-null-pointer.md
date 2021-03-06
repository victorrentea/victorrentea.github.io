---
tags: best practices, lombok, clean code 
---
# Avoiding NullPointerException
The terrible `NullPointerException` (NPE in short) is the most frequent Java exception occurring in production, acording to [a 2016 study](https://www.overops.com/blog/the-top-10-exceptions-types-in-production-java-applications-based-on-1b-events/). In this article we’ll explore the main techniques to fight it: the self-validating model and the `Optional` wrapper.

## Self-Validating Model    

Imagine a business rule: every Customer has to have a birth date set. There are a number of ways to implement this constraint: validating the user data on the create and update use-cases, enforcing it via `NOT NULL` database constraint and/or implementing the null-check right in the constructor of the Customer entity. In this article we'll explore the last one.

Here are the 3 most used forms to null-check in constructor today:
```java
public Customer(@NonNull Date birthDate) { // 3
  if (birthDate == null) { // 1
     throw new IllegalArgumentException();
  }
  this.birthDate = Objects.requireNonNull(birthDate); // 2
}
```

The code above contains 3 _alternative_ ways to do the same thing, any single one is of course enough:
1. Classic `if` check
2. One-liner check using Java 8 `java.util.Objects` - most widely used in projects under development today
3. Lombok `@NonNull` causing an `if` check to be added to the generated bytecode.

If there is a setter for birth date, the check will move there, leaving the constructor to call that setter. 

Enforcing the null-check in the constructor of your data objects has obvious advantages: no one could ever forget to do it. However, frameworks writing directly to the fields of the instance via reflection may bypass this check. Hibernate does this by default, so my advice is to also mark the corresponding required columns as `NOT NULL` in the database to make sure the data comes back consistent. Unfortunately, the problem can get more complicated in legacy systems that have to tolerate 'incorrect historical data’.

> **Trick**: Hibernate requires a no-arg constructor on every persistent entity, but that constructor can be marked as `protected` to hide it from developers. 

The moment all your data objects enforce the validity of their state internally, you do get a better night sleep, but there's also a price to pay: creating dummy incomplete instances in tests becomes impossible. The typical tradeoff is relying more on [Object Mother](https://martinfowler.com/bliki/ObjectMother.html)s for building valid test objects.

So, in general, whenever a null represents a data inconsistency case, throw exception as early as possible.

But what if that `null` is indeed a valid value? For example, imagine our Customer might not have a Member Card because she didn't yet create one or maybe she didn't want to sign up for a member card. We'll discuss this case in the following section.

## Getters returning Optional
> **Best-practice**: Since Java 8, whenever a function needs to return `null`, it should declare to return `Optional` instead

Developers rapidly adopted this practice for functions computing a value or fetching remote data. Unfortunately, that didn't help with the main source of NPEs: our entity model.

> A getter for a field which may be `null` should return `Optional`.

Assuming we're talking about an Entity mapped to a relational database, then if you didn't enforce `NOT NULL` on the corresponding column, the getter for that field should return `Optional`. For non-persistent data objects or NoSQL datastores that don't offer `null` protection, the previous section provides ideas on how enforce null-checks programmatically in the entity code. 

This change might seem frightening at first because we’re touching the 'sacred' getter we are all so familiar with. Indeed, changing a getter in a large codebase may impact up to dozens of places, but to ease the transition you could use the following sequence of safe steps: 

1. Create a second getter returning `Optional`:
    ```java
    public Optional<String> getMemberCardOpt() {
      return Optional.ofNullable(memberCard);
    }
    ```

2. Change the original getter to delegate to the new one:
    ```java
    public String getMemberCard() {
      return getMemberCardOpt().orElse(null);
    }
    ```

3. Make sure all the Java projects using the owner class are loaded in your workspace.
3. **Inline** the original getter everywhere. Everyone will end up calling `getMemberCardOpt()`.
4. **Rename** the getter to the default name (removing the `Opt` suffix)

You're done, with zero compilation failures. After you do this, everyone previously calling the getter will now do `getMemberCard().orElse(null);`. In some cases this might be the right thing to do, as in: `dto.phone=customer.getPhone().orElse(null);`

But let's suppose you wanted to use a property of the MemberCard, and you were careful to check for `null`:
```java
if (customer.getMemeberCard() != null) { // Line X
    applyDiscount(order, customer.getMemeberCard().getPoints());
}
```
After applying the refactoring steps above, the code gets refactored to 
```java
if (customer.getMemeberCard().orElse(null) != null) { // Line X
    applyDiscount(order, customer.getMemeberCard().orElse(null).getPoints());
}
```
The `if` condition can be simplified by using `.isPresent()` and the second line by using `.get()`. Then one could even shorten the code to a single line:
```java
customer.getMemberCard().ifPresent(card -> applyDiscount(order, card.getPoints()));
```
This means that you still need to go through all the places the getter is called to *improve* the code as we saw above. Furthermore, I bet that in large codebases you'll also discover places where the null-check (// Line X) was forgotten because the developer was tired/careless/rushing back then:
```java
applyDiscount(order, customer.getMemeberCard().orElse(null).getPoints());
```
**Tip**: IntelliJ will hint you about the possible NPE in this case, so make sure the inspection 'Constant conditions and exceptions' is turned on. 

Signaling the caller at compile-time that there might be nothing returned to her is an extremely powerful technique that can defeat the most frequent bug in Java applications. Most NPEs occur in large projects mainly because developers aren’t fully aware some parts of the data might be `null`. It happened on our project: we discovered dozens of `NullPointerExcepton`s just waiting to happen when we moved to `Optional` getters.

#### Frameworks and Optional getters 

Would frameworks allow getters to return Optional?

First of all, to make it clear, we only changed the return type of the getter. The setter and the field type kept using the raw type (not `Optional`). 

All modern object-mapper frameworks (eg Hibernate, Mongo, Cassandra, Jackson, JAXB ...) can be instructed to read from private fields via reflection (Hibernated does it by default). So really, the frameworks don’t care about your getters. 

#### When is Optional overkill?

You should consider making null-safe the objects you write logic on: Entities and Value Objects. As I explained in my [Clean Architecture talk](https://www.youtube.com/watch?v=tMHO7_RLxgQ&list=PLggcOULvfLL_MfFS_O0MKQ5W_6oWWbIw5&index=3), you should avoid writing logic on API data objects (aka Data Transfer Objects). Since no logic uses them, null-protection is overkill. 

> Use `Optional` in your Domain Model not in your DTO/API Model.

## Pre-instantiate sub-structures

Never do this:
```java
private List<String> labels;
``` 
> Always initialize the collection fields with an empty one!

```java
private List<String> labels = new ArrayList<>();
```
Those few bytes allocated beforehand almost never matter. On the other hand, the risk for doing `.add` on a `null` list is just to dangerous. In some other cases you might want to make the field `final` and take it via the constructor. Never leave collections references to have a `null` value.

Many teams choose to decompose larger entities into smaller parts. When those parts are mutable, make sure you instantiate the parts in the parent entity:
```java
private BillingInfo billingInfo = new BillingInfo();
``` 
This would allow the users of your model to do `e.getBillingInfo().setCity(city);` without worrying about nulls.

## Conclusion
You should consider upgrading your entity model to either reject a `null` via self-validation or present the nullable field via a getter that returns `Optional`. The effort of changing the getters of the core entities in your app is considerable, but along the way, you may find many dormant NPEs. 

Lastly, always instantiate embedded collections or composite structures.