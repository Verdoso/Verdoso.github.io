---
date: 2019-09-09T16:15:34.000+00:00
tags:
- java
- spring boot
- json
- versioning
featured_image: "/uploads/dna-1811955_1280.jpg"
title: Spring Boot JSON API versioning options

---
One of the common issues when developing an API specification is how to deal with API contract modifications. Hopefully one is very successful and has lots of customers with software clients already using the current version of the spec. so simply dropping the existing version and moving to the new one is not an option, hence the issue.

[Spring Boot](https://spring.io/projects/spring-boot), with its automatic JSON mapping with [Jackson](https://github.com/FasterXML/jackson), is a very productive framework to develop APIs, but one still has to face the problem on how to adapt the code to support several API versions with the same code base.

In this installment, we will explore different option how we can handle coexisting API versions and the advantages and drawbacks of each approach.

Just to get it out of the way, we will mention that there is always the option of not allowing more than one version per codebase, so each version becomes a separate project with a different artifact. I was not even going to mention it, but it is indeed being used in some places, so… yeah, the option is there.

## Introduction

In all the approaches we are going to demonstrate, with a project available at Github ([**VersioningDemo**](https://github.com/Verdoso/VersioningDemo)), we will show a controller that displays a different version of a JSON depending on the version of the API we are requesting. Selecting the version is done through the URL, for example _/controller/version/test_, but the version could also be specified as request header, for example.

It is also usually a good idea to separate the objects that are part of the business logic from the objects that you use to represent the API to the external world, especially if you are using an auto-mapping tool like Jackson to generate the JSON from Java objects, so that’s what we have done. The Java objects that represent your JSON become your view, so the same principles to separate the model from the view apply here.

The classes that represent our business logic model are in the [_org.greeneyed.versioning.demo.model_](https://github.com/Verdoso/VersioningDemo/tree/master/src/main/java/org/greeneyed/versioning/demo/model) package and are fairly simple:

MyPojo

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/model/MyPojo.java?footer=minimal"></script>

And RelatedPojo

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/model/RelatedPojo.java?footer=minimal"></script>

The class that represents our business logic processes is PojoService and it is a simple mock up:

PojoService:
<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/services/PojoService.java?footer=minimal"></script>

The classes that represent the view are mainly in the _org.greeneyed.versioning.demo.api_ package, even though in some cases I have left them as static inner classes of the controllers for the sake of brevity (don’t try this at home :D).

In our service, we will have an API that when called with the version 1, it will return this structure:

![API version 1](/uploads/API-v1.PNG "API version 1")

Let’s imagine that in our business, someone decided later on that including the related id and forcing the clients to make another request to get the related name was not the way to go, so the related name was added to the response, and in order not to clutter the original basic object, a related object appeared. So version 2 looks like this:

![API version 2](/uploads/API-v2.PNG "API version 2")

We use this as an example because it is complex enough: We have some common fields that will appear in both versions and then some fields that will appear in one version but not the other.

And yes, we all know that there exists the option in the client to simply ignore the fields that are not mapped, so you could simply return all the fields and let the client software sort it out, but experience tells us that making the customers responsible for those changes does not look good… and does not even really work for our purposes (make our lives easier).

So let’s explore the alternatives:

## Classic

The classic approach is the one that requires less support from the framework and it can also be used when no automatic approach helps. In this case, what we do is to have several sets of classes that represent the different views/versions of the API and then convert to one set or another depending on the version requested. The controller that demonstrates this behavior in our demo is [_org.greeneyed.versioning.demo.controllers.ClassicVersioningAPI_](https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/ClassicVersioningAPI.java):

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/ClassicVersioningAPI.java?footer=minimal"></script>

where depending on the version specified, we convert the “version neutral” model classes into the view classes from package _org.greeneyed.versioning.demo.api.v1_ or _org.greeneyed.versioning.demo.api.v2_. We would wrap it with more code to make it more reusable etc. but for the demo we have reduced the code to the fundamental parts.

The good thin is that it is simple and straightforward. The main drawbacks is that it requires creating new sets of classes and conversion code per version.

If you don’t create new classes for things that are almost-the-same, then the conversion code becomes cluttered with conditionals to fill up fields in some cases and not in others, which is error prone.

If you create complete new hierarchies for each version, then the code is easier to check and reason about, but then you might end up with repeated boilerplate “copy” operations all over the place.

So, let’s see if we can remove part of that boilerplate code

## Static Jackson JSON Views

Jackon serialization includes a feature where you can “tag” some fields with a version, represented by a class, and those fields will be included or excluded depending on the view you specify at serialization time (you can read more about it at Baeldun’s “[Jackson JSON Views](https://www.baeldung.com/jackson-json-view-annotation)” entry).

Spring included support for such mechanism a while ago ( see “[Latest Jackson integration improvements in Spring](https://spring.io/blog/2014/12/02/latest-jackson-integration-improvements-in-spring)” from December 2014) so we can use that. The idea is to produce the same classes from the business logic and let Jackson serialize them differently depending on the version that is specified in the MVC handler method through the same annotation used to tag the model: _@JsonView_. You can see how this is done in the [_org.greeneyed.versioning.demo.controllers.ViewsVersioningAPI_](https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/ViewsVersioningAPI.java) controller class.

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/ViewsVersioningAPI.java?footer=minimal"></script>

The class includes two handler methods, one per version, each one annotated with the correct class, but both of them defer to the same business logic, so there is no duplicate copying code. The trick here is that both controllers return objects from the type [_org.greeneyed.versioning.demo.api.MyPojoAPI_](https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/api/MyPojoAPI.java) and in that class you can see that the attributes are tagged with _@JsonView_ as well, so Jackson knows which ones to include and which ones to skip.

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/api/MyPojoAPI.java?footer=minimal"></script>

That means we just need one set of classes for both versions and one piece of logic to fill up the objects representing the API. On the other hand, we are duplicating the MVC handler methods just to specify the version, so if we have to do that for each method and version we have… auch! We can do better, as explained in the next technique.

## MappingJacksonValue with JSON Views

So, once we are at this point, we just have one set of classes and one piece of “boilerplate copying”. Unfortunately, the @JsonView annotation just allows us to specify one version at compile time, so how do we get rid of the need to create a handler method per version?

Well, the solution is relatively simple. When you return an object in an MVC handler method, Jackson knows it has to serialize it, but it has no other information about it, so it uses the global mapper for that. On the other hand, if you wrap your object around a [_MappingJacksonValue_](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/http/converter/json/MappingJacksonValue.html), it lets you specify the view class for that object and it is the one that Jackson will use to serialize it.

So, you just need to create a _MappingJacksonValue_ instance from your original object and return it after specifying the appropriate view class. And that’s exactly what the [_org.greeneyed.versioning.demo.controllers.MappingJacksonValueViewsVersioningAPI_](https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/MappingJacksonValueViewsVersioningAPI.java) controller class demonstrates:

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/MappingJacksonValueViewsVersioningAPI.java?footer=minimal"></script>

You can see that the logic and the API model classes are the same as in the previous example. It is just that we are specifying the view to be used per request, instead of statically per method.

## MappingJacksonValue with JSON Filters

Jackson also provides a different way of specifying which fields we want serialized or not, and that’s through the use of JSON filters (you can learn more about them at the Baeldung’s entry “[Serialize Only Fields that meet a Custom Criteria with Jackson](https://www.baeldung.com/jackson-serialize-field-custom-criteria)”).

In this case we need to specify the class/instance, the filter, that will decide what is going to happen to each field. Why use that instead of _@JsonView_ at the model? Well, the annotation is static, it allows you to specify just one value, and it is specified at compile time, so there’s nothing in runtime you can do about it. So, if you need more than that, you have the alternative of using JSON filters.

The good news is that Spring also supports specifying the filter instance to be used through the _MappingJacksonValue_ class that we used in the last technique. That is the reason why the code implementing this approach is pretty similar to the previous one, except this time, instead of a class, we are setting a different PropertyFilter instance depending on the version. You can see how it is done at the [_org.greeneyed.versioning.demo.controllers.MappingJacksonValueFilterVersioningAPI_ ](https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/MappingJacksonValueFilterVersioningAPI.java)controller class:

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/MappingJacksonValueFilterVersioningAPI.java?footer=minimal"></script>

Note that in this case we have included a different version of the API model classes just to show that we are not using _@JsonView_ annotations.

Json filters allow you to specify what happens in runtime for each field, but, on the other hand, the information that you have in runtime about the field you are deciding upon is pretty limited: Basically the name and some limited metadata.

In fact, we are using one of those metadata fields (the description) to encode the versions we want the field to be included in, but we have to admit it does not feel completely right to use a field for a purpose it was not created for. The alternative, though, is to hardcode the names of the fields to include in each version, and that feels even “less right”.

## Jolt

Once we have explored the different versions that Jackson offers, we wanted to show a different technique that is pretty similar to what one would do with XSLT if this was an XML API. And for that we can use [Jolt](https://github.com/bazaarvoice/jolt), a Java library to perform JSON to JSON transformations where the specification is, itself, a JSON document.

Spring Boot does not include support for Jolt processing of the JSON produced by Jackson, but nothing prevents us from adding it (except time and resources, that is ;) ), so that’s what we did (through the class [SummerJoltView](https://github.com/Verdoso/GreenSummer/blob/master/summer-core/src/main/java/org/greeneyed/summer/util/SummerJoltView.java)). In this case, what we need to do is simply return the new version of the API model object and then specify, for each version, which transformation specification we want to apply. You can see that in the controller [_org.greeneyed.versioning.demo.controllers.JoltVersioningAPI_](https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/JoltVersioningAPI.java):

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/java/org/greeneyed/versioning/demo/controllers/JoltVersioningAPI.java?footer=minimal"></script>

The transformation specifications, jolt-v1.json and jolt-v2.json, are located in the resources/json-spec/ folder. The syntax is not exactly intuitive but you should be able to see that in v1 of the spec, the related.id field is copied to the output as related_id:

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/resources/json-spec/jolt-v1.json?footer=minimal"></script>

and in v2, both related.id and related.name are copied following the same hierarchy:

<script src="https://gist-it.appspot.com/https://github.com/Verdoso/VersioningDemo/blob/master/src/main/resources/json-spec/jolt-v2.json?footer=minimal"></script>

The good thing with this solution is that the Java code does not need to be “version aware”, so it becomes much simpler. The drawbacks are that the community around the library is not very active, not to say pretty much dead, and that the syntax is pretty convoluted. But we haveve used it successfully in some small projects and the resulting code was cleaner, so we simply wanted to show a different style of solving the problem.

## Final notes

We have demonstrated several techniques how one can evolve an API version using Spring Boot and Jackson and reasoned about the changes in your code that each of them requires. Now it’s up to you to decide which one suits your requirements and environment better. Bear in mind that you might need to mix and match one of the advanced techniques (Views/Filters) with the classic approach, as radical changes in structure are better handled with a different set of API model classes.

In many cases, you will want to introduce changes in your business code to prevent some functionality to be executed depending on the version. As in “why access the DB to fill up some objects that are not going to be displayed in the final response?”, but that’s outside the realm of Jackson serialization and the scope of this entry.

## Extra bonus

Again, the code with all the techniques is at the [Versioning Demo project @ GitHub](https://github.com/Verdoso/VersioningDemo) where you will also find a test class that shows how to create a parameterized test with MockMVC.

Happy coding!

PS: Kudos @[gist-it.appspot.com](https://gist-it.appspot.com/) for creating the service to embed files directly from GitHub as "gists". Much appreciated!

(Image by [Arek Socha](https://pixabay.com/users/qimono-1962238/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1811955) from [Pixabay](https://pixabay.com/?utm_source=link-attribution&utm_medium=referral&utm_campaign=image&utm_content=1811955))