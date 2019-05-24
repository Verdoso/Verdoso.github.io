---
date: 2018-07-17T18:18:44+02:00
tags:
- Localization
- Java
featured_image: "/uploads/flags.jpg"
title: 'Real life Java: From numbers to localized text ordinals using ICU4J'
draft: true

---
Recently, I had to build a Java application that reads a series of data and puts “human readable” labels on it. Some of the texts the application has to generate use ordinals like “This is the **_first _**report…” and they have to be displayed in three different languages, so instead of using a non-scalable solution (if/else, switch), I decided to generate the messages from the numeric data using something more flexible. A web search turned up [ICU4J](http://icu-project.org/apiref/icu4j/)(from the project International Components for Unicode) as a common solution, so I decided to give it a go.

The technical documentation is quite complete, but I found the samples and usability of the library a bit lacking, especially in the localization aspect, so I decided to show my results and contribute a bit back.

If you want to convert numbers into their text ordinals counterparts in Java, the first step would be to include the [ICU4J ](http://icu-project.org/apiref/icu4j/)library in your project, for example using.

{{< gist Verdoso 2cfb89121fd898b65e647ba119f73dcd>}}

Once we have done that, there are several ways you can display a message with [ICU4J](http://icu-project.org/apiref/icu4j/), but when you are dealing with ordinals, you have to become familiar with the [RuleBasedNumberFormat](http://icu-project.org/apiref/icu4j/com/ibm/icu/text/RuleBasedNumberFormat.html) concept. A [RuleBasedNumberFormat](http://icu-project.org/apiref/icu4j/com/ibm/icu/text/RuleBasedNumberFormat.html) uses a given locale and a format specification (duration, numbering, ordinal or spellout) to format numbers using a specified rule. For example, 1 in English as a spelled out ordinal is “first”, as a spelled out cardinal is “one”, as duration “with words” is “1 second”. In our case, we want the spelled out ordinal (“_%spellout-ordinal_” rule), so we can test it like this:

{{< gist Verdoso a4b20b23590834f3251b4a4a13116893>}}

So far, so good, but how about doing the same with other languages, like Spanish? Well, if we try we’ll get a runtime error because with the Spanish locale we don’t get the same rules as with the English one.

{{< gist Verdoso 440f98ded5e7723444f2f116f106d70e>}}

So, how do you find out what rules are in there for a given locale? Well, you can open the jar files and browse some binary files, as I did first :O, or you can use the API and find it the easy way, with this code.

{{< gist Verdoso 0157f165a2231ddc527c3e573fc7df59>}}

After executing this code, we can see the problem: Spanish has genders so we cannot use the same rule for masculine and feminine nouns. On top of that, the masculine form is different when you use it as a noun or an adjective. In our sample sentence, “This is the **_first _**report…”, the noun, report, in Spanish is masculine and the ordinal is used as an adjective, so we’ll need to use the rule “%spellout-ordinal-masculine-adjective”. Let’s try again:

{{< gist Verdoso 458eadb2f2d4b08802b0f6d2aa5a4bc5>}}

Great, that works. But the code is kind of clumsy and we would have to change the rule depending on the context… there must be an easy way, right? Well, I was about to try to extend the [MessageFormat](https://docs.oracle.com/javase/8/docs/api/java/text/MessageFormat.html) class to handle this use case more easily when I realized IC4J already does that, and their class is named… [MessageFormat](http://icu-project.org/apiref/icu4j/com/ibm/icu/text/MessageFormat.html), oh yeah.

In this case, when we create the [MessageFormat](http://icu-project.org/apiref/icu4j/com/ibm/icu/text/MessageFormat.html) instance, we have to choose which locale will be used along with it, and then we can provide the rule as part of the patterns for the parameters. For example, we can create two different [MessageFormat](http://icu-project.org/apiref/icu4j/com/ibm/icu/text/MessageFormat.html) instances with the messages in English and Spanish, and tell them, in the patterns, to use the appropriate rule in each case. After that, a call to MessageFormat.format, passing the number as a parameter, will return the correct text.

{{< gist Verdoso 9eb2f5eba4ac7ce422a1087145ea7057>}}

Let’s test if we can adapt the messages easily. In some contexts, we could translate the English word “report” for the Spanish word “memoria” instead of “informe”. “Memoria” is feminine, so we just have to change the rule specified in the pattern and…

{{< gist Verdoso 4e684fc2edc8738ce9cff532cdab7fd5>}}

Voilà!

With that logic in place, it is fairly easy to create files that contain the MessageFormat text for each locale and from that, render the messages correctly.

There are plenty of other things you can do with the library, like creating your own definitions if your locale is not supported by default etc., so keep playing with the library if you are interested.

Happy coding!

PS: Photo by [Vladislav Klapin](https://unsplash.com/@lemonvlad?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com/?utm_source=medium&utm_medium=referral)