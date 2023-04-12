# Coroutine

# Kotlin Coroutines

## Table of Contents

* [Use cases](#use-cases)
  * [Why Need](#why-need)
  * [What Are Kotlin Coroutines](#what-are-kotlin-coroutines)
  * [WithContext](#withcontext)
  * [Launch Function](#launch-function)
  * [Async Function](#async-function)
  * [Table of Differences Function](#table-of-differences)
  * [Dispatchers](#dispatchers)
  * [Coroutine Scope](#coroutine-scope)
## Use cases




A coroutine can be thought of as an instance of _suspendable computation_, i.e. the one that can suspend at some 
points and later resume execution possibly on another thread. Coroutines calling each other 
(and passing data back and forth) can form 
the machinery for cooperative multitasking.

Asynchronous programming is used to provide a smooth flowing experience for the client-side and server-side. Now asynchronous means it can execute simultaneously multiple tasks, and other tasks don't need to wait for another task. It is easy to write the asynchronous code in Coroutines and it makes it more easy and readable by eliminating the requirement of callback pattern.
 
 
 
 ### Why Need
 
 

Asynchronous programming is used to provide a smooth flowing experience for the client-side and server-side. Now asynchronous means it can execute simultaneously multiple tasks, and other tasks don't need to wait for another task. It is easy to write the asynchronous code in Coroutines and it makes it more easy and readable by eliminating the requirement of callback pattern.

### What Are Kotlin Coroutines?

Coroutine stands for cooperating functions; it is derived from two words - co and routine. The word co stands for cooperation, and routine stands for functions. A coroutine is a feature of Kotlin; it can do all the things that a thread can and is also very efficient. They are lightweight threads that help to write simplified asynchronous code that keeps the application responsive while maintaining heavy tasks like network calls and avoiding tasks from blocking.
Coroutines are executed inside the thread and are also suspendable. Here suspendable means that you can execute some instructions, then stop the coroutine in between the execution and continue when you wish to. Coroutines can also switch between the threads, which gives coroutines an edge over threads.
So as you have understood Kotlin coroutines, now look at some coroutines' features.


### withContext

withContext is a suspend function through which we can do a task by providing the Dispatchers on which we want the task to be done. withContext does not create a new coroutine, it only shifts the context of the existing coroutine and it's a suspend function whereas launch and async create a new coroutine and they are not suspend functions. As withContext is a suspend function, it can be only called from a suspend function or a coroutine. So, both the above functions are suspend functions.


### launch

* launch is a function that creates a coroutine and dispatches the execution of its function body to the corresponding dispatcher.
* The launch is basically fire and forget.
* launch does not return anything.
* launch cannot be used when you need the parallel execution of network calls.
* launch will not block your main thread.


### Async Function

Async is also used to start the coroutines, but it blocks the main thread at the entry point of the await() function in the program. Following is a Kotlin Program Using Async:

```kotlin
// kotlin program for demonstration of async
fun GFG
{
  Log.i("Async", "Before")
  val resultOne = Async(Dispatchers.IO) { function1() }
  val resultTwo = Async(Dispatchers.IO) { function2() }
  Log.i("Async", "After")
  val resultText = resultOne.await() + resultTwo.await()
  Log.i("Async", resultText)
}
 
suspend fun function1(): String
{
  delay(1000L)
  val message = "function1"
  Log.i("Async", message)
  return message
}
 
suspend fun function2(): String
{
  delay(100L)
  val message = "function2"
  Log.i("Async", message)
  return message
}
```

When to Use Async?

 

When making two or more network call in parallel, but you need to wait for the answers before computing the output, ie use async for results from multiple tasks that run in parallel. If you use async and do not wait for the result, it will work exactly the same as launch.


### Table of Differences
 

Below is the table of differences between Launch and Async

| Launch | Async  |
| :-----: | :---: |
| The launch is basically fire and forget. | Async is basically performing a task and return a result. |
| launch{} does not return anything. | async{ }, which has an await() function returns the result of the coroutine. |
| launch{} cannot be used when you need the parallel execution of network calls. | Use async only when you need the parallel execution network calls. |
| launch{} will not block your main thread. | Async will block the main thread at the entry point of the await() function.  |
| Execution of other parts  of the code will not wait for the launch result since launch is not a suspend call | Execution of the other parts of the code will have to wait for the result of the await() function.|
| It is not possible for the launch to work like async in any case or condition. | If you use async and do not wait for the result, it will work exactly the same as launch.|
| Launch can be used at places if you don’t need the result from the method called. | Use async when you need the results from the multiple tasks that run in parallel.|


### Dispatchers

Dispatchers help coroutines in deciding the thread on which the work has to be done. Dispatchers are passed as the arguments to the GlobalScope by mentioning which type of dispatchers we can use depending on the work that we want the coroutine to do.

Types of Dispatchers
There are majorly 4 types of Dispatchers.
1.Main Dispatcher
2.IO Dispatcher
3.Default Dispatcher
4.Unconfined Dispatcher


### Main Dispatcher:

It starts the coroutine in the main thread. It is mostly used when we need to perform the UI operations within the coroutine, as UI can only be changed from the main thread(also called the UI thread).

```kotlin
GlobalScope.launch(Dispatchers.Main) {
	Log.i("Inside Global Scope ",Thread.currentThread().name.toString())
		// getting the name of thread in which
			// our coroutine has been launched
	}
	Log.i("Main Activity ",Thread.currentThread().name.toString())
```

### IO Dispatcher:

It starts the coroutine in the IO thread, it is used to perform all the data operations such as networking, reading, or writing from the database, reading, or writing to the files eg: Fetching data from the database is an IO operation, which is done on the IO thread.

```kotlin
GlobalScope.launch(Dispatchers.IO) {
	Log.i("Inside IO dispatcher ",Thread.currentThread().name.toString())
		// getting the name of thread in which
			// our coroutine has been launched
	}
	Log.i("Main Activity ",Thread.currentThread().name.toString())
```

Default Dispatcher:
It starts the coroutine in the Default Thread. We should choose this when we are planning to do Complex and long-running calculations, which can block the main thread and freeze the UI eg: Suppose we need to do the 10,000 calculations and we are doing all these calculations on the UI thread ie main thread, and if we wait for the result or 10,000 calculations, till that time our main thread would be blocked, and our UI will be frozen, leading to poor user experience. So in this case we need to use the Default Thread. The default dispatcher that is used when coroutines are launched in GlobalScope is represented by Dispatchers. Default and uses a shared background pool of threads, so launch(Dispatchers.Default) { … } uses the same dispatcher as GlobalScope.launch { … }.

```kotlin
GlobalScope.launch(Dispatchers.Default) {
		Log.i("Inside Default dispatcher ",Thread.currentThread().name.toString())
			// getting the name of thread in which
			// our coroutine has been launched
		}
		Log.i("Main Activity ",Thread.currentThread().name.toString())
```


### Unconfined Dispatcher:
As the name suggests unconfined dispatcher is not confined to any specific thread. It executes the initial continuation of a coroutine in the current call-frame and lets the coroutine resume in whatever thread that is used by the corresponding suspending function, without mandating any specific threading policy. 

### Coroutine Scope

To launch a coroutine, we need to use a coroutine builder like launch or async. These builder functions are actually extensions of the CoroutineScope interface. So, whenever we want to launch a coroutine, we need to start it in some scope.

The scope creates relationships between coroutines inside it and allows us to manage the lifecycles of these coroutines. There are several scopes provided by the kotlinx.coroutines library that we can use when launching a coroutine. There’s also a way to create a custom scope. Let’s have a look.

  * GlobalScope :- The lifecycle of this scope is tied to the lifecycle of the whole application. This means that the scope will stop running either after all of its coroutines have been completed or when the application is stopped.
  

  * runBlocking :-Another scope that comes right out of the box is runBlocking. From the name, we might guess that it creates a scope and runs a coroutine in a blocking way. This means it blocks the current thread until all childrens’ coroutines complete their executions.

  * coroutineScope :-For all the cases when we don’t need thread blocking, we can use coroutineScope. Similarly to runBlocking, it will wait for its children to complete. But unlike runBlocking, this scope doesn’t block the current thread but only suspends it because coroutineScope is a suspending function.

  * Custom Coroutine Scope :- There might be cases when we need to have some specific behavior of the scope to get a different approach in managing the coroutines. To achieve that, we can implement the CoroutineScope interface and implement our custom scope for coroutine handling.




