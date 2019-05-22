---
date: 2018-02-22T21:20:18+01:00
tags:
- Java
- Spring Boot
- Logging
- Web development
featured_image: ''
title: 'Spring Boot: Setting a unique ID per request'
draft: true

---
[Spring Boot](https://projects.spring.io/spring-boot/) web applications, like many others unless you go for the mono-threaded option (yuck!) serve requests concurrently. That brings many advantages, but one of the drawbacks that come with it is that it makes harder to trace what each request is doing, as it is doing it while other requests are doing other, often similar, things. That hurts logging information badly, as the messages generated for one request can easily be intermingled with other messages generated for other requests.

To alleviate this issue, the first thing we are going to do is to use a common technique that both [Logback ](https://logback.qos.ch/)and [Log4j2](https://logging.apache.org/log4j/2.0/) support: [setting a key through the Mapped Diagnostic Context (MDC)](http://www.baeldung.com/mdc-in-log4j-2-logback). As Logback mentions: “The MDC manages contextual information on a per thread basis” so once you have set the key in a thread, all the log requests performed from that thread will use that context. It is important to remark the “on a per thread basis” because the calls you perform in a different thread won’t be using that context, hence they won’t have the same key.

Setting it an MDC key is as easy as typing

_MDC.put(“key“, “value”);_

And once you have the key defined in the logging context, you can define your log formatter so it uses that key and prints it in each line. (No worries, I’ll show you how to do that in a minute.)

But let’s dig a bit further. Wouldn’t it be nice that if clients sending requests would be able to tell you “the request that failed had the key xxxxx”. You would then be able to filter your logs with that key and go straight to the lines that you need to check. Cool, huh? That works not only for clients but for proxies, gateways, interceptors and other mechanisms that you might be using to analyze your responses. For that, what we’ll do is add an HTTP header to the response with the ID we used for all the processes producing that response.

Finally, let’s go the extra mile. Sometimes, if you are using proxy-gateway services or your application is one service called by other services, the request itself might already have and ID assigned. That ID, sometimes called correlation ID, serves the purpose of identifying the various requests performed at the various services to serve one final client request. A common mechanism to provide that ID in the world of web applications is through an http header, so what we’ll do is check if there is a header providing the correlation ID, and if that’s the case, use it instead of generating a new ID.

Ok, now let’s do that in Spring Boot: We’ll need a configuration class that creates a FilterRegistrationBean with our servlet. We’ll also use Spring Boot’s great configuration capabilities and let the user configure the names of the headers involved and the key of the MDC, while providing some defaults. In the end, the configuration class might look something like this:

{{< gist Verdoso 7e11004f0ee73efc6e9b1ce3527a6022>}}

We can then create the filter itself. We need to verify if we received a correlation ID in the request header, if specified, or else create a new ID. Set the ID with the right key at the MDC and add the response header, if necessary, and make sure we are cleaning up the MDC key correctly once the request is done.

All in all, that’s how the filter might look like:

{{< gist Verdoso 1d609e6d621a9cc4cb1bff6bcbf65ef9>}}

You can then configure your logs with a pattern like this one(in this example using Log4j2) where we add the MDC key (_Slf4jMDCFilter.UUID_ in this case)

![](/uploads/log4j_pattern.png "Log4j2 pattern")

And your log messages will look something like these, where the UUID for the request is the second field after the date:

![Log messages with a unique ID added](/uploads/log_messages.png "Log messages with a unique ID added")

Now imagine you launch a battery of tests against your service using [Postman collection runner](https://www.getpostman.com/docs/postman/collection_runs/starting_a_collection_run), for example. If you wanted to identify which log lines correspond to each request, you would just need to check the headers of the response and look for the one including the ID (_Response_Token_ in this case).

![](/uploads/postman.png)

and with that ID you can browse the logs and identify exactly which are the log messages that correspond exactly to that request:

![](/uploads/log_lines.png)

Log lines identified by the corresponding ID

The request header can also be pretty useful if you are doing some test requests and you don’t want really want a unique ID per request but a way to distinguish the log messages from your requests from the other log messages.

For example, you could configure a request header like this (remember that by default we set no request header):

![Setting a request header to check for a correlation ID](/uploads/request_header.png "Setting a request header to check for a correlation ID")

set the request header like this in Postman:

![Setting the ID in a request header](/uploads/setting_id_request_header.png "Setting the ID in a request header")

and your log messages would look something like this:

![Setting a fixed correlation ID for testing](/uploads/correlation.png "Setting a fixed correlation ID for testing")

The ID is also a perfect candidate to parse in centralised logging platforms ([Elastic Stack,](https://www.elastic.co/elk-stack) [Graylog](https://www.graylog.org/)…) so you can filter log messages directly for that field.

Just a friendly reminder: if you launch an asynchronous call from your code serving a request, the log messages produced from the code executed asynchronously won’t have the same ID so you will “lose” them unless you don something. The best solution I have found so far to that is to pass the value explicitely to the asynchronous code and do there the same thing the filter does: Set the ID at MDC, perform the task, clear the MDC.

You can find a complete working implemention of the code demonstrated in this entry here:

The filter configuration: [Slf4jMDCFilterConfiguration](https://github.com/Verdoso/GreenSummer/blob/master/summer-core/src/main/java/org/greeneyed/summer/config/Slf4jMDCFilterConfiguration.java)  
The filter: [Slf4jMDCFilter](https://github.com/Verdoso/GreenSummer/blob/master/summer-core/src/main/java/org/greeneyed/summer/filter/Slf4jMDCFilter.java)

Happy coding and happy logging as well!

D.

Edit: If you need to go full “enterprisey” with correlation IDs and distributed tracing, I would recommend having a look at [Spring Cloud Sleuth](https://cloud.spring.io/spring-cloud-sleuth/).