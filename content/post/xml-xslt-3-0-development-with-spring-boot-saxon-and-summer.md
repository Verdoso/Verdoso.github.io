---
date: 2019-07-27T16:34:09+02:00
tags:
- Spring Boot
- Java
- XSLT
featured_image: "/uploads/code-1076536_640.jpg"
title: XML/XSLT 3.0 development with Spring Boot, Saxon and Summer

---
(Image by [James Osborne](https://pixabay.com/users/jamesmarkosborne-1640589/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1076536)  from [Pixabay](https://pixabay.com/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1076536))

In this entry, I am going to explain how you can develop web applications using [Spring Boot](https://spring.io/projects/spring-boot) using [XSLT 3.0](https://www.w3.org/TR/xslt-30/) as a view technology. The [XSLT 3.0](https://www.w3.org/TR/xslt-30/) processor I’m going to be using is [Saxon](http://saxon.sourceforge.net/) as that is the only working Java implementation I’m aware of. To facilitate development, I will add my own library to the mix, [GreenSummer](https://github.com/Verdoso/GreenSummer/), because it’s the one I’m most familiar with.

First all, I know XSLT is a controversial technology and prone to abuse, which has caused people a lot of pain and hate. On the other hand, we’ve been using it as our main view technology for more than 20 years and we’ve learnt not to abuse it and work around its pain points, for example, by using other view technologies when necessary. So, if you feel XSLT is not for you, no problem, just skip this entry and keep swimming.

## The basics

The overall idea when building a web application and using XSLT as view technology is that you produce XML from your business logic and then you process that XML with an XSLT stylesheet to produce the HTML (please don’t try to generate JSON with XSLT, it will hurt). The good news is that generating XML with a Spring Boot web application is fairly easy as you can see in the many Spring Boot REST tutorials that perform [automagic XML mapping using JAXB](https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#oxm). The bad news is that the [current default way of using XSLT with Spring Boot](https://docs.spring.io/spring/docs/current/spring-framework-reference/web.html#mvc-view-xslt) is by creating “manually” an [org.w3c.dom.Document](https://docs.oracle.com/javase/8/docs/api/org/w3c/dom/Document.html), or XML as a String, and returning that. That becomes a pain quite fast, so, why not combine JAXB with XSLT so we can return regular Java objects directly and let the JAXB turn them into XML before being processed by the view? That’s what we do with GreenSummer and, boy, it makes life quite simpler!

Let’s start

In order to demonstrate the whole process, let’s begin with a bare bones application created from scratch using [Spring Initializr.](https://start.spring.io/) We will make it very basic so let’s use the options: Maven, Java, Spring Boot 1.5.21, Jar and Java 8:

![](/uploads/Spring-Init.PNG)

After the project is generated, we just need to import it in our IDE of choice. We want this to be a web application, so let’s make some changes in the pom.xml. First, change the spring-boot-starter dependency and make it a spring-boot-starter-web dependency, so it looks like this:

In my case, I’m also adding the Lombok dependency to skip tons of boilerplate; I certainly recommend it. The basic POM now looks like this:

{{< gist Verdoso fcbaffcc740dbdf65251d0373bc0207a>}}

Now, let’s create a controller that fakes some business logic that returns some objects:

The idea is that we will have an API, XsltDemoApi

{{< gist Verdoso f2e927e8ba710a5a1194c6b1064ff3f0>}}

that calls a service, PojoService

{{< gist Verdoso 645e2062b2f1d7264b21b95333e2fbd9>}}

that returns a list of objects of the type MyPojo

{{< gist Verdoso 3c81020a559b20eccfc7205fd3d43962>}}

and wrapps this list in a common class (so we have always a common XML root), App

{{< gist Verdoso 3a85e3c50e8873c2df8522136b75ca38>}}

before returning it.

We are returning the wrapper class (App) directly, so Spring Boot automagically uses JAXB to transform it into XML when the response to our request is returned:

If we start the app and call [http://localhost:8080/test](http://localhost:8080/test), this is what we get back:

{{< gist Verdoso eddf8ee0c9b5cca681cd2c5ab11c0e20>}}

## Adding the view

So far, so good, the XML we just generated contains the information we need to create our user interface. One of the good points of using XML is that you can see in your XML source pretty much* all the information that you have available to play with. ( * pretty much because you can play some tricks in XSLT to get extra XML from external sources)

In order to add the XSLT processing we need to add some other libraries: Let’s add [GreenSummer](https://github.com/Verdoso/GreenSummer/) and [Saxon](http://saxon.sourceforge.net/) to the mix:

{{< gist Verdoso 3dff3fb2dd4cb6a028d628b18e2c3d76>}}

We also need to tell the application that we will be using _GreenSummer_, so we have to edit _XsltDemoApplication_ and add the annotation _@EnableSummer(xslt_view = true, log4j = false)_, so it ends up like this:

{{< gist Verdoso 69691f9459fde60c0f93a7ab467dfa8a>}}

(note: if you forget enable GreenSummer, you’ll get a 404 error when trying to execute the operation as it will try to find a default view with the name "_pojo_process.xslt"_ in it and it won’t be able to find it, hence the 404). We add log4j = false becase by default we always use Log4J2 instead of Logback and Spring does not.

We then need to create the XSLT that will be used to process the XML. We place it in _main/resources/xslt_ (default summer XSLT location) and name it _pojo_process.xslt,_ with this content:

{{< gist Verdoso 37c48217b659489af55aa353480061ef>}}

It is a very simple XSLT and we are not using the full power of XSLT 3.0, but let's start simple. Now we update the controller class so it uses the XSLT. What we are doing is telling Spring we are using a specific ModelAndView class (org.greeneyed.summer.config.XsltConfiguration.XsltModelAndView) that needs the name of the view and the root object for our XML.

{{< gist Verdoso 2a6a985d050aaf76dffab4b5b847ab0e>}}

To end up, an extra detail:

First, as we are in development, we want to be able to change the XSLT and see the changes without having to restart the application. In order to enable that, we just need to set the property _summer.xslt.devMode=true_ in _application.properties_. That configures GreenSummer so it compiles the XSLT each time you use it. If you don’t set it, or set it to false, as you will want to do in production, the XSLT is compiled just once, and then the compiled form is reused each time, so it is more performant.

If you are using Eclipse as IDE, make sure the project is built each time you save your XSLT: _Preferences > General > Workspace > Build > Build automatically_. Otherwise you’ll have to build your project manually each time you want to see your changes in the XSLT (auch!).

Now when you access [http://localhost:8080/test](http://localhost:8080/test), this is what you should see:

![](/uploads/HTML_test.PNG)

And, bonus point, thanks to being in development mode, you can add the parameter showXMLSource=true to see the XML that is being used to produce the page, so accessing [http://localhost:8080/test?showXMLSource=true](http://localhost:8080/test?showXMLSource=true) you should see again:

![](/uploads/showXMLSource.PNG)

This lets you check the XML you are using as source when editing the XSLT stylesheet to verify that your paths and attributes names are correct. When devMode is set to false, access to the XML source is restricted for security reasons.

And that’s it. After that it’s just a matter of creating returning the appropriate classes that contain the relevant information for the page you want to display and create the XSLT stylesheets and templates to process them and generate the desired output.

Creating different interfaces reusing the business logic is fairly easy, just make the XSLT name depend on some parameter and you can produce a completely different result (for example, generating XML/RSS instead of HTML). That makes producing two different XML API versions from the same business logic very straight-forward.

## Final touch: testing

To show the full development cycle of using XML/XSLT with Spring Boot, I’ll show one way how one can test the controller part when using these technologies.

We don’t want to have to start the whole application when doing each test, so we will use MockMvc (see [https://spring.io/guides/gs/testing-web/](https://spring.io/guides/gs/testing-web/ "https://spring.io/guides/gs/testing-web/") for more information) to just test the web layer. A sample testing class is this:

{{< gist Verdoso 6a1cf77bc6786f664730a2a22a46b6dd>}}

We have added two test methods, testXMLIsFine() and testHTMLIsFine(), one to test that the XML we are using as origin for the view is generated correctly, and the other to test that the final HTML is correct. Depending on your role or your requirements, you might want to do both or just one of those, but I added them both so you can see how the two approaches work.

Note that the tests just start the web layer of Spring Boot, so they run relatively fast, compared to having to start the whole app.

Also note that, in order to test the HTML, we have used [jsoup](https://jsoup.org/), so we added it as dependency in the final POM. And as we used “showXMLSource=true” in the tests, we need to enable Green Summer XSLT development mode in src/test/resources/applications.properties

You can see the final complete code here:

Happy coding!