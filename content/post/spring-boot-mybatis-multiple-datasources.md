---
date: 2017-05-22T22:00:00.000+00:00
tags:
- Java
- Spring Boot
- MyBatis
featured_image: ''
title: Spring Boot, MyBatis, multiple datasources and multiple mappers, all together
  holding hands

---
One of the issues with Spring Boot that I have come across a couple of times and that are usually a bit painful to solve is how to configure multiple datasources and mappers with MyBatis. I have always had to browse the web, check different sources and try different examples while fighting unclear error messages until I get it right. And then the next time configuration has changed in some subtle way either in MyBatis or Spring Boot and it’s all over again. Duh! So, now that I found another working solution, I wanted to share it in case it can help other developers going through the same pain. In this case, I’ll show how it’s done using a pure Java configuration approach instead of XML.

First, make sure you include the dependencies for the datasource implementation, in my case HikaryCP, and mybatis-spring-boot-starter. As I use HikaryCP instead of the default Tomcat datasource and I’m using Jetty instead of Tomcat, I have to exclude the Tomcat dependencies from mybatis-spring-boot-starter. Hence:

{{< gist Verdoso a1a9e6ca5f0325ed708d62737e598eaf >}}

In order to use more than one datasource you cannot relay on Spring Boot’s datasource autoconfiguration, but you still can set them up very easily using @ConfigurationProperties.

{{< gist Verdoso bf1f9c4613f042b612341966daa5117e >}}

Note that the @Primary annotation is specified so the elements that require the default datasource do not complain about having more than one bean to choose from.

You can then set up both datasources like this in your application.yml:

{{< gist Verdoso fffdb9330bf5a68e649a9600b55a3e65 >}}

Now here it comes the tricky part, you have to configure more than one session with MyBatis and also configure the different mappers so each one use the right MyBatis session. This is the part that I have had to change a couple of times, depending on the Spring Boot and Mybatis versions, as there are different ways one can configure a Mapper in MyBatis and depending on the order you set up things, you will get all your mappers to use the primary datasource no matter what. You can see that the trick is that, in order to create each SqlSessionFactoryBean, we autowire each datasource specifying the name, and then we add each Mapper class directly to each SqlSessionFactory. We then create a MapperFactory for each Mapper using the right SqlSessionFactory, again specified by name. In my case, each Mapper interface has its corresponding .xml mapper file in resources but I suppouse annotation based mappers will work as well. Specifying the mapper twice seems like overkill, but you can try yourself what happens if you don’t do it*.

{{< gist Verdoso 9e89bf986ce68371c5142f7189052c98 >}}

After all of this, simply autowire the mapper class on your components and each one should connect to a different database.

To reiterate, *this works now using Spring Boot version 1.5.3 and MyBatis Spring Boot starter version 1.3.0. I hope it works for you as well, but if it does not, I would recommend trying to configure the Mappers differently like creating the Mapper beans directly, or using MapperScannerConfigurer beans (this one used to work for me, but I could not get it to work with the aforementioned versions).

At least, I hope it gave you some hints about where to start tweaking.  
Happy coding!  
D.

Edit February 2018: Added a project to Github demonstrating the technique: [https://github.com/Verdoso/multi-mybatis-demo](https://github.com/Verdoso/multi-mybatis-demo "https://github.com/Verdoso/multi-mybatis-demo")