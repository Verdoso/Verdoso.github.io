---
date: 2017-10-16T21:09:38+02:00
tags:
- Java
- Spring Boot
- Caching
featured_image: ''
title: Configuring Multiple TTL Caches in Spring Boot dinamically
draft: true

---
Recently, I started using Spring caching in my Spring Boot applications and I came across a very nice entry in Daniel Olszewski’s blog ([MULTIPLE TTL CACHES IN SPRING BOOT](http://dolszewski.com/spring/multiple-ttl-caches-in-spring-boot/)) that indeed explains how to set up multiple caches in Spring Boot with a specified TTL (Time To Live) setting and is really worth reading it as it even explains how to test the functionality. So, if you have not read it already, I invite you to do go ahead and do it now, as I’ll build on top of that. Don’t worry, I’ll wait for you :X

When I was applying his advice in my code, I noticed that what I really wanted to do was externalize my cache settings so I could change them depending on the environment (for example, to disable long term caching in development). I also shared the article with some friends and they asked me if was applying the same settings to all the different caches or using the same settings as the blog entry does.

As I told them the blog entry is just the beginning and you should build on top of that, as I did, but I decided to share the step forward that I did to make things clearer.

Once you have your custom cache manager, it might look something like this:

{{< gist Verdoso baa283bacb04d5ba32d12de0d9a61ef0>}}

Now, what we need to do is externalize the settings of the caches… but what if we want to introduce a new cache later on, will we need to come back to this class to build manually a new CaffeineCache and add it to the manager? That does not sound right, huh? I don’t like it either so let’s make it dynamic.

First, we’ll set up the configuration class to be filled up with external properties and make room for the caching properties in our configuration namespace by setting up a prefix. This is done with the following annotation (use whatever prefix you like):

{{< gist Verdoso 557d8c60b855b88f13442dc72b906bb2>}}

Then, we want to specify various settings for each cache, so let’s create a class to hold those values:

{{< gist Verdoso df7267ad47af618ec7689d48d562121c>}}

I know, I know, I’m “cheating” by using [Lombok](https://projectlombok.org/) to avoid having to write the boilerplate code but such is life for us lazy programmers. And also… where’s the cache name? Fear not, it comes next.

We want to be able to add new caches settings dynamically without having to add more members to the CacheConfiguration class, so let’s use a dynamic structure and while we are at it, we need a name per configuration. This smells like a Map, so let’s add it.

{{< gist Verdoso 874ec50143cdcddc4c2c9cb1fbddbfd1>}}

And we now need just to modify slightly the CacheManager code so it uses the Map to create the List of caches dynamically. We can even make use of Java 8 streams to do it in one woosh.

{{< gist Verdoso f5cde1ae62462220f282d4421404f290>}}

Note that if you wanted to add more configuration capabilities to the class, you would just need to change the _CacheSpec_ class and then the internals of the _buildCache(…)_ method so it takes those new values into account. As you can see in the _CacheSpec_ class, you can have defaults in case no value is specified in the properties for certain attributes.

With these changes, in order to configure the same caches as previously, you just need to configure some properties like this: (for example in application.yml or an externalized per environment configuration file)

    caching:
      specs:
        xCache:
          timeout: 10  
        yCache:
          timeout: 60
          max: 500

And if you want to add a new cache, you dont’a have to touch the CacheManager class, you just add a new section in the properties and that’s it:

    caching:
      specs:
        xCache:
          timeout: 10  
        yCache:
          timeout: 60
          max: 500
        aNewCache:
          timeout: 5
          max: 10

All in all, the resulting CacheConfiguration class might look like this (thanks to Lombok)

{{< gist Verdoso 10647537c004a3df482bcf2141cf03fd>}}