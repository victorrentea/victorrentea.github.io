# Definitive Guide to Working with Exceptions in J ava

_Victor is a Java Champion, former Lead Architect at IBM, currently an Independent Trainer, having taught thousands of developers in dozens of companies over 8 years of training activity. He is the founder of one of the largest [developer community](https://www.meetup.com/bucharest-software-craftsmanship-community/) in Romania and one of the highest-rated European trainer on Clean Code, Refactoring and Unit Testing. More about him on [victorrentea.ro](http://victorrentea.ro/)_

_All the code of this article is available at [https://github.com/victorrentea/exceptions-guide/tree/article](https://github.com/victorrentea/exceptions-guide/tree/article)_

## Table of Contents
1. [A bit of history](#a-bit-of-history)
2. [Why you should only use Runtime Exceptions](#why-you-should-only-use-runtime-exceptions)
3. [How to handle checked exceptions](#how-to-handle-checked-exceptions)
4. [Presenting errors to users](#presenting-errors-to-users) 
5. [Handling unexpected Exceptions](#handling-unexpected-exceptions)
6. [Avoiding NullPointerException](#avoiding-nullpointerexception)
7.  [Checked Exceptions and Streams](#checked-exceptions-and-streams)
8. [The Try Monad](#the-try-monad)
9. [Left-over tips](#left-over-tips)

## A Bit of History

At first, there were no exceptions. 40 years ago, when a C function wanted to signal an abnormal termination, it would return a -1 or `NULL` value. But the caller might forget to check the returned value and the error would go unnoticed. In 1985, C++ tried to fix that by introducing exceptions that propagate down the call stack and, unless caught, terminate the program execution. Developers then faced a new challenge: how would the caller know what exceptions could be thrown by a function?

10 years later, Java was invented. Since the vast majority of developers at that time were using C/C++, one goal of Java was to attract them with features like Garbage Collection, safe references (vs pointers), smooth build process, unified source code (vs .h/.cpp), portability, and others. At the same time, Gosling tried to fix the problem of mysterious exceptions by forcing developers to be aware of _checked exceptions_ thrown by the functions they called. Of course, some errors couldn’t be foreseen (like `ArrayOutOfBoundsException` or `NullPointerException`), so these were left invisible _runtime exceptions_. If you did your homework, you know that checked exceptions are supposed to be thrown for ‘recoverable errors’. Only then is it worth the pain of forcing your caller to try-catch or throws. 

But all this was happening in The Age of Libraries: in the ‘90s developers were starting to realize the importance of reuse; libraries and frameworks were being created everywhere. When writing a library, it’s very tempting to consider that your callers might recover from a wide range of errors, especially if you don’t have a very clear picture of how your library will be used by its clients. Thus, the urge towards checked exceptions: let’s notify our callers of everything,.. they might be able to recover.

That age is gone.

In the applications we write today, we rarely recover from exceptions. Instead, we terminate the execution of that use-case ASAP. In a web application, you’d return a 400/500 status code. When handling messages, you’d reject the message. And in a batch, you’d skip or report that line. We almost never recover from an error anymore. (Disclaimer: This article is about building custom applications, like almost all of my client companies are doing.)

But 25 years later, checked exceptions are still here.

## Why you should only use Runtime Exceptions
_If that’s obvious to you, please skip to the next section_

Imagine the following simplified configuration loader 1xxxxsuperscript: 

    public static Date getLastPromoDate() throws ParseException, IOException {
      Properties props = new Properties();
      try (Reader reader = new FileReader("config.properties")) {
         props.load(reader);
      }
      SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd");
      return format.parse(props.getProperty("last.promo.date"));
    }

Used by the following business logic:

    public void applyDiscount(Order order, Customer customer) {
      if (order.getOfferDate().before(DateUtils.addDays(new Date(), 1))) {
         System.out.println("APPLYING DISCOUNT");
         order.setPrice(order.getPrice() * (100 - 2 * customer.getMemberCard().getFidelityDiscount()) / 100);
      } else {
         System.out.println("NO DISCOUNT");
      }
    }

Then, someday after Black Friday, he has to replace _tomorrow_ with a fixed date read from the application configuration. To do that, he has to call `Config.getLastPromoDate()`. Unfortunately, that method throws a `ParseException` back at him because its author, Alan, imagines you might be able to recover (and frankly, he also doesn’t care/know how to handle it). Being a checked exception, Bill decides to catch it and sends an email to ask his colleagues how to handle that case, leaving a careful `// TODO` in the catch block.

    public void applyDiscount(Order order, Customer customer) {
      try { 
        if (order.getOfferDate().before(Config.getLastPromoDate())) {
           System.out.println("APPLYING DISCOUNT");
           order.setPrice(order.getPrice() * (100 - 2 * customer.getMemberCard().getFidelityDiscount()) / 100);
        } else {
           System.out.println("NO DISCOUNT");
        }
      } catch (ParseException e) {
        // TODO
      }
    }

Of course, there are more important tasks in a project than error handling, so the email is forgotten. 

Two months later, Alan needs to change `Config.getLastPromoDate()` to read the end of promotions date from a properties file. After updating the code accordingly, he throws the new `IOException` to its callers, as usual.
**Note**: config files shouldn’t be read repeatedly, but cached somehow; this is just a simplified example to illustrate some common exceptions occurring.

Unfortunately, now the code doesn’t compile anymore. Annoyed that he has to handle exceptions, Alan replaces in Bill’s method `catch (ParseException e)` with `catch (Exception e)`, since that’s the common supertype of both `ParseException` and `IOException`. Leaving the `// TODO` in there he goes to ask Bill what’s to be done there. A fatal mistake.

Soon, they both forget about the issue. Because it’s ... error handling.

And the code goes to production.

Two weeks later, a strange bug is reported: for some orders, when the `applyDiscount()` method was called, it abruptly terminates without printing neither `APPLYING DISCOUNT`, nor `NO DISCOUNT`. Here’s the code again - try to guess the cause of the error:


    public void applyDiscount(Order order, Customer customer) {
      try { 
        if (order.getOfferDate().before(Config.getLastPromoDate())) {
           System.out.println("APPLYING DISCOUNT");
           order.setPrice(order.getPrice() * (100 - 2 * customer.getMemberCard().getFidelityDiscount()) / 100);
        } else {
           System.out.println("NO DISCOUNT");
        }
      } catch (Exception e) {
        // TODO
      }
    }

You find the bug at 1:00 AM, after 3 hours of remote debugging using breakpoints everywhere to trace what’s going on, because real production systems don’t look as simple as the code snippet above. What happened? A `NullPointerException` was occurring in the `if` condition for the Orders with a `null` offer date. Indeed, the data was corrupted (every Order has to have an offer date set according to business), but there was nothing in the logs to help debug the error. 

Never do this if you want to keep your job.

This is the infamous **Diaper Anti-Pattern**: catching all exceptions and just “swallowing” them. Run the `InProduction` class from [this commit](https://github.com/victorrentea/exceptions-guide/commit/2f9b92a2e1ae9ab35bc08cadf9cd10b5e12c46ff) to experience the swallowed exception.

You’d never do that. I know. Plus, instead of `Exception` you would have put `ParseException | IOException`. Furthermore, even the default catch block auto-generated by Eclipse or IntelliJ would `.printStackTrace()`, so you’re safe, right? 

    C:\workspace\openjdk-15_windows-x64_bin\jdk-15\bin\java.exe ... victor.training.exceptions.InProduction
    java.lang.NullPointerException: Cannot invoke "java.util.Date.before(java.util.Date)" because the return value of "victor.training.exceptions.model.Order.getOfferDate()" is null
        at victor.training.exceptions.Biz.applyDiscount(Biz.java:20)
        at victor.training.exceptions.InProduction.main(InProduction.java:13)

For a moment, let’s enjoy the nice message reported by Java 15 for `NullPointerException`s. You could be an absolute rookie and still get where the problem is. Up to the last LTS (Java 11), the NPE had a very dull message: 

    C:\Users\victo\.jdks\corretto-11.0.7\bin\java.exe ... victor.training.exceptions.InProduction
    java.lang.NullPointerException
        at victor.training.exceptions.Biz.applyDiscount(Biz.java:20)
        at victor.training.exceptions.InProduction.main(InProduction.java:13)

:ok_hand:

Oh, and here’s a fun fact for you: `.printStackTrace()` prints the exception to `System.err`, not `System.out`. By default most frameworks (Spring Boot included) don't capture `System.err` to the logfile unless you [take explicit steps](https://stackoverflow.com/questions/11187461/redirect-system-out-and-system-err-to-slf4j) :smirk: So despite that generated `e.printStackTrace()`, you will probably NOT see anything in your log file.


## How to handle checked exceptions

By rethrowing them wrapped in a `new RuntimeException(checkedException)`.

 > **Best Practice**: Always remember to set the _cause_ of the new exception to the original exception, whenever there is one. Don’t _decapitate_ your exceptions. 

But where should we wrap our two exceptions: inside `Config.getLastPromoDate()` or in `Biz.applyDiscount()`?

To understand the right answer, we need to talk about abstractions. An abstraction is a simplification of reality, hiding (complex) implementation details. From this design perspective, a method like `getLastPromoDate()` declaring `throws IOException`, is an _abstraction leak_ as it reveals the implementation detail that it’s working with files, or I/O. That’s why, `getLastPromoDate()` should NOT declare any checked exceptions but instead wrap the checked exceptions in a `RuntimeException` and throw that instead. Heres the new code ([commit](https://github.com/victorrentea/exceptions-guide/commit/99288a3fd8b34e64323855f05d6377dea495c1b8)):

    public static Date getLastPromoDate() {
      try{
         Properties props = new Properties();
         try (FileReader reader = new FileReader("config.properties")) {
            props.load(reader);
         }
         SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd");
         return format.parse(props.getProperty("last.promo.date"));
      } catch (ParseException | IOException e) {
         throw new RuntimeException(e);
      }
    }

> **Best Practice**: Avoid catching Exception unless in a top-level global exception handler (read below); instead, prefer E1\|E2 style (as in the code above).

There are mainly two reasons why we only allow RuntimeExceptions through our code today: 
1. Avoid abstraction leak and 
2. Lower chance of Diaper Anti-Pattern, since developers aren’t forced to `try/catch/throws` them.

Many years after this became the universally accepted best practice, many developers were still suffering from [PTSD](https://en.wikipedia.org/wiki/Post-traumatic_stress_disorder) after many debugging nights hunting for swallowed exceptions.

That’s why another bad habit developed, **The Log-Rethrow Anti-Pattern**:

    } catch (ParseException | IOException e) {
      log.error(e.getMessage(), e);
      throw new RuntimeException(e);
    }

Printing and throwing back the exception will cause the same error to be logged multiple times, turning your log into a frustrating, unreadable pile of garbage. Try it: ([commit](https://github.com/victorrentea/exceptions-guide/commit/dd66304cdd7e808e7e277efb2e02876e2735819f)). All because the developer was afraid that the exception might slip out without being logged anywhere.

> **Best Practice**: always make sure any unhandled exceptions are logged with their stack-traces in the log file of your application, whatever the thread they might occur in.

The real solution to this is to have a global exception handler in place that safely logs every unhandled exception. We will discuss how to implement such a handler for Spring REST services, in the following section below. 

## Presenting errors to users
Before we even start, let’s make it clear: 

> **NEVER expose stack traces in your API** responses.

That’s not only ugly but dangerous from a security point of view. Based on the line numbers in that stack trace, an attacker might infer the key libraries and versions that you’re using, and attack you by exploiting their known weaknesses.

Then what to show our users?

The simplest idea is to display the exception message string to the users. For simple applications, this might work out, but in medium larger applications two issues arise:

First, you might want to unit-test that a certain exception was thrown. Asserting message strings will lead to brittle tests, especially when you start adding user input to the messages of thrown exceptions.

Secondly, one might argue that you’re violating [MVC](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller), or at least, that you’re mixing diverging concerns in the same code: formatting the user-visible message AND detecting the business error condition.

> **Best Practice**: Use the exception message to report technical details about the error, key parameters, ids, current record index, anything that might be useful for developers investigating the exception. 

You might even catch an exception in a method X and rethrow it back wrapped in an exception with a useful message capturing key debug info from method X. That’s the **Catch-rethrow-with-debug-info Pattern**:

    throw new RuntimeException("For id: " + keyDebugId, e);

The final stack trace will contain all the useful information, making breakpoint-debugging it useless. Perfect!

So don’t use exception messages to hold the user error messages, but keep them for messages for developers’ eyes.

It might be tempting to start creating multiple exception subtypes, one per each business error, and distinguish between errors based on the exception type. Although fun at first, it will lead us to hundreds of exception types in typical applications. Unreasonable.

> **Best Practice**: Only create a new exception type E1 if you need a catch(E1) to selectively handle that particular exception type to work-around or recover from it (but in practice, you’ll seldom need that).

So the best solution to distinguish between your non-recoverable error conditions is based on a dedicated error code field, not the exception message, nor the exception type. 

Should that error code be an int? If it were, we would need the _Exception Manual_ at hand every time we walk through the code. Horrid scenario! But wait! Every time the range of values is finite, pre-determined, we should always consider using an enum. Let’s give it a first try:

    public class MyException extends RuntimeException {
      public enum ErrorCode {
         GENERAL,
         BAD_CONFIG;
      }

      private final ErrorCode code;

      public MyException(ErrorCode code, Throwable cause) {
         super(cause);
         this.code = code;
      }
 
      public ErrorCode getCode() {
         return code;
      }
    }

And we’ll replace throwing an opaque RuntimeException with `throw new MyException(ErrorCode.BAD_CONFIG, e);` in the catch clause in `getLastPromoDate()`. 

Unless required to report an error condition, always consider displaying an opaque general-purpose error message, like “Internal Server Error, please check the logs”. Realistically weight whether the ~~developer~~ user can do anything about that error condition. If not, do not bother distinguishing between error causes.

> **Best Practice**: Avoid reporting errors to users unless they can do something to fix it.

Then let’s write a GlobalExceptionHandler to catch it and translate it into a friendly user message, using Spring Boot (all other major web frameworks today offer similar functionality):

    @RestControllerAdvice
    public class GlobalExceptionHandler {
      private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

      private final MessageSource messageSource;
   
      public GlobalExceptionHandler(MessageSource messageSource) {
         this.messageSource = messageSource;
      }

      @ExceptionHandler(MyException.class)
      @ResponseStatus
      public String handleMyException(MyException exception, HttpServletRequest request) {
         String userMessage = messageSource.getMessage(exception.getCode().name(), null, request.getLocale());
         log.error(userMessage, exception);
         return userMessage;
      }
    }

In src/main/resources/messages.properties:

    BAD_CONFIG=Incorect application configuration.

The framework will hand to this `@RestControllerAdvice` every `MyException` that slips unhandled from any REST request handler (`@RequestMapping`, `@GetMapping`...). Using a Spring `MessageSource` we read the user message from a message.properties file for the error code of the exception. Based on the Locale sent by the browser in the `Accept-Language` header of the HTTP Request, Spring determines what translation file to read from, eg: messages_RO.properties, messages_FR.properties, or default from messages.properties. Ta-da! Free internationalization for our error messages.

> **Tip**: keeping the translations of error codes on the backend lowers the impact of defining a new error code to report and allows to check at startup that all the enum values are mapped.

There’s a problem though. The messages in the properties files above are fixed. How can we report the user input causing the error back to the UI? We need parameterized messages. After adding other convenience constructors, here’s the complete code of `MyException`:


    public class MyException extends RuntimeException {
      public enum ErrorCode {
         GENERAL,
         BAD_CONFIG;
      }

      private final ErrorCode code;
      private final Object[] params;

      // canonical constructor
      public MyException(String message, ErrorCode code, Throwable cause, Object... params) {
         super(message, cause);
         this.code = code;
         this.params = params;
      }
      // overloaded constructors:
      public MyException(ErrorCode code, Throwable cause, Object... params) {
         this(null, code, cause, params);
      }
      public MyException(ErrorCode code, Object... params) {
         this(null, code, null, params);
      }
      public MyException(String message, ErrorCode code, Object... params) {
         this(message, code, null, params);
      }
      public MyException(String message, Throwable cause) {
         this(message, ErrorCode.GENERAL, cause);
      }
      public MyException(Throwable cause) {
         this(ErrorCode.GENERAL);
      }
      public MyException(String message) {
         this(message, ErrorCode.GENERAL);
      }

      public ErrorCode getCode() {
         return code;
      }
      public Object[] getParams() {
         return params;
      }
    }  


In src/main/resources/messages.properties:

    BAD_CONFIG=Incorect application configuration. User info: {0}

To see the entire app running, checkout [this commit](https://github.com/victorrentea/exceptions-guide/commit/7aa14bd5215077a415db4cffb05c3bebb0a7405b), start `SpringBoot` class and navigate to [http://localhost:8080](http://localhost:8080).

## Handling Unexpected Exceptions

How about other unexpected runtime exceptions (not MyException instances)? To test that, let’s “forget” to set the offer date on the Order we pass to `biz.applyDiscount()`...

Oh NO!

A Stack Trace in the browser!!

To stop the trace from reaching the user, we add a second method in our GlobalExceptionHandler marked with  `@ExceptionHandler(RuntimeException.class)`.

    @ExceptionHandler(RuntimeException.class)
    @ResponseStatus
    public String handleRuntimeException(RuntimeException exception, HttpServletRequest request) {
      String userMessage = messageSource.getMessage(ErrorCode.GENERAL.name(), null, request.getLocale());
      log.error(userMessage, exception);
      return userMessage;
    }

Note that we reused a similar code to the previous method to read the user error messages from the same `.properties` file.

### Reasons to Catch-Rethrow

There are four reasons to catch an exception and rethrow it wrapped in a new one:
1. Report user message, via enum codes
2. Unit-Test a thrown exception, via enum code
3. Report debugging info, via new exception message
4. Get rid of a Checked Exception, when none of the above applies

Let’s focus on the last one.

In our current setup, we have multiple options to get rid of those annoying checked exception(s):
A. `throw new MyException(GENERAL, checkedException)`
B.  `throw new MyException(checkedException)` - identic to (A), via overloaded constructor
C. `throw new RuntimeException(checkedException)` - handled just as (A) by our Global Exception Handler

But if all you’re trying to do is get rid of a checked exception, there’s another shorter solution for you. Coming from the black magic realm of the Lombok library, `@SneakyThrows` will instruct the Lombok annotation processor to hack the bytecode while javac generates it. This would allow the following code to compile (mind that there’s no `catch`, nor `throws` clause for the checked exceptions).

    @SneakyThrows
    public static Date getLastPromoDate() {
      Properties props = new Properties();
      try (FileReader reader = new FileReader("config.properties")) {
         props.load(reader);
      }
      SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd");
      return format.parse(props.getProperty("last.promo.date"));
    }

Knowing that Lombok is able to tweak your code __as it’s compiled__, one might expect that the current body of my function will be surrounded by `try { ... } catch (Exception e) { throw new RuntimeException(e); }`.

That’s a fair expectation. 

However, since Java8, Lombok does it a bit differently. Let’s see.

Decompiling the Config.class file we find the following code: 

    public static Date getLastPromoDate() {
      try {
        ...<the body of my method>...
      } catch (Throwable var6) {
         throw var6;
      }
    }

Indeed, Lombok did add a try-catch block around my function’s body, but it caught Throwable and rethrew it without wrapping it at all! 

Something is wrong!

The bytecode throws a `Throwable` from a method that doesn’t declare `throws Throwable`! If you copy-paste the code in a .java file, you’ll quickly see that this doesn’t compile!! But was it compiled in the first place?!

To understand how’s that even possible, you have to learn that the distinction between checked and runtime exceptions is only enforced by the Java compiler (javac). The JVM (Java Runtime) does NOT care what kind of exception you throw - it propagates any exception the same way. So the Lombok processor tricked javac into producing bytecode that represents code that wouldn’t be compilable by javac.

That’s very creepy!!

..

And a minute after, you get this strange concern: if the checked exception is invisibly thrown, would you be able to catch it later? Writing `try { Config.getLastPromoDate(); } catch (ParseException e) {}` won’t compile because nothing in the `try` block throws a ParseException. And javac will reject that. See for yourself: [commit](https://github.com/victorrentea/exceptions-guide/commit/aa1ba41cb1b7694ebb087bd405b7580b572b8ab2)

So what does this mean? It means that the exceptions hidden using `@SneakyThrows` aren’t supposed to be caught again individually. Instead, a general `catch (Exception)` should be in place somewhere down the call stack to catch the _invisible checked exception_.

Some of you might be disgusted at this point. Others might be super-excited. I’m not here to judge but only to report the techniques that have become wide-spread in the hundreds of projects I trained for. I agree that it can be misused if careless (more in an upcoming talk or article).

Okay, okay... But why?!

Because Java is a 25-years old language. 25 years is a long time to carry some baggage. So Lombok is effectively hacking the language to make us write less and more focussed code. As [my tweet puts it](https://twitter.com/VictorRentea/status/1282566184672665601), when people get tired of Java’s verbosity, they usually face a decision: become a Scala or Clojure scientist, Kotlin hacker or start cheating Java with Lombok.


But if you run the project now with an incorrect configuration (just mess a bit with the config.properties), you’ll see a stack trace again! Why?! Because the Global Exception Handler was only catching `RuntimeExceptions`, but suddenly we have hidden checked exceptions sneaking around.

Don’t worry: changing `RuntimeException` to `Exception` in our Global Exception Handler fixes the issue. Here’s the [final commit on this section](https://github.com/victorrentea/exceptions-guide/commit/85b43e6c5a1521a9202672d659e1e8b8319cfe3f).

## Avoiding NullPointerException
This section turned into a separate article: [Avoiding NullPointerException](avoiding-null-pointer.html)

## Checked Exceptions and Streams
Java 8 gave us an extremely powerful weapon against the most frequent Exception in Java: the `Optional`. Unfortunately, Java 8 also brought new headaches regarding exceptions, as the default functional interfaces in Java 8 don’t declare throwing any checked exceptions. So every time you get a checked exception within a lambda, you have to fix that somehow:

    List<String> dateList = asList("2020-10-11", "2020-nov-12", "2020-12-01");
    SimpleDateFormat format = new SimpleDateFormat("yyyy-MM-dd");

    List<Date> dates = dateList.stream().map(s -> {
      try {
         return format.parse(s);
      } catch (ParseException e) {
         throw new RuntimeException(e);
      }
    }).collect(toList());
    System.out.println(dates);


Horrible code. Of course, we could create a dedicated function doing just .parse and casting a spell on it with `@SneakyThrows`. That would work since it’s OUR code we’re hacking.

      ...
      List<Date> dates = dateList.stream().map(s -> uglyParse(format, s)).collect(toList());
      System.out.println(dates);
    }

    @SneakyThrows
    private static Date uglyParse(SimpleDateFormat format, String s) {
      return format.parse(s);
    }


But creating this new method feels wrong. Or you might not be a Lombok fan. To dive in a little more, if there were a `ThrowingFunction` interface out there, our `s->format.parse(s)` could be [target-typed](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html#target-typing) to id. This compiles:

      ...
      ThrowingFunction<String, Date> p = s -> format.parse(s);
      Function<String, Date> f = wrapAsRuntime(p); // TODO
      List<Date> dates = dateList.stream().map(f).collect(toList());
      System.out.println(dates);
    }

    interface ThrowingFunction<T,R> {
      R apply(T t) throws Exception;
    }

Unfortunately, the `Stream.map()` operation demands a `java.util.Function`. What we need is a way to convert from a `ThrowingFunction into a (non-throwing) `Function`. Let’s write the `wrapAsRuntime()` function:

    private static <T,R> Function<T, R> wrapAsRuntime(ThrowingFunction<T, R> p) {
      return t -> {
         try {
            return p.apply(t);
         } catch (Exception e) {
            throw new RuntimeException(e);
         }
      };
    }


Notice that we’ve used generics to make it highly reusable. Now that’s a very convenient idea. So convenient that others thought of creating it many years ago... :smile:

Introducing the `Unchecked.function()` from the [jool library](https://github.com/jOOQ/jOOL) that does EXACTLY what we did just now.

    List<Date> dates = dateList.stream().map(Unchecked.function(format::parse)).collect(toList());

If you are deep in Java 8, working for years with it, then this library is a must-have IMHO.

> **Best-practice**: Whenever checked exceptions annoy in lambdas or method `::` references, use `Unchecked.*` to rethrow it as a `RuntimeException`

And just when you thought it’s over ...

## The Try Monad

Let’s twist the requirements a bit: we are now requested to parse the valid dates and print them IF at least half are valid. This time you can’t let the exceptions propagate anymore, as this would terminate the execution of your stream. Instead, you want to go through all of the items and collect both results and exceptions. 

Introducing .... the `Try<>` monad from the [vavr library](https://www.vavr.io/). This class is a specialization of the `Either<>` concept present in many functional programming languages. It can store either the result or the occurred exception in a single object.

    List<Try<Date>> tries = dateList.stream().map(s -> Try.of(() -> format.parse(s)) ).collect(toList());

In the list of tries above, there can be items with `isSuccess()` either true or false. To count the success ratio, the shortest form is:

    double successRatio = tries.stream().mapToInt(t -> t.isSuccess() ? 1 : 0).average().orElse(0);

Then,

    if (successRatio > .5) {
      List<Date> dates = tries.stream().filter(Try::isSuccess).map(Try::get).collect(toList());
      System.out.println(dates);
    }

Inside the if above we got the successful items and collected them into a list. Problem solved. Twisting a bit the code, we can extract a function returning a `Try<>`, which resembles the style of handling exceptions in other languages like Go and Haskell.

    private static Try<Date> tryParse(SimpleDateFormat format, String s) {
      return Try.of(() -> format.parse(s));
    }

> **Tip**: Consider `vavr.Try<>` when you want to collect both results and exceptions in a single pass through data.

By the way, if you keep thinking at the “Monad” word, here’s a nice article for you: [Monads for Java developers](https://dzone.com/articles/functor-and-monad-examples-in-plain-java)
## Conclusions
- Only allow Runtime Exceptions through the logic of your application. 
- Consider `Unchecked` or `@SneakyThrows` to work around checked exceptions
- Avoid Diaper Anti-Pattern: don’t ever swallow exceptions
- Avoid Log-Rethrow Anti-Pattern: set up and trust the global exception handler
- Use enum error codes to report errors to users or unit-tests
- Use exception message for key debugging information
- Fight `NullPointerException` with early checks or `Optionals`
- Be aware of `Try<>`

If you liked this article, check out [my website](http://victorrentea.ro/) for more talks, articles, company, or personal training. If you want to reach me for questions, comments, or any other ideas, the best way is via Twitter: [@victorrentea](https://twitter.com/VictorRentea). I’m also posting there as well as on [LinkedIN](https://www.linkedin.com/in/victor-rentea-trainer) and [Facebook](https://www.facebook.com/VictorRentea.ro), so you can find me everywhere :smile:
### Left-over tips
- Don’t throw from a `finally` block: the new exception might hide a preexisting exception
- Throw _before_ mutating an object state: leave the objects’ state consistent
- Don’t use exceptions for _normal_ flow control: they are expensive
### Out of Scope
- Async exceptions: exceptions occurring in other threads (Executors, `@Async`, CompletableFuture, Reactive-X, Message Handlers...)
- How to handle `InterruptedExceptions`: [article](https://dzone.com/articles/how-to-handle-the-interruptedexception)

### Disclaimer
I kept using `SimpleDateFormat` throughout this article. I am sorry for that. You should prefer the new Java 8 date types whenever possible: `LocalDate` and `LocalDateTime` which you can parse without having to deal with any checked exceptions (phew!). I used the old `java.util.Date` because it’s commonly used and easily accessible. One last warning: `SimpleDateFormat` instances are NOT thread-safe: never cache them in singletons or static fields in a multithread environment.

