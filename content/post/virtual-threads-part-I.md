---
date: 2022-12-02T13:45:03Z
tags:
- java
- virtual threads
featured_image: "/uploads/loom-g85da6e78d_640.jpg"
title: 'On (virtual) threads and pools'

---

Much has already been written about the new feature known as virtual threads, introduced as a preview in Java 19, and about the changes that this feature will enable. Among those, we can find that the {{< highlighted option="yellow">}}new virtual threads are light{{< /highlighted >}}, as opposed to the regular ones tied to native threads, {{< highlighted >}}so there is no need to cache or pool them{{< /highlighted >}}. So far, so good.

{{< annotation >}}Native threads are considered heavy, so you can create a limited number of them, but virtual threads are light so they are virtually, pun intended, limitless.{{< /annotation >}}

However, beware! We should not make the mistake of immediately assuming that given that there is no need to pool them, we can simply substitute the “now old-fashioned” pools of threads with a new executor service that simply spawns a new virtual thread for every task as fast as it can create them. Why? Because {{< highlighted >}}threads being heavy was not the only reason why we are using pools of threads{{< /highlighted >}}. 

In many occasions, the reason behind a pool of threads is controlling explicitly the level of concurrency that we want to allow for a certain set of tasks. For example, to prevent a resource from being “overwhelmed” if we know it is not prepared to handle more than a certain level of simultaneous requests. Yes, it might not be the ideal approach, as we are killing two birds with one stone (controlling concurrency and preventing the creation of too many heavy native threads) but that is how many pieces of software currently do it.

With the new virtual threads, why not send all the requests anyway and let the resources handle & queue them as they see fit? Because doing so would put the control, the responsibility, and the burden at the resource. And that might not be convenient at all. The resource might not be in our control, and might simply refuse to do it, or be unable to carry those tasks; or, if it did carry them, we would lose control over how it is done, so we might not be happy with the results anyway. If the resource were in our hands, at a minimum we would need first to make sure that it is able to handle the load, or we would have to modify it so it is able to handle it.

{{< annotation >}}Simply spawning virtual threads puts the control, the responsibility, and the burden of handling concurrency at the resource on the other side of the task.{{< /annotation >}}

Let’s see some examples, based on real-life code that I have had to work with, to see why {{< highlighted >}}a direct translation of pools of threads to simply spawning virtual threads might not be an appropriate strategy{{< /highlighted >}}.

1. We have an application that uses a pool of database connections with a max number of connections: nothing new. This pool is used by some interactive fast requests and by some background jobs that periodically fill up some cache data and for that, they run some slow queries. To control the number of slow heavy queries that are sent against the database concurrently we use a pool of threads, for two reasons: First, because sending too many of them at the same time does indeed make them slower as they interfere with each other. Second, because if we used all the connections of the database pool for the background jobs, then there would be no connection left for the interactive requests to use. Then the interactive requests would block waiting for the slow queries to finish and the latency would suffer.

2. There is another application where we have to update periodically some data that queries a third party service. It has to send a couple of thousand request every day but we use a pool to throttle the requests. Why? Because we know this external service is in a shared host with limited resources and sending more than a handful simultaneous requests simply increases the latency. Apart of our requests, this service has a very light load, so they do not intend to overprovision it to make it able to handle our daily requests, just so we can finish a background job faster.

3. I worked for some time in a B2B accommodation engine, designing the API. One of the things that we had in place was an API Gateway, to control customers who would pay a fee depending on the number of requests that they were allowed to make in a given interval of time. If they controlled the number of requests sent at their side, they could decide to queue the ones over their allotted quota. If they decided to simply send all their requests along, the API Gateway would simply block the ones over their quota, which was a loss for both parties.

Those are just a couple of examples of “not too uncommon” situations where developers use nowadays a pool of threads to control the level of concurrency. {{< highlighted >}}If we simply substitute pools of threads with the new VirtualThreadPerTaskExecutor, we are going to _DoS_ (Denial of Service attack) some our services{{< /highlighted >}}.

