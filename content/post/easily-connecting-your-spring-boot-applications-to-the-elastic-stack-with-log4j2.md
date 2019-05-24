---
date: 2019-02-25T18:24:54+01:00
tags:
- Elasticsearch
- Spring Boot
- Log4j2
featured_image: ''
title: Easily connecting your Spring Boot applications to the Elastic Stack with Log4j2
draft: true

---
**N**ow that logging aggregators are becoming ubiquitous, a common issue that we, developers, have to face is how to connect the log output of our applications to those tools. Some of the issues that we encounter during that process are:

· Usually we don’t use **an aggregator in development**, we simply want to see all the logs in the development console, so the application has to start without issues when there is no aggregator listening.

· One does not simply throw the logs to Mordor… I mean the aggregator. **We need to parse the different fields of the log lines so the messages can be easily filtered afterwards**.

· **The aggregator might go down**, for some reason, and then it would be nice not to lose the logs completely.

· On the other hand, we have to be **careful with the size of the logs generated in the machine where the app is deployed**, as space is usually a concern, especially now that multiple small machines are now the norm.

As people used to store the logs in files the production machines, and then move/copy/process those files, a very common approach is to use one of the beats of the [Elastic Stack](https://www.elastic.co/products) family, [Filebeat](https://www.elastic.co/products/beats/filebeat), to read those files and send them up the stack. That has the advantage of being simple and not requiring many modifications to our previuos flow, but it means we have now to process all those log lines in [Logstash](https://www.elastic.co/products/logstash) and extract the different fields so the logs can be indexed properly. If you have had to create a 380-char regular expression to extract important fields from log lines, as I had to do, you know how “fun” that can get. We could also generate the log fields in JSON format and then send that to prevent the line parsing, but the logs files become then non human-readable.

Fortunately, there’s a very useful set of libraries from [Mark Paluch](https://github.com/mp911de) that one can use: [Logstash/Gelf Loggers](http://logging.paluch.biz/). More specifically, I’m going to explain how to use the [Log4j2](https://logging.apache.org/log4j/2.x/) one to connect the application directly to [Logstash](https://www.elastic.co/products/logstash). It communicates using a the Graylog Extended Logging Format, so there’s no need to parse the log lines in [Logstash](https://www.elastic.co/products/logstash), unless you want to.

So let’s start:

#### Preparing the environment

As I said before, we don’t want to try to connect to the log aggregator during development, so we create two different [Log4j2](https://logging.apache.org/log4j/2.x/) configuration files: log4j2-spring.xml and log4j2-spring-prod.xml. The first one is the typical Log4j2 development configuration, and the second is the one that will be used in production. How do we tell [Spring Boot](https://spring.io/projects/spring-boot) that we want to use one or the other depending on the environment? Well, that’s what profiles are for! Edit **src/main/resources/application.yml** and add a piece like this one at the end:

{{< gist Verdoso 0073b2b80f2976241b1f48108cd80155>}}

This basically tells Spring Boot that the default logging configuration file is log4j2-spring.xml, found in the classpath. We also say explicitely that in test and devel we want to use that one, and then that in prod we want to use log4j2-spring-prod.xml. Why do we tell test and devel that we want to use that one explicitely? Because if we run some tests using the production configuration but overriding some values, we might start the application with the profile _prod,test_ and if we did not tell Spring Boot again that we want to use log4j2-spring.xml in test, it would use log4j2-spring-prod.xml. This way we are safe that the latest active profile is the one that sets the logging configuration file to use. If we have other environments in the pipeline, with or without aggregators, changing the settings to decide which configuration to use in each environment is fairly easy.

Before delving into the configuration files, let’s make sure we have the Logstash/Gelf Logger library added to the project by editing the **pom.xml** and adding this dependency:

{{< gist Verdoso bc12b0447e0478c655cc8abca98d97c4>}}

Version 1.11.2 is the one I tested the code with, but you can check what is the latest version at [Maven central](https://mvnrepository.com/artifact/biz.paluch.logging/logstash-gelf).

Once we have the library, we can define the logging configuration files. For the sake of completeness, we’ll also show the development one:

#### log4j2-spring.xml

{{< gist Verdoso 0fad07d6c49a889426ac7a1f1ba91019>}}

Of course this is a basic example and a real one will have more filters and the pattern might be different but you get the idea. Now let’s see the “production” one:

#### log4j2-spring-prod.xml

{{< gist Verdoso 843984e61a0fa7093253d4f9b1751664>}}

In this file we are, in fact, defining two appenders: one sends the logs directly to the [Elastic Stack](https://www.elastic.co/products) and the other one is a typical file appender that rolls the files quite aggressively. Why? Because if the [Elastic Stack](https://www.elastic.co/products) fails for some reason, we can still access the logs, stored in the files, for some time, before they are overwritten.

The settings for how long you want to keep the “backup” files depend on the situation and the application and is something to agree with the production guys. In any case, the interesting bit is the Gelf appender. With that one, we are sending the logs directly to the [Elastic Stack](https://www.elastic.co/products) and defining the fields we want to send. The appender has some default fields that are sent if you don’t specify anything, check the documentation for details, but I always build on top of that. In this case I’m sending:

· **Timestamp**: essential

· **Log level**: essential

· **Simple class name** producing the log: I prefer it over the lengthy fully qualified name

· **Class name**: I send it, just in case, but I’ve never used it so I have it filtered out in [Logstash](https://www.elastic.co/products/logstash).

· **Hostname** and **simple hostname**: The same as with class name, simple name is usually good enough and is shorter.

· **Application name** so I can filter easily and also so I can have different settings in [Elastic Stack](https://www.elastic.co/products) depending on the application

· Thanks to **includeFullMdc=”true”**, all the fields added to Log4J Mapped Diagnostic Context (MDC) will be added as fields to the log. This feature is very useful as, for example, it means that the token-per-request ID described in the entry “[Spring Boot: Setting a unique ID per request](https://medium.com/@d.lopez.j/spring-boot-setting-a-unique-id-per-request-dd648efef2b)” is added automatically, isn’t it cool?

### Configuration and environment variables needed

If you check the production file, you’ll see that there _baseDir_, the directory where the log files are stored is set to the logs directory at the home of the user running the application ($${env:HOME}/logs). Change that if you want to store the logs elsewhere.

Also, the address of the machine where [Logstash](https://www.elastic.co/products/logstash) is defined is passed through an environment variable _LOGSTASH_PROXY_ because we want to be able to change the [Logstash](https://www.elastic.co/products/logstash) server on different environments and we don’t want to redeploy all the applications when the [Logstash](https://www.elastic.co/products/logstash) server address changes, but it is up to you.

The _applicationName_ variable is something that I have to unfortunately write explicitely in the configuration file, because Log4j2 is configured before Spring Boot has finished starting and I have not found a way to get Log4j2 to use a JVM property. If someone can find a better way to do it, I’m all ears.

### Logstash configuration

In order to be able to accept requests from the library, you have to define a [Logstash](https://www.elastic.co/products/logstash) pipeline with a configuration similar to this one:

#### gelf-pipeline.conf

{{< gist Verdoso d195a6cba5e17ff49a5647ebccd43b6a>}}

We are using TCP instead of UDP because this way we can load-balance the [Logstash](https://www.elastic.co/products/logstash) automatically with [Fabio](https://github.com/fabiolb/fabio) and it does not support UDP yet, but if you don’t have that restriction, UDP is fine.

After that, we are removing some of the fields that we never use (source_host, className, facility), so they don’t consume space in the indexes, and then set the application name just in case it is not being sent, because we need it in the next step.

Finally, we define the index based on the application name and the date. Why? Because ElasticSearch index management, usually through the [curator utility](https://github.com/elastic/curator), is based in patterns against the names, and this way we can, for example, keep the logs of one application for 30 days and the logs of another application for just a week. [Elastic Stack](https://www.elastic.co/products) introduced some index lifecyle management options in 6.6, but I still find it easier to manage them through the curator.

As the comments in the configuration file tell you, if you uncomment the stdout output, you will be able to see in the console where you launched [Logstash](https://www.elastic.co/products/logstash) what it is received and then sent to elastic (or better yet, comment the elasticsearch section until you are comfortable with what you are seeing). That’s pretty useful if you want to test if it really works by installing [Logstash](https://www.elastic.co/products/logstash)locally and sending the logs there so you can see what gets there in the console.

### Extra bonus

As mentioned earlier, with the configuration we have shown the Gelf library sends all the MDC fields to be indexed, and that’s something you can take advantage of. For example, we are using it to add some extra fields like “operation called”, “parameters”, “Authenticated principal”, “client ip” through some AOP magic (see [Aspect Oriented Programming with Spring](https://docs.spring.io/spring/docs/current/spring-framework-reference/core.html#aop) for reference). But basically, the idea is to have some code like this one to add extra fields automatically:

{{< gist Verdoso 0799d5a435d968246b5252d67e63ce2d>}}

We use this technique just inside Aspects, not to clutter the code, to add some common fields, but it is still possible to parse the message string in [Logstash](https://www.elastic.co/products/logstash)and extract the fields there using the typical grok filter ( see the [Grok filter plugin](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html) for reference ).

You can see here a sample log message, as it is displayed in [Kibana](https://www.elastic.co/products/kibana) when looking at the details:

![Sample log message details as seen in Kibana](/uploads/SampleLogMessage.png "Sample log message details as seen in Kibana")

### Conclusion

With these settings, one should be able to develop applications that, when deployed, send the logs automatically to the [Elastic Stack](https://www.elastic.co/products) without wasting much space due to old log files and with messages that are easier to index and filter. Don’t forget that what I have shown is one way of doing it, the one that serves us well, so there are lots of things that can be customized to suit a different set of requirements. In any case, remember not to log everything but just what is necessary, especially in production ;).

Happy coding!