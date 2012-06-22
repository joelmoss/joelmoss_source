---
layout: post
title: Speeding up SQL queries containing TEXT fields
categories: articles
---

Last week, we encountered some performance issues with a particular MySQL query that was creating a lot of disk based temporary tables, despite the servers having plenty of room to create these temporary tables in RAM. After a little digging we discovered that when MySQL is running an ordered query that involves a TEXT field, it will create temporary tables on disk every time.

So a query like this would create temporary tables on disk, if any one of the included columns has a type of TEXT:

{% codeblock %}
SELECT deals.* FROM deals ORDER BY created_at;
{% endcodeblock %}

The above is a very simple example, and our query was much larger, so was even more pronounced for us. But you get the point - I hope!

So what we needed to do was avoid the inclusion of our TEXT fields in the query, but while still returning the value of our TEXT field. The following is what we came up with:

{% codeblock %}
SELECT * FROM deals INNER JOIN ( SELECT DISTINCT deals.id FROM deals ORDER BY RAND() ) x ON (x.id = deals.id)
{% endcodeblock %}

The above simply wraps the current query in an `INNER JOIN`, but the key to this is the change in what fields we are selecting. The old query is returning all fields using `deals.*` which includes any TEXT fields, but this new one is only including the ID with `deals.id`, which already makes things much faster, and of course avoids the need to create disk based temporary tables.

We ended up speeding up some of our SQL queries by up to 300% using this technique.