---
layout: post
title: "How we sped up Elasticsearch queries by 100x - Part 1: Reindexing"
date: 2013-09-07 10:58
comments: true
categories: ['Elasticsearch', 'Play', 'Scala', 'Performance']
---

## A little performance story

At Iterable, we use Elasticsearch to store all our email related events. This includes, email sends, opens, clicks, unsubscribes.

For each event we attach the recipient email, timestamp, and additionaly for opens & clicks, the client ip and user agent.

We then aggregate all the events and display them nicely in our campaign dashboard

![Iterable Dashboard](http://static.iterable.com/iterabletoken/13-09-07-Screenshot%202013-09-07%2011.36.46.png)

You'll see above the column, Unique Opens. To calculate that, we need to fetch recipient emails from all the open events and to a unique operation on the email.

As you can imagine, this query is farily slow. It took upwards of 5 seconds to fetch 500,000 events to despite gig-e connections between our app server & elasticsearch.

```
[2013-08-26 20:57:33,856][WARN ][index.search.slowlog.fetch] [][][0] took[4.2s], took_millis[4207], types[emailOpen], stats[], search_type[QUERY_THEN_FETCH], total_shards[5], source[{"size":500000,"facets":{"dateFacet":{"date_histogram":{"field":"createdAt","interval":"minute"}}}}], extra_source[],
[2013-08-26 20:57:34,293][WARN ][index.search.slowlog.fetch] [][][4] took[4.6s], took_millis[4659], types[emailOpen], stats[], search_type[QUERY_THEN_FETCH], total_shards[5], source[{"size":500000,"facets":{"dateFacet":{"date_histogram":{"field":"createdAt","interval":"minute"}}}}], extra_source[]
```

This sloggishness slowed up in our [elasticsearch's slow log](http://www.elasticsearch.org/guide/reference/index-modules/slowlog/), excerpted above.

To solve this slowness, we had to move the uniquify queries to the elasticsearch server and simply return the counts. This came with a few challenges.

## The need for mapping a field as "not_analyzed"

By default, Elasticsearch applies the [standard analyzer](http://www.elasticsearch.org/guide/reference/index-modules/analysis/standard-analyzer/) to string fields, create tokens for the value and is used in search.

For the email address `goku.9000@gmail.com`, the standard analyzer create 3 tokens: `justin`, `9000` and `gmail.com`. So if we're to use a terms facet to get unique email counts, we'd get bad results with tokens of the username and domain.

To fix this, we simply set the mapping of our email field to "not_analyzed" which treats the whole email as one token. However, this requires reindexing of that field. As of Elasticsearch 0.90.3, there is no built in redexing support. You instead rely on a [scroll query](http://www.elasticsearch.org/guide/reference/api/search/scroll/) which essentially maintains the query results in memory, snapshotted to the data when you made the query.

Here's the code to do it in our favorite language, Scala

```scala
def reindexType(srcIndex: String, newIndex: String, searchType: String, client: TransportClient) = {
  val scrollTime = new TimeValue(600000)
  val scrollSize = 10000

  // Setup the query
  val query = client.prepareSearch(srcIndex)
    .setTypes(searchType)
    .setScroll(scrollTime)
    .setQuery(QueryBuilders.matchAllQuery())
    .setSize(scrollSize).addFields("email", "_source")

  var scrollResp = query.execute().actionGet()

  // Keep processing results if hits are non empty
  while(scrollResp.getHits.hits().length > 0) {
    // Retrieve items
    val bulkReq = client.prepareBulk()
    scrollResp.getHits.hits().foreach { hit =>
      Option(hit.field("email").getValue[String]).map { email =>
        bulkReq.add(
          client.prepareIndex(newIndex, searchType).setSource(hit.source()).setParent(email)
        )
      }
    }

    scrollResp = client.prepareSearchScroll(scrollResp.getScrollId)
      .setScroll(scrollTime).execute().actionGet()
  }

}

```

The code above fetches the `_source` and `email` fields from each entry and reindexes to `newIndex`. Be sure to setup the new mapping properly on the new index first, otherwise all that was for now.

In the next part, I'll go into how we ran the uniquification queries on the server side, now that the email field is not analyzed and treated as one token.
