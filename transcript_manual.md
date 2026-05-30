# Slide 0 - Title

Hello everyone and welcome to my talk. I'll be talking about structured concurrency today and
more specifically, I'll be comparing CoroutineScope and StructuredTaskScope.
They are both available on the JVM, with the first one available only from Kotlin and
the second one both from Kotlin and Java. So in order to have a proper comparison, I'll be using Kotlin today.

# Slide 1 - About me

A bit about me: My name is Filip and I currently work as a Software Engineer at Clue.
Clue is a period tracking app with about ten million monthly active users in about a hundred and ninety countries.
In the past 5 years or so, I've been mainly working with Kotlin, Spring, and Ktor.
And here on the slide, are some links where you can find me online.

# Slide 2 - The idea

So what is the idea behind structured concurrency? The main idea is that every task is bound to its parent or scope, in a similar way
that a variable is bound to a block for example. This means that no task can outlive the parent scope, that errors in tasks propagate to the parent scope, 
and that cancelling a scope, cancels all of its children.

# Slide 3 - Demo

Now, all this may seem a bit abstract so we'll look at some code to hopefully make it more concrete.

# Live coding

## BooksController

Here's our example. We have a bookstore and an endpoint that returns a book with reviews. The example is very straightforward: 
We find the book in the database, we fetch the reviews for the book from some third party service, and then we combine it all together and return it.
Both of these calls take about one second to complete, because we simulate a delay here, and because we run them sequentially, 
the entire endpoint takes two seconds to return. And we can see this if we try to call the endpoint.
Here I have the service running and if I call the endpoint here, I get the response after about two seconds, and in the logs we can see how it was executed.
We see START log from the controller, then we see START from the repository and after about a second we see DONE from the repository. After that we 
start fetching the reviews, which also takes a second, and then finally we see DONE from the controller.

Now if we take a closer look in the controller, we can see that these two calls are completely independent and we don't really need to run them sequentially.
We could run them concurrently and that way hopefully make the entire endpoint faster.

If I were to google: "two calls in parallel in kotlin", I would most likely get a result for coroutines. Coroutines are a default when it comes to concurrency 
in Kotlin. So let's see how we can use them in our example here.

Coroutines have two main components, one of them is part of the language and the other is part of the coroutines library. The one that's part of the language 
is the `suspend` keyword. `suspend` is a function modifier, and it basically tells the compiler that this function can "pause" its execution and then continue it
at some later point. Under the hood, the compiler converts that to a callback somehow, and we don't need the details today. The other part comes from the library, and that's the coroutines builders.

One of the most used ones is `coroutineScope` and we'll try to use it here. Since we have a lamda here now, we have to make this an implicit return. 
So the idea is that we can run these two calls asynchronously and that way 
make the entire endpoint faster and the way to do that is with `async` coroutine builder.

`async` returns a `Deferred` (think `Future` or `Promise`) so now we have to `await` these two values before we can return them.

How does this work exactly? In the lambda here from `coroutineScope`, we have an implicit receiver `this` which is of type `CoroutineScope` and the `async` 
function is actually an extension function on `CoroutineScope`. This is a very clever design, and it's actually what enforces structured concurrency.
We can create and start new coroutines onlt from a scope and the lifetime of those coroutines is bound to the scope.

So let's try to call the endpoint. Oops, it doesn't work. We get a `ClassNotFoundException`. Apparently we are missing `Publisher`.
Why would we need that? Because `coroutineScope` is a suspend function, our controller method had to become `suspend` too, and Spring MVC adapts suspend handlers
through a reactive `Publisher` under the hood, which the webmvc starter doesn't ship.
If we take a look in `build.gradle.kts`, we can see that we are using `webmvc`, which is the blocking variant of Spring.
If we were using `webflux`, `Publisher` would be included by default and this would work fine. So what are our options here:
We don't want to migrate to `webflux` because of just one endpoint, so instead we can use `runBlocking` and in that case we no longer need `suspend` here.
`runBlocking` is a regular, non-suspend function that blocks the calling thread and bridges into the coroutine world, so our controller method goes back to being
an ordinary blocking function, and Spring MVC is happy again.

If we now try to run this, it works, but it still takes 2 seconds. Let's try to understand why that happens by reading the documentation for `runBlocking`.

This part is important:
`The default CoroutineDispatcher for this builder is an internal implementation of event loop that processes continuations 
in this blocked thread until the completion of this coroutine.`

This means that all of our coroutines will run on the same thread, and since both of these IO calls are blocking, we still get a sequential behavior.
How do we fix this? Well, again, by reading the documentation. It says here:

