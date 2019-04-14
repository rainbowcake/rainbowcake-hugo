+++
title = "Threading"
weight = 20
+++

At this point, you know we're using coroutines as our threading solution. But how exactly are we going to do that?

![Threading between the layers](/images/arch_threading.png)

As mentioned before, ViewModels are going to start coroutines when Fragments call them. They'll start these in the UI dispatcher (i.e. on the UI thread), so that they can easily set the state in the `LiveData` they'll be storing.

As soon as these calls hit the Presenter layer, they'll be moved to the IO dispatcher, so everything below this point will run on a background threadpool. This means that in the lower layers we may perform blocking calls freely, because they'll just block one of these background threads.

One approach may be to make all calls from the Presenter downwards blocking, and this is the easiest thing to do. Just write non-suspending functions in the Interactors and in the Data sources - this shouldn't be hard, as most APIs are blocking by default. If you do this, you'll only be using coroutines to easily hop between the UI and IO threads. 

However, if you want, you can propagate the suspending nature of these calls down into the lower layers. You just need to mark the functions along the way with the `suspend` keyword, through the Interactors, and in the Data sources. If you do this, the lower layers will be aware that they are running inside coroutines. What are the benefits of this?

- You can support cancellation better. If all calls are blocking in the lower layers, cancelling the coroutine will only cut off execution when your calls return to the suspending parts of your code, the continuous blocking parts will always be all executed together.
- You'll be free to easily switch the thread your code is executing on. For example, you might have a data source which wants to have its own single thread wrapped in a `CoroutineContext`. If its interface has all suspending functions, it can move all calls it receives to that single thread very easily with a `withContext` call at the start of each of its public methods.
- You might use advanced coroutine features in lower layers to implement either business logic or data sources. 

In general, we probably want to make all of our Interactor methods suspending, as there's at least one Data source in the application that supports coroutines. This way, blocking data sources will block whatever background thread the call is currently on, and suspending data sources can move computations to their own dispatchers if they need to. 

A network data source can easily provide a suspend API [using Retrofit](/content/datasources#retrofit-and-coroutines).
