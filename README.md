# Coroutine

# Why Do You Need Coroutines?

Asynchronous programming is used to provide a smooth flowing experience for the client-side and server-side. Now asynchronous means it can execute simultaneously multiple tasks, and other tasks don't need to wait for another task. It is easy to write the asynchronous code in Coroutines and it makes it more easy and readable by eliminating the requirement of callback pattern.

# What Are Kotlin Coroutines?

Coroutine stands for cooperating functions; it is derived from two words - co and routine. The word co stands for cooperation, and routine stands for functions. A coroutine is a feature of Kotlin; it can do all the things that a thread can and is also very efficient. They are lightweight threads that help to write simplified asynchronous code that keeps the application responsive while maintaining heavy tasks like network calls and avoiding tasks from blocking.
Coroutines are executed inside the thread and are also suspendable. Here suspendable means that you can execute some instructions, then stop the coroutine in between the execution and continue when you wish to. Coroutines can also switch between the threads, which gives coroutines an edge over threads.
So as you have understood Kotlin coroutines, now look at some coroutines' features.

# Kotlin Coroutines

## Table of Contents

* [Use cases](#use-cases)
  * [Asynchronous computations](#asynchronous-computations)
  * [Futures](#futures)
  * [Generators](#generators)
  * [Asynchronous UI](#asynchronous-ui)
  * [More use cases](#more-use-cases)
* [Coroutines overview](#coroutines-overview)
  * [Terminology](#terminology)
  * [Continuation interface](#continuation-interface)
  * [Suspending functions](#suspending-functions)
  * [Coroutine builders](#coroutine-builders)
  * [Coroutine context](#coroutine-context)
  * [Continuation interceptor](#continuation-interceptor)
 
## Use cases

A coroutine can be thought of as an instance of _suspendable computation_, i.e. the one that can suspend at some 
points and later resume execution possibly on another thread. Coroutines calling each other 
(and passing data back and forth) can form 
the machinery for cooperative multitasking.

Asynchronous programming is used to provide a smooth flowing experience for the client-side and server-side. Now asynchronous means it can execute simultaneously multiple tasks, and other tasks don't need to wait for another task. It is easy to write the asynchronous code in Coroutines and it makes it more easy and readable by eliminating the requirement of callback pattern.
 
### Asynchronous computations 
 
The first class of motivating use cases for coroutines are asynchronous computations 
Let's take a look at how such computations are done with callbacks. As an inspiration, let's take 
asynchronous I/O (the APIs below are simplified):

```kotlin
// asynchronously read into `buf`, and when done run the lambda
inChannel.read(buf) {
    // this lambda is executed when the reading completes
    bytesRead ->
    ...
    ...
    process(buf, bytesRead)
    
    // asynchronously write from `buf`, and when done run the lambda
    outChannel.write(buf) {
        // this lambda is executed when the writing completes
        ...
        ...
        outFile.close()          
    }
}
```

Note that we have a callback inside a callback here, and while it saves us from a lot of boilerplate (e.g. there's no 
need to pass the `buf` parameter explicitly to callbacks, they just see it as a part of their closure), the indentation
levels are growing every time, and one can easily anticipate the problems that may come at nesting levels greater 
than one (google for "callback hell" to see how much people suffer from this in JavaScript).

This same computation can be expressed straightforwardly as a coroutine (provided that there's a library that adapts
the I/O APIs to coroutine requirements):
 
```kotlin
launch {
    // suspend while asynchronously reading
    val bytesRead = inChannel.aRead(buf) 
    // we only get to this line when reading completes
    ...
    ...
    process(buf, bytesRead)
    // suspend while asynchronously writing   
    outChannel.aWrite(buf)
    // we only get to this line when writing completes  
    ...
    ...
    outFile.close()
}
```

The `aRead()` and `aWrite()` here are special _suspending functions_ — they can _suspend_ execution 
(which does not mean blocking the thread it has been running on) and _resume_ when the call has completed. 
If we squint our eyes just enough to imagine that all the code after `aRead()` has been wrapped in a 
lambda and passed to `aRead()` as a callback, and the same has been done for `aWrite()`, 
we can see that this code is the same as above, only more readable. 

It is our explicit goal to support coroutines in a very generic way, so in this example,
 `launch{}`, `.aRead()`, and `.aWrite()` are just **library functions** geared for
working with coroutines: `launch` is the _coroutine builder_ — it builds and launches coroutine, 
while `aRead`/`aWrite` are special
_suspending functions_ which implicitly receive 
_continuations_ (continuations are just generic callbacks).  

> The example code for `launch{}` is shown in [coroutine builders](#coroutine-builders) section, and
the example code for `.aRead()` is shown in [wrapping callbacks](#wrapping-callbacks) section.

Note, that with explicitly passed callbacks having an asynchronous call in the middle of a loop can be tricky, 
but in a coroutine it is a perfectly normal thing to have:

```kotlin
launch {
    while (true) {
        // suspend while asynchronously reading
        val bytesRead = inFile.aRead(buf)
        // continue when the reading is done
        if (bytesRead == -1) break
        ...
        process(buf, bytesRead)
        // suspend while asynchronously writing
        outFile.aWrite(buf) 
        // continue when the writing is done
        ...
    }
}
```

One can imagine that handling exceptions is also a bit more convenient in a coroutine.

### Futures

There's another style of expressing asynchronous computations: through futures (also known as promises or deferreds).
We'll use an imaginary API here, to apply an overlay to an image:

```kotlin
val future = runAfterBoth(
    loadImageAsync("...original..."), // creates a Future 
    loadImageAsync("...overlay...")   // creates a Future
) {
    original, overlay ->
    ...
    applyOverlay(original, overlay)
}
```

With coroutines, this could be rewritten as

```kotlin
val future = future {
    val original = loadImageAsync("...original...") // creates a Future
    val overlay = loadImageAsync("...overlay...")   // creates a Future
    ...
    // suspend while awaiting the loading of the images
    // then run `applyOverlay(...)` when they are both loaded
    applyOverlay(original.await(), overlay.await())
}
```

> The example code for `future{}` is shown in [building futures](#building-futures) section, and
the example code for `.await()` is shown in [suspending functions](#suspending-functions) section.

Again, less indentation and more natural composition logic (and exception handling, not shown here), 
and no special keywords (like `async` and `await` in C#, JS and other languages)
to support futures: `future{}` and `.await()` are just functions in a library.

### Generators

Another typical use case for coroutines would be lazily computed sequences (handled by `yield` in C#, Python 
and many other languages). Such a sequence can be generated by seemingly sequential code, but at runtime only 
requested elements are computed:

```kotlin
// inferred type is Sequence<Int>
val fibonacci = sequence {
    yield(1) // first Fibonacci number
    var cur = 1
    var next = 1
    while (true) {
        yield(next) // next Fibonacci number
        val tmp = cur + next
        cur = next
        next = tmp
    }
}
```

This code creates a lazy `Sequence` of [Fibonacci numbers](https://en.wikipedia.org/wiki/Fibonacci_number), 
that is potentially infinite 
(exactly like [Haskell's infinite lists](http://www.techrepublic.com/article/infinite-list-tricks-in-haskell/)). 
We can request some of it, for example, through `take()`:
 
```kotlin
println(fibonacci.take(10).joinToString())
```

> This will print `1, 1, 2, 3, 5, 8, 13, 21, 34, 55`
  You can try this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/fibonacci.kt)
 
The strength of generators is in supporting arbitrary control flow, such as `while` (from the example above),
`if`, `try`/`catch`/`finally` and everything else: 
 
```kotlin
val seq = sequence {
    yield(firstItem) // suspension point

    for (item in input) {
        if (!item.isValid()) break // don't generate any more items
        val foo = item.toFoo()
        if (!foo.isGood()) continue
        yield(foo) // suspension point        
    }
    
    try {
        yield(lastItem()) // suspension point
    }
    finally {
        // some finalization code
    }
} 
```

> The example code for `sequence{}` and `yield()` is shown in
[restricted suspension](#restricted-suspension) section.

Note that this approach also allows to express `yieldAll(sequence)` as a library function 
(as well as `sequence{}` and `yield()` are), which simplifies joining lazy sequences and allows
for efficient implementation.

### Asynchronous UI

A typical UI application has a single event dispatch thread where all UI operations happen. 
Modification of UI state from other threads is usually not allowed. All UI libraries provide
some kind of primitive to move execution back to UI thread. Swing, for example, has 
[`SwingUtilities.invokeLater`](https://docs.oracle.com/javase/8/docs/api/javax/swing/SwingUtilities.html#invokeLater-java.lang.Runnable-),
JavaFX has 
[`Platform.runLater`](https://docs.oracle.com/javase/8/javafx/api/javafx/application/Platform.html#runLater-java.lang.Runnable-), 
Android has
[`Activity.runOnUiThread`](https://developer.android.com/reference/android/app/Activity.html#runOnUiThread(java.lang.Runnable)),
etc.
Here is a snippet of code from a typical Swing application that does some asynchronous
operation and then displays its result in the UI:

```kotlin
makeAsyncRequest {
    // this lambda is executed when the async request completes
    result, exception ->
    
    if (exception == null) {
        // display result in UI
        SwingUtilities.invokeLater {
            display(result)   
        }
    } else {
       // process exception
    }
}
```

This is similar to callback hell that we've seen in [asynchronous computations](#asynchronous-computations) use case
and it is elegantly solved by coroutines, too:
 
```kotlin
launch(Swing) {
    try {
        // suspend while asynchronously making request
        val result = makeRequest()
        // display result in UI, here Swing context ensures that we always stay in event dispatch thread
        display(result)
    } catch (exception: Throwable) {
        // process exception
    }
}
```
 
> The example code for `Swing` context is shown in the [continuation interceptor](#continuation-interceptor) section.
 
All exception handling is performed using natural language constructs. 

### More use cases
 
Coroutines can cover many more use cases, including these:  
 
* Channel-based concurrency (aka goroutines and channels);
* Actor-based concurrency;
* Background processes occasionally requiring user interaction, e.g., show a modal dialog;
* Communication protocols: implement each actor as a sequence rather than a state machine;
* Web application workflows: register a user, validate email, log them in 
(a suspended coroutine may be serialized and stored in a DB).

## Coroutines overview

This section gives an overview of the language mechanisms that enable writing coroutines and 
the standard libraries that govern their semantics.
  
 *  A _suspending lambda_ — a block of code that have to run in a coroutine.
    It looks exactly like an ordinary [lambda expression](https://kotlinlang.org/docs/reference/lambdas.html)
    but its functional type is marked with `suspend` modifier.
    Just like a regular lambda expression is a short syntactic form for an anonymous local function,
    a suspending lambda is a short syntactic form for an anonymous suspending function. It may _suspend_ execution of
    the code without blocking the current thread of execution by invoking suspending functions.
    For example, blocks of code in curly braces following `launch`, `future`, and `sequence` functions,
    as shown in [use cases](#use-cases), are suspending lambdas.

    > Note: Regular lambdas may invoke suspending functions in all places of their code where a 
    [non-local](https://kotlinlang.org/docs/reference/returns.html) `return` statement
    from this lambda is allowed. That is, suspending function calls inside inline lambdas 
    like [`apply{}` block](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/apply.html) are allowed,
    but not in the `noinline` nor in `crossinline` inner lambda expressions. 
    A _suspension_ is treated as a special kind of non-local control transfer.

 *  A _suspending function type_ — is a function type for suspending functions and lambdas. It is just like 
    a regular [function type](https://kotlinlang.org/docs/reference/lambdas.html#function-types), 
    but with `suspend` modifier. For example, `suspend () -> Int` is a type of suspending
    function without arguments that returns `Int`. A suspending function that is declared like `suspend fun foo(): Int`
    conforms to this function type.

 *  A _coroutine builder_ — a function that takes some _suspending lambda_ as an argument, creates a coroutine,
    and, optionally, gives access to its result in some form. For example, `launch{}`, `future{}`,
    and `sequence{}` as shown in [use cases](#use-cases), are coroutine builders.
    The standard library provides primitive coroutine builders that are used to define all other coroutine builders.

    > Note: Some languages have hard-coded support for particular ways to create and start a coroutines that define
    how their execution and result are represented. For example, `generate` _keyword_ may define a coroutine that 
    returns a certain kind of iterable object, while `async` _keyword_ may define a coroutine that returns a
    certain kind of promise or task. Kotlin does not have keywords or modifiers to define and start a coroutine. 
    Coroutine builders are simply functions defined in a library. 
    In case where a coroutine definition takes the form of a method body in another language, 
    in Kotlin such method would typically be a regular method with an expression body, 
    consisting of an invocation of some library-defined coroutine builder whose last argument is a suspending lambda:
 
    ```kotlin
    fun doSomethingAsync() = async { ... }
    ```

 *  A _suspension point_ — is a point during coroutine execution where the execution of the coroutine _may be suspended_. 
    Syntactically, a suspension point is an invocation of suspending function, but the _actual_
    suspension happens when the suspending function invokes the standard library primitive to suspend the execution.

 *  A _continuation_ — is a state of the suspended coroutine at suspension point. It conceptually represents 
    the rest of its execution after the suspension point. For example:

    ```kotlin
    sequence {
        for (i in 1..10) yield(i * i)
        println("over")
    }  
    ```  

    Here, every time the coroutine is suspended at a call to suspending function `yield()`, 
    _the rest of its execution_ is represented as a continuation, so we have 10 continuations: 
    first runs the loop with `i = 2` and suspends, second runs the loop with `i = 3` and suspends, etc, 
    the last one prints "over" and completes the coroutine. The coroutine that is _created_, but is not 
    _started_ yet, is represented by its _initial continuation_ of type `Continuation<Unit>` that consists of
    its whole execution.

As mentioned above, one of the driving requirements for coroutines is flexibility:
we want to be able to support many existing asynchronous APIs and other use cases and minimize 
the parts hard-coded into the compiler. As a result, the compiler is only responsible for support
of suspending functions, suspending lambdas, and the corresponding suspending function types. 
There are few primitives in the standard library and the rest is left to application libraries. 

### Continuation interface

Here is the definition of the standard library interface 
[`Continuation`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/index.html)  
(defined in `kotlin.coroutines` package), which represents a generic callback:

```kotlin
interface Continuation<in T> {
   val context: CoroutineContext
   fun resumeWith(result: Result<T>)
}
```

The context is covered in details in [coroutine context](#coroutine-context) section and represents an arbitrary
user-defined context that is associated with the coroutine. The `resumeWith` function is a _completion_
callback that is used to report either a success (with a value) or a failure (with an exception) on coroutine completion.

There are two extension functions defined in the same package for convenience:

```kotlin
fun <T> Continuation<T>.resume(value: T)
fun <T> Continuation<T>.resumeWithException(exception: Throwable)
```

### Suspending functions

An implementation of a typical _suspending function_ like `.await()` looks like this:
  
```kotlin
suspend fun <T> CompletableFuture<T>.await(): T =
    suspendCoroutine<T> { cont: Continuation<T> ->
        whenComplete { result, exception ->
            if (exception == null) // the future has been completed normally
                cont.resume(result)
            else // the future has completed with an exception
                cont.resumeWithException(exception)
        }
    }
``` 

> You can get this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/future/await.kt).
  Note: this simple implementation suspends coroutine forever if the future never completes.
  The actual implementation in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines)
  supports cancellation.

The `suspend` modifier indicates that this is a function that can suspend execution of a coroutine.
This particular function is defined as an 
[extension function](https://kotlinlang.org/docs/reference/extensions.html)
on `CompletableFuture<T>` type so that its usage reads naturally in the left-to-right order
that corresponds to the actual order of execution:

```kotlin
doSomethingAsync(...).await()
```
 
A modifier `suspend` may be used on any function: top-level function, extension function, member function, 
local function, or operator function.

> Property getters and setters, constructors, and some operators functions 
  (namely `getValue`, `setValue`, `provideDelegate`, `get`, `set`, and `equals`) cannot have `suspend` modifier. 
  These restrictions may be lifted in the future.
 
Suspending functions may invoke any regular functions, but to actually suspend execution they must
invoke some other suspending function. In particular, this `await` implementation invokes a suspending function
[`suspendCoroutine`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html) 
that is defined in the standard library (in `kotlin.coroutines` package) 
as a top-level suspending function:

```kotlin
suspend fun <T> suspendCoroutine(block: (Continuation<T>) -> Unit): T
```

When `suspendCoroutine` is called inside a coroutine (and it can _only_ be called inside
a coroutine, because it is a suspending function) it captures the execution state of a coroutine 
in a _continuation_ instance and passes this continuation to the specified `block` as an argument.
To resume execution of the coroutine, the block invokes `continuation.resumeWith()`
(either directly or using `continuation.resume()` or `continuation.resumeWithException()` extensions)
in this thread or in some other thread at some later time. 
The _actual_ suspension of a coroutine happens when the `suspendCoroutine` block returns without invoking `resumeWith`.
If continuation was resumed before returning from inside of the block,
then the coroutine is not considered to have been suspended and continues to execute.

The result passed to `continuation.resumeWith()` becomes the result of `suspendCoroutine` call,
which, in turn, becomes the result of `.await()`.

Resuming the same continuation more than once is not allowed and produces `IllegalStateException`.

> Note: That is the key difference between coroutines in Kotlin and first-class delimited continuations in 
functional languages like Scheme or continuation monad in Haskell. The choice to support only resume-once 
continuations is purely pragmatic as none of the intended [use cases](#use-cases) need multi-shot continuations. 
However, multi-shot continuations can be implemented as a separate library by suspending coroutine
with low-level [coroutine intrinsics](#coroutine-intrinsics) and cloning the state of the coroutine that is
captured in continuation, so that its clone can be resumed again. 

### Coroutine builders

Suspending functions cannot be invoked from regular functions, so the standard library provides functions
to start coroutine execution from a regular non-suspending scope. Here is the implementation of a simple
`launch{}` _coroutine builder_:

```kotlin
fun launch(context: CoroutineContext = EmptyCoroutineContext, block: suspend () -> Unit) =
    block.startCoroutine(Continuation(context) { result ->
        result.onFailure { exception ->
            val currentThread = Thread.currentThread()
            currentThread.uncaughtExceptionHandler.uncaughtException(currentThread, exception)
        }
    })
```

> You can get this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/run/launch.kt).

This implementation uses [`Continuation(context) { ... }`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation.html) 
function (from `kotlin.coroutines` package) that provides a 
shortcut to implementing a `Continuation` interface with a given value for its `context` and the body of 
`resumeWith` function. This continuation is passed to 
[`block.startCoroutine(...)`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/start-coroutine.html) extension function 
(from `kotlin.coroutines` package) as a _completion continuation_.

The completion of coroutine invokes its completion continuation. Its `resumeWith`
function is invoked when coroutine _completes_ with success or failure.
Because `launch` does "fire-and-forget"
coroutine, it is defined for suspending functions with `Unit` return type and actually ignores
this result in its `resume` function. If coroutine execution completes with exception,
then the uncaught exception handler of the current thread is used to report it.

> Note: this simple implementation returns `Unit` and provides no access to the state of the coroutine at all. 
  The actual implementation in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) is more
  complex, because it returns an instance of `Job` interface that represents a coroutine and can be cancelled.

The context is covered in details in [coroutine context](#coroutine-context) section.
The `startCoroutine` is defined in the standard library as an extension for no parameter and
single parameter suspending function types:

```kotlin
fun <T> (suspend  () -> T).startCoroutine(completion: Continuation<T>)
fun <R, T> (suspend  R.() -> T).startCoroutine(receiver: R, completion: Continuation<T>)
```

The `startCoroutine` creates coroutine and starts its execution immediately, in the current thread (but see remark below),
until the first _suspension point_, then it returns.
Suspension point is an invocation of some [suspending function](#suspending-functions) in the body of the coroutine and
it is up to the code of the corresponding suspending function to define when and how the coroutine execution resumes.

> Note: continuation interceptor (from the context) that is covered [later](#continuation-interceptor), can dispatch
the execution of the coroutine, _including_ its initial continuation, into another thread.

### Coroutine context

Coroutine context is a persistent set of user-defined objects that can be attached to the coroutine. It
may include objects responsible for coroutine threading policy, logging, security and transaction aspects of the
coroutine execution, coroutine identity and name, etc. Here is the simple mental model of coroutines and their
contexts. Think of a coroutine as a light-weight thread. In this case, coroutine context is just like a collection 
of thread-local variables. The difference is that thread-local variables are mutable, while coroutine context is
immutable, which is not a serious limitation for coroutines, because they are so light-weight that it is easy to
launch a new coroutine when there is a need to change anything in the context.

The standard library does not contain any concrete implementations of the context elements, 
but has interfaces and abstract classes so that all these aspects
can be defined in libraries in a _composable_ way, so that aspects from different libraries can coexist
peacefully as elements of the same context.

Conceptually, coroutine context is an indexed set of elements, where each element has a unique key.
It is a mix between a set and a map. Its elements have keys like in a map, but its keys are directly associated
with elements, more like in a set. The standard library defines the minimal interface for 
[`CoroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/index.html)
(in `kotlin.coroutines` package):

```kotlin
interface CoroutineContext {
    operator fun <E : Element> get(key: Key<E>): E?
    fun <R> fold(initial: R, operation: (R, Element) -> R): R
    operator fun plus(context: CoroutineContext): CoroutineContext
    fun minusKey(key: Key<*>): CoroutineContext

    interface Element : CoroutineContext {
        val key: Key<*>
    }

    interface Key<E : Element>
}
```

The `CoroutineContext` itself has four core operations available on it:

* Operator [`get`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/get.html)
  provides type-safe access to an element for a given key. It can be used with `[..]` notation
  as explained in [Kotlin operator overloading](https://kotlinlang.org/docs/reference/operator-overloading.html).
* Function [`fold`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/fold.html)
  works like [`Collection.fold`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/fold.html)
  extension in the standard library and provides means to iterate all elements in the context.
* Operator [`plus`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/plus.html)
  works like [`Set.plus`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.collections/plus.html)
  extension in the standard library and returns a combination of two contexts with elements on the right-hand side
  of plus replacing elements with the same key on the left-hand side.
* Function [`minusKey`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/minus-key.html)
  returns a context that does not contain a specified key.

An [`Element`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-coroutine-context/-element/index.html) 
of the coroutine context is a context itself. 
It is a singleton context with this element only.
This enables creation of composite contexts by taking library definitions of coroutine context elements and
joining them with `+`. For example, if one library defines `auth` element with user authorization information,
and some other library defines `threadPool` object with some execution context information,
then you can use a `launch{}` [coroutine builder](#coroutine-builders) with the combined context using
`launch(auth + threadPool) {...}` invocation.

> Note: [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) provides several context elements, 
  including `Dispatchers.Default` object that dispatches execution of coroutine onto a shared pool of background threads.

Standard library provides
[`EmptyCoroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-empty-coroutine-context/index.html)
&mdash; an instance of `CoroutineContext` without any elements (empty).

All third-party context elements shall extend 
[`AbstractCoroutineContextElement`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-abstract-coroutine-context-element/index.html)
class that is provided by the standard library (in `kotlin.coroutines` package). 
The following style is recommended for library defined context elements.
The example below shows a hypothetical authorization context element that stores current user name:

```kotlin
class AuthUser(val name: String) : AbstractCoroutineContextElement(AuthUser) {
    companion object Key : CoroutineContext.Key<AuthUser>
}
```

> This example can be found [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/auth.kt).

The definition of context `Key` as a companion object of the corresponding element class enables fluent access
to the corresponding element of the context. Here is a hypothetical implementation of suspending function that
needs to check the name of the current user:

```kotlin
suspend fun doSomething() {
    val currentUser = coroutineContext[AuthUser]?.name ?: throw SecurityException("unauthorized")
    // do something user-specific
}
```

It uses a top-level [`coroutineContext`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/coroutine-context.html)
property (from `kotlin.coroutines` package) that is available in
suspending functions to retrieve the context of the current coroutine.

### Continuation interceptor

Let's recap [asynchronous UI](#asynchronous-ui) use case. Asynchronous UI applications must ensure that the 
coroutine body itself is always executed in UI thread, despite the fact that various suspending functions 
resume coroutine execution in arbitrary threads. This is accomplished using a _continuation interceptor_.
First of all, we need to fully understand the life-cycle of a coroutine. Consider a snippet of code that uses 
[`launch{}`](#coroutine-builders) coroutine builder:

```kotlin
launch(Swing) {
    initialCode() // execution of initial code
    f1.await() // suspension point #1
    block1() // execution #1
    f2.await() // suspension point #2
    block2() // execution #2
}
```

Coroutine starts with execution of its `initialCode` until the first suspension point. At the suspension point it
_suspends_ and, after some time, as defined by the corresponding suspending function, it _resumes_ to execute 
`block1`, then it suspends again and resumes to execute `block2`, after which it _completes_.

Continuation interceptor has an option to intercept and wrap the continuation that corresponds to the
execution of `initialCode`, `block1`, and `block2` from their resumption to the subsequent suspension points.
The initial code of the coroutine is treated as a
resumption of its _initial continuation_. The standard library provides the 
[`ContinuationInterceptor`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation-interceptor/index.html) 
interface (in `kotlin.coroutines` package):
 
```kotlin
interface ContinuationInterceptor : CoroutineContext.Element {
    companion object Key : CoroutineContext.Key<ContinuationInterceptor>
    fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T>
    fun releaseInterceptedContinuation(continuation: Continuation<*>)
}
 ```
 
The `interceptContinuation` function wraps the continuation of the coroutine. Whenever coroutine is suspended,
coroutine framework uses the following line of code to wrap the actual `continuation` for the subsequent
resumption:

```kotlin
val intercepted = continuation.context[ContinuationInterceptor]?.interceptContinuation(continuation) ?: continuation
```
 
Coroutine framework caches the resulting intercepted continuation for each actual instance of continuation
and invokes `releaseInterceptedContinuation(intercepted)` when it is no longer needed. 
See [implementation details](#implementation-details) section for more details.

> Note, that suspending functions like `await` may or may not actually suspend execution of
a coroutine. For example, `await` implementation that was shown in [suspending functions](#suspending-functions) section
does not actually suspend coroutine when a future is already complete (in this case it invokes `resume` immediately and 
execution continues without the actual suspension). A continuation is intercepted only when the actual suspension 
happens during execution of a coroutine, that is when `suspendCoroutine` block returns without invoking
`resume`.

Let us take a look at a concrete example code for `Swing` interceptor that dispatches execution onto
Swing UI event dispatch thread. We start with a definition of a `SwingContinuation` wrapper class that
dispatches continuation onto the Swing event dispatch thread using `SwingUtilities.invokeLater`:

```kotlin
private class SwingContinuation<T>(val cont: Continuation<T>) : Continuation<T> {
    override val context: CoroutineContext = cont.context
    
    override fun resumeWith(result: Result<T>) {
        SwingUtilities.invokeLater { cont.resumeWith(result) }
    }
}
```

Then define `Swing` object that is going to serve as the corresponding context element and implement
`ContinuationInterceptor` interface:
  
```kotlin
object Swing : AbstractCoroutineContextElement(ContinuationInterceptor), ContinuationInterceptor {
    override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        SwingContinuation(continuation)
}
```

> You can get this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/context/swing.kt).
  Note: the actual implementation of `Swing` object in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 
  also supports coroutine debugging facilities that provide and display the identifier of the currently running 
  coroutine in the name of the thread that is currently running this coroutine.

Now, one can use `launch{}` [coroutine builder](#coroutine-builders) with `Swing` parameter to 
execute a coroutine that is running completely in Swing event dispatch thread:
 
 ```kotlin
launch(Swing) {
   // code in here can suspend, but will always resume in Swing EDT
}
```

> The actual implementation of `Swing` context in [kotlinx.coroutines](https://github.com/kotlin/kotlinx.coroutines) 
  is more complex since it is integrated with time and debugging facilities of the library.

### Restricted suspension

A different kind of coroutine builder and suspension function is needed to implement `sequence{}` and `yield()`
from [generators](#generators) use case. Here is the example code for `sequence{}` coroutine builder:

```kotlin
fun <T> sequence(block: suspend SequenceScope<T>.() -> Unit): Sequence<T> = Sequence {
    SequenceCoroutine<T>().apply {
        nextStep = block.createCoroutine(receiver = this, completion = this)
    }
}
```

It uses a different primitive from the standard library called 
[`createCoroutine`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/create-coroutine.html) 
which is similar `startCoroutine` (that was explained in [coroutine builders](coroutine-builders) section). 
However it _creates_ a coroutine, but does _not_ start it. 
Instead, it returns its _initial continuation_ as a reference to `Continuation<Unit>`:

```kotlin
fun <T> (suspend () -> T).createCoroutine(completion: Continuation<T>): Continuation<Unit>
fun <R, T> (suspend R.() -> T).createCoroutine(receiver: R, completion: Continuation<T>): Continuation<Unit>
```

The other difference is that _suspending lambda_
`block` for this builder is an 
[_extension lambda_](https://kotlinlang.org/docs/reference/lambdas.html#function-literals-with-receiver) 
with `SequenceScope<T>` receiver.
The `SequenceScope` interface provides the _scope_ for the generator block and is defined in a library as:

```kotlin
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

To avoid creation of multiple objects, `sequence{}` implementation defines `SequenceCoroutine<T>` class that
implements `SequenceScope<T>` and also implements `Continuation<Unit>`, so it can serve both as
a `receiver` parameter for `createCoroutine` and as its `completion` continuation parameter. 
The simple implementation for `SequenceCoroutine<T>` is shown below:

```kotlin
private class SequenceCoroutine<T>: AbstractIterator<T>(), SequenceScope<T>, Continuation<Unit> {
    lateinit var nextStep: Continuation<Unit>

    // AbstractIterator implementation
    override fun computeNext() { nextStep.resume(Unit) }

    // Completion continuation implementation
    override val context: CoroutineContext get() = EmptyCoroutineContext

    override fun resumeWith(result: Result<Unit>) {
        result.getOrThrow() // bail out on error
        done()
    }

    // Generator implementation
    override suspend fun yield(value: T) {
        setNext(value)
        return suspendCoroutine { cont -> nextStep = cont }
    }
}
```
 
> You can get this code [here](https://github.com/kotlin/kotlin-coroutines-examples/tree/master/examples/sequence/sequence.kt).
  Note, that the standard library provides out-of-the-box optimized implementation of this
  [`sequence`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/sequence.html)
  function (in `kotlin.sequences` package) with additional support for 
  [`yieldAll`](http://kotlinlang.org/api/latest/jvm/stdlib/kotlin.sequences/-sequence-scope/yield-all.html) function.
  
> The actual code for `sequence` uses experimental `BuilderInference` feature that enables declaration 
  of `fibonacci` as shown in [generators](#generators) section without explicit specification of
  sequence type parameter `T`. Instead, it is inferred from the types passed to `yield` calls.

The implementation of `yield` uses `suspendCoroutine` [suspending function](#suspending-functions) to suspend
the coroutine and to capture its continuation. Continuation is stored as `nextStep` to be resumed when the 
`computeNext` is invoked.
 
However, `sequence{}` and `yield()`, as shown above, are not ready for an arbitrary suspending function
to capture the continuation in their scope. They work _synchronously_.
They need absolute control on how continuation is captured, 
where it is stored, and when it is resumed. They form _restricted suspension scope_. 
The ability to restrict suspensions is provided by `@RestrictsSuspension` annotation that is placed
on the scope class or interface, in the above example this scope interface is `SequenceScope`:

```kotlin
@RestrictsSuspension
interface SequenceScope<in T> {
    suspend fun yield(value: T)
}
```

This annotation enforces certain restrictions on suspending functions that can be used in the
scope of `sequence{}` or similar synchronous coroutine builder.
Any extension suspending lambda or function that has _restricted suspension scope_ class or interface 
(marked with `@RestrictsSuspension`) as its receiver is 
called a _restricted suspending function_.
Restricted suspending functions can only invoke member or
extension suspending functions on the same instance of their restricted suspension scope. 

In particular, it means that
no `SequenceScope` extension of lambda in its scope can invoke `suspendContinuation` or other
general suspending function. To suspend the execution of a `sequence` coroutine they must ultimately invoke
`SequenceScope.yield`. The implementation of `yield` itself is a member function of `SequenceScope`
implementation and it does not have any restrictions (only _extension_ suspending lambdas and functions are restricted).

It makes little sense to support arbitrary contexts for such a restricted coroutine builder as `sequence`,
because their scope class or interface (`SequenceScope` in this example) serves as a context, so restricted
coroutines must always use `EmptyCoroutineContext` as their context and that is what `context` property getter implementation
of `SequenceCoroutine` returns. An attempt to create a restricted coroutine with the context that is other than 
`EmptyCoroutinesContext` results in `IllegalArgumentException`.





### References

* Further reading:
   * [Coroutines Reference Guide](http://kotlinlang.org/docs/reference/coroutines/coroutines-guide.html) **READ IT FIRST!**.




