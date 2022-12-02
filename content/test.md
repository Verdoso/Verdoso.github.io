---
date: 2022-12-02T13:45:03Z
tags: []
featured_image: ''
title: 'On (virtual) threads and pools:  First, the problem'

---
Much has already been written about the new feature known as virtual threads, introduced as a preview in Java 19, and about the changes that this feature will enable. Among those, we can find that **the new virtual threads are light**, as opposed to the regular ones tied to native threads, so there is no need to cache or pool them. So far, so good.

However, beware! We should not make the mistake of immediately assuming that given that there is no need to pool them, we can simply substitute the “now old-fashioned” pools of threads with a new executor service that simply spawns a new virtual thread for every task as fast as it can create them. Why? Because threads being heavy was not the only reason why we are using pools of threads. In many occasions, the reason behind a pool of threads is controlling explicitly the level of concurrency that we want to allow for a certain set of tasks.

For example, to prevent a resource from being “overwhelmed” if we know it is not prepared to handle more than a certain level of simultaneous requests. Yes, it might not be the ideal approach, as we are killing two birds with one stone (controlling concurrency and preventing the creation of too many heavy native threads) but that is how many pieces of software currently do it.

With the new virtual threads, why not send all the requests anyway and let the resources handle & queue them as they see fit? Because doing so would put the control, the responsibility, and the burden at the resource. And that might not be convenient at all. The resource might not be in our control, and then it might simply refuse to do it, or it might be unable to carry those tasks; or, if it did carry them, we would lose control over how it is done, so we might not be happy with the results anyway. If the resource were in our hands, at a minimum we would need first to make sure that it is able to handle the load, or we would have to modify it so it is able to handle it.

Let’s see some examples, based on real-life code that I have had to work with, to see why **a direct translation of pools of threads to simply spawning virtual threads is not an appropriate strategy**.

We have an application that uses a pool of database connections with a max number of connections: nothing new. This pool is used by some interactive fast requests and by some background jobs that periodically fill up some cache data and for that, they run some slow queries. To control the number of slow queries that are sent against the database concurrently we use a pool of threads, for two reasons: First because sending too many of them at the same time does indeed make them slower as they interfere with each other. Second, because if we use all the connections of the database pool for the background jobs, then there is no connection left for the interactive requests to use. Then the interactive requests block waiting for the slow queries to finish and the latency suffers.

There is another application where we have to update periodically some data that queries a third party service. It has to send a couple of thousand request every day but we use a pool to throttle the requests. Why? Because we know this external service is in a shared host with limited resources and sending more than handful simultaneous requests simply increases the latency. Apart of our requests, this service has a very light load, so they do not intend to overprovision it to make it able to handle our daily requests, just so we can finish a background job faster.

I worked for some time in a B2B accommodation engine, designing the API. One of the things that we had in place was an API Gateway, to control customers who would pay a fee depending on the number of requests that they were allowed to make in a given interval of time. If they controlled the number of requests sent at their side, they could decide to queue the ones over their allotted quota. If they decided to simply send all their requests along, the API Gateway would simply block the ones over their quota, which was a loss for both parties.

Those are just a couple of examples of “not too uncommon” situations where developers use nowadays a pool of threads to control the level of concurrency. **If we simply substitute pools of threads with the new VirtualThreadPerTaskExecutor, we are going to DoS (Denial of Service attack) some our services**.

So, **does that mean we should skip Virtual threads? No**, what it means is that we should make sure that if we need to control the level of concurrency of our tasks, we keep being in control. That we can accomplish by using the good ol’ concurrency mechanisms that Java provides, for example: semaphores, and with a little code reuse, we can simplify the task and create a good candidate to replace our thread pools.

And to keep this post short that’s something I will show in the installment, already available here.