`When CoroutineDispatcher is explicitly specified in the context, then the new coroutine runs in the context of the specified dispatcher 
while the current thread is blocked.`

So what we have to do is, specify the Dispatcher explicitly, and an appropriate dispatcher here is the `IO` dispatcher: it's a large pool sized for blocking I/O,
unlike the single-threaded `runBlocking` event loop.
If we call the endpoint again, we now see that it takes only one second to complete and in the logs we can see that our two calls run concurrently
and on different threads.

With this we've achieved concurrency, but have we achieved structured concurrency? We have an easy way to check this: In the `properties` file, 
we can increase the delay for the repository to three seconds and make the reviews service fail.
What we expect to happen in this case is that after one second the reviews service will fail, the database call will be canceled, 
and we'll get an error on the client. So let's try it out...

Not quite what we wanted. We get an error, but only after three seconds, and the only thing worse than an error is a slow error.
So why does this happen and how to fix it?

It happens because we're mixing blocking and non-blocking code. When the reviews service fails after one second, the error is propagated to the parent scope, 
and it then tries to cancel all the children, but since this `findBook` is blocking, it cannot be canceled, it can only be interrupted, and that's why we 
wait for two more seconds before we see an error. If we check the logs, we can see `DONE` from the repository, which means it was never canceled.

How do we fix this?
We need to wrap these blocking calls in `runInterruptible` and then the compiler also makes us add the `suspend` keyword.
If we check the documentation, it says: 

`Calls the specified block with a given coroutine context in an interruptible manner. 
The blocking code block will be interrupted and this function will throw CancellationException if the coroutine is cancelled.`

Let's try it again...

Now we get an error after one second and in the logs, we don't see `DONE` from the repository, which means it was properly canceled.
This is exactly properties #2 and #3 from the intro, errors propagating to the parent and a cancelled scope cancelling its children, happening live.

And that's it, we've achieved what we wanted. We run the two calls concurrently and we have proper error handling and cancellation.

However, there were quite a few things to remember:

We can't use `coroutineScope` and we have to use `runBlocking`, we have to remember to pass the dispatcher explicitly, and we have to remember
to wrap all our blocking calls in `runInterruptible`, otherwise we don't get proper cancellation.

So is there an easier way maybe?

Let's copy this controller and make it a version `2`.

There's a new API for structured concurrency in Java called `StructuredTaskScope`. We can see the JEP here and how it's used from Java.

Here in the Java example, it's used within `try-with-resources` which makes all subtasks confined to the block, and the scope has this `fork` method, 
which is similar to the `async` we used earlier. One thing that is different is that now we have to call this `join` method which 
awaits for all "forks" to complete. After that we can access the values from subtasks by using this `get` method.

So let's see how we can use this from Kotlin.

In Kotlin, we have this `taskScope` builder function, which is a small Kotlin helper I wrapped around the raw Java API, and I'll show what's inside in a moment.
Instead of `async` we use `fork`, and instead of `await` we use `get`.
Each `fork` runs on its own virtual thread, so blocking is cheap, and cancellation works through real thread interruption natively, exactly the machinery
`runInterruptible` had to bolt onto coroutines. That's why everything here can just be blocking, and we no longer need `suspend` and `runInterruptible`.

And one final thing that we mustn't forget is to call the `join` method.
The default behavior of the scope is to wait for all subtasks and shut down on the first failure, so when the reviews call throws, the scope interrupts the
sibling subtask's virtual thread. That's also why `join` is mandatory and why `get` only works after it.

Let's try running this...

The happy path works. We see that it takes one second and in the logs we can see that the IO calls are executed on different virtual threads.

Let's see if the cancellation works as we would expect...

It does, we get an error after one second and in the logs we don't see `DONE` from the repository which means that it was properly canceled.

Let's see both examples side by side now...

It looks similar and there are pros and cons of both approaches.
With coroutines we have to remember to use the right dispatcher and to wrap all blocking calls in `runInterruptible`.
With `StructuredTaskScope` we have to remeber to call the `join` method.

Now a small disclaimer about the example on the right here: The `taskScope` builder doesn't really exist. At least not yet, 
hopefully the Kotlin team will add it at some point.
It's a wrapper that I created to make the code more Kotlin idiomatic.
We have to remember that the `StructuredTaskScope` API is still in preview even in Java, so Kotlin still hasn't developed anything around it.

It's a small price to pay in the end, for not having to deal with some of the problems inherent to coroutines and non-blocking code.

That's all I had for today. Thank you very much for your attention and if you have any questions, I'll gladly try to answer them.

