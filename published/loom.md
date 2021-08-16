2021.06.28

### Why project Loom will make Java a better cloud language than Go  &nbsp; <img src="https://state-of-the-art.org/graphics/chips/chips-32.png" srcset="https://state-of-the-art.org/graphics/chips/chips-32-2x.png 2x" style="vertical-align:baseline"/>

At first glance, the *Go* programming language in 2021 looks and feels to me a bit like garbage collected C. It *isn't* for a number of reasons, but it is certainly more of a systems programming language than a pure applications programming language like Java. Putting aside lots of small features and improvements, we've seen most of what Go has to offer before. So *why are so many people using Go to write cloud applications*? The main reason is that the cloud is highly concurrent and Go has a killer feature for concurrency: *go routines*.

By simply putting the word *go* in front of any statement in Go, you can execute that code concurrently. This is a neat feature itself, but the big deal here is that this concurrent execution occurs on a *lightweight thread* that is scheduled by the Go runtime rather than by the kernel. When a go routine reaches certain points in the code, such as entering a function or blocking on I/O, the Go runtime can cause the go routine to yield execution to another go routine that is associated with the same underlying kernel thread. To do this, Go saves the stack of the go routine that is yielding in a *continuation* data structure and restores the stack of whatever go routine is being scheduled to execute next.

Why is this so important? Because kernel threads are an expensive and limited resource. They have relatively large stacks and context switching between them is expensive (particularly between threads running in different processes). By comparison, Go's lightweight user-mode threads are very cheap. They have low memory overhead and context switching between them is very fast when compared with kernel mode time-slicing. This allows the creation of *millions* of lightweight threads with little performance degradation. By comparison, only a few thousand kernel threads will cause your OS to grid to a halt.

In the world of the cloud, where huge numbers of requests are typically blocked on I/O talking to users or other cloud resources at any given time, being able to schedule all those concurrent requests efficiently is crucial. This is only possible at the present time in Java by using limited thread pools that schedule and perform asynchronous operations for client threads. This asynchronous style of coding is complex, ugly, error-prone and very difficult to debug, so being able to do this with the *go* keyword is a huge advantage today.

Enter project Loom.

[Project Loom](https://blogs.oracle.com/javamagazine/going-inside-javas-project-loom-and-virtual-threads) is Java's answer to lightweight user-mode threads, and it will likely make Java very competitive with Go as a cloud language. Loom's lightweight threading model is designed to be fully compatible with Java's existing threading model, so you won't have to learn anything to use it. This feat has been accomplished by integrating Loom's *virtual threads* with *java.lang.Thread* and *java.util.concurrent.Executor*. The only different between kernel threads and Loom's virtual threads are how threads are initially created.

A thread builder in the *Thread* class will allow threads to be created as virtual. With this functionality, Java can easily mimic the functionality provided by the *go* statement in Go:

    public static Thread go(Runnable code) 
    {
        return Thread.builder()
                     .virtual()
                     .task(code)
                     .build()
                     .start();
    }

Usage of this utility method then looks like this:

    go(() -> launchSpaceShip(cargo));

In addition, Java will add seamless support for *structured concurrency*, where sub-tasks can be executed with the parent thread waiting for them to complete, all in one nice try-with-resources statement:

    try (var executor = Executors.newVirtualThreadExecutor()) 
    {
        executor.submit(this::warmUpEngines);
        executor.submit(this::stabilizeCoolBlinkingLights);
    }
    
    launchSpaceShip();
    
Here, the two submitted tasks will execute on virtual threads, and when both have completed, the *AutoCloseable.close()* method invoked by the try block will return and the spaceship will be launched. We could even make a utility method to reduce this code to the minimum:

    public static void run(Runnable... runnables)
    {
        try (var executor = Executors.newVirtualThreadExecutor()) 
        {
            for (var code : runnables)
            {
                executor.submit(code);
            }
        }
    }
    
This method allows our code is reduced to just this:
    
    run(this::warmUpEngine, this::stabilizeCoolBlinkingLights);
    launchSpaceShip();

Right now, Go is a reasonable choice for cloud development, but when Project Loom does arrive (when will this be?), Java will have Go's killer cloud feature. It will also have a much bigger ecosystem, excellent tooling and, in my opinion, better readability. Many people will stick with Go when this happens, but there will be no good reason at that point not to prefer Java for new projects.

<img src="https://www.kivakit.org/images/horizontal-line-512.png" srcset="https://www.kivakit.org/images/horizontal-line-512-2x.png 2x" />

Questions? Comments? Tweet yours to @OpenKivaKit or post here:

<script
  async
  src="https://utteranc.es/client.js"
  repo="jonathanlocke/jonathanlocke.github.io"
  issue-term="loom"
  theme="github-dark"
  crossorigin="anonymous"
></script>