{{< annotation >}}We have to be careful when moving to virtual threads, else we might _DoS_ some of the services in use!{{< /annotation >}}

So, **does that mean we should skip virtual threads? No**, what it means is that we should make sure that if we need to control the level of concurrency of our tasks, we keep being in control. That we can accomplish by using the good ol’ concurrency mechanisms that Java provides, for example: semaphores, and with a little code reuse, we can simplify the task and create a good candidate to replace our thread pools. In the next section I'll show an example of how we can implement such a tool.

## Controlling the level concurrency with virtual threads
We have talked about why it is important to make the migration to virtual threads without losing control of how much load we are causing with our tasks. Now, we will explain how to do so, using the concurrency mechanisms provided by the language itself.

There are many solutions one could employ to limiting the concurrency of tasks being executed in Java, but we are going to use the class _[java.util.concurrent.Semaphore](https://docs.oracle.com/en/java/javase/19/docs/api/java.base/java/util/concurrent/Semaphore.html)_, as it allows us to directly specify the number of threads we want to be able to execute a piece of code at the same time. We could have done something similar to the class _[java.util.concurrent.ThreadPoolExecutor](https://github.com/openjdk/jdk/blob/master/src/java.base/share/classes/java/util/concurrent/ThreadPoolExecutor.java)_ and keep the scheduled tasks in a queue, but in this case we opted for more straightforward approach, and we let all the virtual threads be spawned and simply use the semaphore to control how many of them are executing the real work at the same time. Given that virtual threads are supposed to be light, we also wanted to check it by letting them all spawn at the same time.
You can see one possible implementation that I uploaded to GitHub: [ThrottledVirtualThreadsExecutor](https://github.com/Verdoso/jmh-throttled-virtual-threads/blob/master/src/main/java/org/greeneyed/jmh_throttled_virtual_threads/ThrottledVirtualThreadsExecutor.java)

{{< annotation >}}Java includes mechanisms to control the concurrency of our programs, and we can apply those same mechanisms to virtual threads.{{< /annotation >}}

### Testing CPU bound tasks
We have already mentioned that the problem with simply spawning virtual threads without control is the danger of overwhelming other services, but what about CPU bound tasks? I wanted to test how well virtual threads were performing when stressing the CPU and see if throttling them would make any difference, so I created and uploaded a sample project to GitHub, [jmh-throttled-virtual-threads](https://github.com/Verdoso/jmh-throttled-virtual-threads). I used that code to simulate creating many threads to perform a CPU consuming task by executing some floating-point arithmetic (see the method _[VirtualThreadsTester.cpuConsumingMethod](https://github.com/Verdoso/jmh-throttled-virtual-threads/blob/master/src/main/java/org/greeneyed/jmh_throttled_virtual_threads/VirtualThreadsTester.java#L67)_). In order for the experiment to be more reliable, I used the [JMH benchmarking framework.](https://github.com/openjdk/jmh) as it launches the same tasks repeatedly and adds a warm up phase to prevent some of the issues that plague short-lived microbenchmarks.

I have performed the tests executing just the CPU consuming function and repeated them adding a small delay of 50ms to simulate some I/O task or other non-CPU related factor. The test launches 10.000 virtual threads to perform the calculation and the maximum level of concurrency allowed for the throttled executor is set to 30 simultaneous tasks.

After running some tests, the result are interesting:
#### Pure CPU function
```
Benchmark                                   Mode  Cnt      Score    Error  Units
testFreeVirtualThreadsWithLongFactor        avgt   25  21764,275 ± 58,243  ms/op
testFreeVirtualThreadsWithSmallFactor       avgt   25     34,053 ±  0,197  ms/op
testThrottledVirtualThreadsWithLongFactor   avgt   25  21759,747 ± 66,738  ms/op
testThrottledVirtualThreadsWithSmallFactor  avgt   25     34,125 ±  0,104  ms/op
```
In this test, you can see that there is no difference between controlling concurrency or not. In fact, if you run the tests individually, grain of salt applied, you can see that with the small factor, the level of concurrency obtained is within a single digit so the throttling never limits anything. On the other hand, when the factor is big, the CPU consumed is so huge that it becomes the limiting factor before the throttling has any chance to limit it (30 is too big). Playing with the concurrency limit and the load factor, I was unable to come up with a combo that gets an unambiguous better result when throttling, and that's good because it means that the JDK/OS combo are really good at handling the tasks/CPU.

But there is more
#### CPU + small delay (50ms) function
```
Benchmark                                   Mode  Cnt      Score     Error  Units
testFreeVirtualThreadsWithLongFactor        avgt   25  22036,852 ± 134,104  ms/op
testFreeVirtualThreadsWithSmallFactor       avgt   25     95,549 ±   0,062  ms/op
testThrottledVirtualThreadsWithLongFactor   avgt   25  23505,818 ±  43,687  ms/op
testThrottledVirtualThreadsWithSmallFactor  avgt   25  20617,513 ±  11,155  ms/op
```
Adding a small delay, 50ms, after the calculation changes things a bit. When the CPU is not the constraint (small factor), then limiting the number of simultaneous tasks via throttling makes a big difference, for the worst. Checking the logs during the experiment showed us that with the small delay, the level of concurrency without throttling gets to 10.000, remember that without the delay it would not reach two digits. Given that the concurrency limit is set to 30 in the throttling experiment, it is clear where the delay comes from

Playing with the parameters (the delay, the concurrency limit) we can see different results but the overall conclusion is that when the tasks are CPU-bound, {{< highlighted >}}the new virtual threads and the JDK do a very good job when we let them handle everything{{< /highlighted >}}. If we wanted to remain in control and make sure we don't simply consume as much CPU as available, we know now one way to do it.

{{< annotation >}}JDK/OS handle pretty well CPU bound tasks, but don't let them consume all your resources{{< /annotation >}}

## Some conclusions
Taking into account that this different results happen with exactly the same code by simply changing one or two variables. It is quite clear, in case there was any doubt, that {{< highlighted >}}the best approach is not to follow one strategy blindly but to measure and see what will work best given the circumstances in each case{{< /highlighted >}}, and then be ready to change if those circumstances change.

A **conservative approach to move to virtual threads** would be to replace all the “_Executors.newFixedThreadPool(int)_” calls with something like “_new ThrottledVirtualThreadsExecutor(int)_” ones and then play with the concurrency level to see if we can handle more. If our tasks are CPU bound it is very likely that we will be able to increase the concurrency level, no more native threads limit yay!, but be careful not to consume the whole CPU if that causes issues with other parts of your application (or the GC, for example). Be also aware that if the CPU consumption is high, spawning all tasks might not give you much better results, as the CPU has not changed. At the very least, your application will be a tad lighter as virtual threads are indeed lighter than regular ones.

Remember that, as there are limited resources, there is no such thing as THE bottleneck of an application, but just the CURRENT bottleneck. If we eliminate the current one, another one will surface so don't let the next one in line catch you unprepared.

### Future experiments
One could easily perform different experiments by changing the cpuConsumingMethod method and fiddling with the parameters or doing some other task, like some I/O intensive operation or querying a database. For example, using this project as a template, one could check if sending 10.000 simultaneous queries to a DB gives better performance than throttling them and, in that case, see which concurrency level get the better results). Be careful when doing this type of tests though, never in production, as you might _DoS_ your DB. Monitoring the DB would be important, as your service might respond fine but might be making the DB unusable for anything else.

If you do your own tests and see some interesting results, please share them!

(Image by [Anna](https://pixabay.com/users/annaer-35513/?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=173821) from [Pixabay](https://pixabay.com//?utm_source=link-attribution&amp;utm_medium=referral&amp;utm_campaign=image&amp;utm_content=173821))