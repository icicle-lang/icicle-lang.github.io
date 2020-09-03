---
title: How Icicle Makes Collaboration Easy
---

By allowing queries to be specified separately in an intuitive
module system, Icicle provides a great collaborative experience.

To see this in action, let's imagine we're a large media organisation,
and want to look at how our customers are exploring our sites.

We'll start with a simple model for a customer's interaction with
a page. For this we define a record named `page_views` as input stream

```
input page_views : {
    url : String
  , referrer: Option String
  , category: String
  , subcategory: String
}
```

This type represents our raw data; and all input streams carry two
implicit pieces of information:

- A primary key (think user), and
- A time at which the event was recorded.


With this we could imagine that two
data scientists may want to know: has this user gone to the home screen;
and what's the number of views for "sport" in the past 4 weeks.

They might write this as:
```
feature viewed_homepage =
  from page_views
    in any (category == "home")

feature mean_view_time =
  from page_views
    in windowed 4 weeks
    in filter category == "sport"
    in count category
```

To introduce a new query, no existing features need to be touched, which makes
reviewing new queries easy.

When a large number of queries are specified, it can make sense to break them
up into separate modules, and Icicle makes this easy. One module can import
functions and input definitions from others.

If one finds that that they are reusing similar terms in their query expressions,
they can reduce repetition by writing functions, aiding long term maintenance.

