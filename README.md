# Coroutine

# Why Do You Need Coroutines?

Asynchronous programming is used to provide a smooth flowing experience for the client-side and server-side. Now asynchronous means it can execute simultaneously multiple tasks, and other tasks don't need to wait for another task. It is easy to write the asynchronous code in Coroutines and it makes it more easy and readable by eliminating the requirement of callback pattern.

# What Are Kotlin Coroutines?

Coroutine stands for cooperating functions; it is derived from two words - co and routine. The word co stands for cooperation, and routine stands for functions. A coroutine is a feature of Kotlin; it can do all the things that a thread can and is also very efficient. They are lightweight threads that help to write simplified asynchronous code that keeps the application responsive while maintaining heavy tasks like network calls and avoiding tasks from blocking.
Coroutines are executed inside the thread and are also suspendable. Here suspendable means that you can execute some instructions, then stop the coroutine in between the execution and continue when you wish to. Coroutines can also switch between the threads, which gives coroutines an edge over threads.
So as you have understood Kotlin coroutines, now look at some coroutines' features.

