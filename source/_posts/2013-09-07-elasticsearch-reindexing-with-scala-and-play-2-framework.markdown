---
layout: post
title: "How we sped up Elasticsearch queries by 100x - Part 1: Reindexing"
date: 2013-09-07 10:58
comments: true
categories: ['Elasticsearch', 'Play', 'Scala', 'Performance']
---

## A Little Performance Story

At [Iterable](http://iterable.com), we use [Elasticsearch](http://www.elasticsearch.org/) to store all our email campaign events. This includes email sends, opens and clicks.

For each event, we attach the recipient email and display the events nicely in our campaign dashboard

![Iterable Dashboard](http://static.iterable.com/iterabletoken/13-09-07-Screenshot%202013-09-07%2011.36.46.png)

Notice `Unique Opens` column above. To calculate that, we need to fetch recipient emails from all the open events and do a unique operation on the emails. As you can imagine, this query is quite inefficient. It took upwards of 5 seconds to fetch 500,000 events, despite gigabit connections between our app server and Elasticsearch.

```
[2013-08-26 20:57:33,856][WARN ][index.search.slowlog.fetch] [][][0] took[4.2s], took_millis[4207], types[emailOpen], stats[], search_type[QUERY_THEN_FETCH], total_shards[5], source[{"size":500000,"facets":{"dateFacet":{"date_histogram":{"field":"createdAt","interval":"minute"}}}}], extra_source[],
[2013-08-26 20:57:34,293][WARN ][index.search.slowlog.fetch] [][][4] took[4.6s], took_millis[4659], types[emailOpen], stats[], search_type[QUERY_THEN_FETCH], total_shards[5], source[{"size":500000,"facets":{"dateFacet":{"date_histogram":{"field":"createdAt","interval":"minute"}}}}], extra_source[]
```

This sluggishness slowed up in our [Elasticsearch's slow log](http://www.elasticsearch.org/guide/reference/index-modules/slowlog/), excerpted above.

To solve this slowness, we had to off load the uniquify queries to the Elasticsearch server and simply return the counts, though this came with a few challenges.

## The Power of the Scroll

### How tokenization affects your searches

By default, Elasticsearch applies the [standard analyzer](http://www.elasticsearch.org/guide/reference/index-modules/analysis/standard-analyzer/) to fields. For the email address `justin.9000@gmail.com`, the standard analyzer creates three tokens: `justin`, `9000` and `gmail.com`. If we're to search the field with a terms facet to get unique email counts, the result will include an entry for each token, which inflates the unique count.

The [Elasticsearch inquisitor plugin](https://github.com/polyfractal/elasticsearch-inquisitor) has a handy tool that shows tokenizations with different analyzers.

![Inquisitor analyzer tool](http://static.iterable.com/iterabletoken/13-09-07-inquisitor-plugin.png)

### Taming the scroll with Scala

We must set the mapping of the email field to `not_analyzed` for Elasticsearch to return the email as one token. However, changing a mapping requires reindexing of the data for the new mapping to take effect.

As of Elasticsearch 0.90.3, there is no built in reindexing support. Instead, we rely on the [scroll query](http://www.elasticsearch.org/guide/reference/api/search/scroll/) which essentially maintains the query results in memory, for performance and consistency reasons. There will be some data loss in the next index which can be fixed with another scroll query.

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

The code above fetches the `_source` and `email` fields from each entry and reindexes to `newIndex`. Be sure to set up the new mapping properly on the new index first, otherwise all that effort was for naught.

Once the data is in the new index, the last step is to delete the old index and create an [alias](http://www.elasticsearch.org/guide/reference/api/admin-indices-aliases/) to the new one.

```
curl -XPOST 'http://localhost:9200/_aliases' -d '
{
    "actions" : [
        { "add" : { "index" : "the_old_index", "alias" : "a_new_index" } }
    ]
}'
```

You can create the alias through the [ElasticSearch Head plugin](http://mobz.github.io/elasticsearch-head/) which also nicely displays the aliases on an index.

![Elasticsearch Head Alias](http://static.iterable.com/iterable/13-09-09-ElasticSearch_Head-2.png)

## Coming up next...

In the next part, I'll go into how we ran the uniquification queries on the server side, now that the email field is not analyzed and treated as one token.
