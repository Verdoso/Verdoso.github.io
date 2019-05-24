---
date: 2019-05-24T18:13:25+02:00
tags:
- Java
- Collection
- Functional Programming
- Jira
featured_image: ''
title: 'Real life Java: Aggregating nested collection with streams'
draft: true

---
Most of the times I had used the “relatively new” streams in Java, it was for straight-forward tasks with operations on one stream. I sometimes use more sophisticated collectors, like [Collectors.toMap](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collectors.html#toMap-java.util.function.Function-java.util.function.Function-), but recently I had to use more complicated stuff to solve a real problem and I thought it might interesting writing about it, as the examples one usually reads about are synthetic and lack the focus of solving a real problem.

In this case, my problem was gathering some data from some [Jira](https://www.atlassian.com/software/jira) reports and producing some statistics. The part about gathering the data I did simply by exporting some Jira filters into Excel files and reading them using the [Apache POI library](https://poi.apache.org/) (also using streams). I was just interested in some of the data (Issues, components, versions and time reported by users) so I created two simple classes to hold the data:

{{< gist Verdoso 3fd10707edeb0696bdc1663d7fd87312>}}

Basically, an issue holds the component and version attributes and a list of reports that include the username and the time reported in that task. In this case, I weren’t interested in the dates of the reports, as they had already been filtered at the Jira level.

First, I read the data from several excel files, one with the issues definition and others with the time reports for a given dates interval and end up with a “_List<Issue>_ nonEmptyIssues” structure where we have the issues that have some useful reports.

Once we have the data read, the questions to answer were:

· **How much time was reported by each user?**

· **How much time was reported for each component?**

· **How much time was reported for each version?**

The **user report** is the simplest, as the user attribute is at the report level: I just had to convert the stream of issues into a stream of reports and then aggregate the time for each report, for each user. In “stream-speak”, that means flattening the stream, grouping by user and then reducing each list of “reports by user”. The final code looks like this:

{{< gist Verdoso 9339112dc719ca83ab832b654e00763e>}}

The **component and version reports** are a bit more complicated, as they are attributes at the _Issue_ level, but we want to aggregate the time at the _Report_level. So, in this case, we have to group first and then for the values of the grouping (a list of issues), transform them into a stream of issues and then aggregate them. Fortunately, there is a Collectors.groupingBy function that allows you to operate further on the values associated to each key. The final code looks like this:

{{< gist Verdoso 386cf92c641948948f72789c3c80cf2e>}}

Unfortunately, the Collector that we need to convert the collection of issues into a stream of reports ([Collectors.flatMapping](https://docs.oracle.com/javase/9/docs/api/java/util/stream/Collectors.html#flatMapping-java.util.function.Function-java.util.stream.Collector-)) was not present in Java until version 9, but a bit of [StackOverflow](https://stackoverflow.com/questions/41878646/flat-mapping-collector-for-property-of-a-class-using-groupingby) and [s](https://stackoverflow.com/questions/41878646/flat-mapping-collector-for-property-of-a-class-using-groupingby)ome smart extension and voilà, you can backport it to Java 8:

{{< gist Verdoso 4c3f034d110b5a4785a39e8232633c7b>}}

And that’s it, you can execute these pieces of code in order to get aggregated stats for each user, component and version, and the code is much simpler than what is used to be. These are some of the type of things that make streams really shine.

Happy coding!

PS: As a side note, if you encapsulate some functions properly, reading the Excel sheets can look also quite elegant:

{{< gist Verdoso 183d33b03f65a6be2ded2c99ac6f9139>}}