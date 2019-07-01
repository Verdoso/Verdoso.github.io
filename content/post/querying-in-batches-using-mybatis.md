---
date: 2019-07-01T20:11:27+02:00
tags:
- java
- mybatis
featured_image: ''
title: Querying in batches using MyBatis
draft: true

---
This trick comes straight ahead from an application I'm currently developing: Imagine you have a query that needs to be filtered using the results of a previous resource-intensive query that you already  performed. 

Let's see some pseudo.code:

    -- First query, resource intensive
    select ele.from t_elements ele
    inner join trelated rel
    ... -- many joins and filters
    
    -- Second query, depends on the first one
    select sel.from t_selected
    ... -- other query options
    where sel.ele_id = ...-- Here you need the elements from the first query

You have several options to solve it: 

* One is reusing the first query as part of the where clause of the second. To do that you might define in MyBatis the first query as an <sql> block and then re-use it using 'where exists (<sql id="..."/>). This solutions works and prevents duplicating the query and introducing a bug when someone changes some option in the filter and forgets updating one of the places where the filter appears.   
  It suffers from some issues, though. First, if the first query really is resource intensive, repeating it when you perform the second one is not really going to help performance-wise. Second, you might run into inconsistencies if the t_elements table is updated between the execution of the first and the second query, as you might have some selected elements that are related to elements you did not retrieve with the first query.
* Another option is to actually obtain the elements of the first query when you perform the second query. With MyBatis, defining carefully the result sets you could be able to retrieve a list of elements and related elements in just one query. Even though this solution also works and is efficient, you cannot always apply it because you cannot use this trick if you have several related elements that you need to retrieve based on the original query and/or the first query already retrieves some other related elements. 
* The last option, the one I wanted to explore in detail today, is passing the results of the first query as parameter to the second query and use them to build a where clause using either _where sele.ele_id in (...list of elements...)_ or, given that '[Large IN clauses are problematic](https://www.xaprb.com/blog/2006/06/28/why-large-in-clauses-are-problematic/ "Large IN clauses are problematic")', you can mimic a table from a list of literarls, insert elements into temporary tables or some other alternatives. You can do this with as many queries as you want, so you are not limited by the relationships you want to retrieve; and it also helps having the queries that retrieve different elements separated.  
    
  A piece of warning though, the _IN_ condition might have DB limitations in the number of elements (1.000 in Oracle, Short.MAX_VALUE in PostgreSQL, for example) so one way of circumventing that limitation and turning it into an advantage is dividing your list of values in smaller batches and then run them in parallel.