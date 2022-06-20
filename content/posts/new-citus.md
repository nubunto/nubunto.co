+++
title = "Citus 11.0"
date = "2022-06-20T13:06:12-03:00"
author = "Bruno Panuto"
authorTwitter = "panuto_"
cover = ""
tags = ["citus", "postgres", "dev"]
keywords = ["database"]
description = "The new version of the amazing Citus Postgres extension!"
showFullContent = false
readingTime = false
hideComments = false
+++

New Citus version on the way! The new Postgres extension version 11.0 is so cool I decided to blog a little about it.

For those of you who never heard of it, Citus is a Postgres extension that allows you to shard your Postgres database. It's a scaling problem: relational databases tend to be heavy machines that, eventually, hit processing and storage limits. Sharding allows you to distribute your data in a cluster of machines, essentially increasing your processing and storage power. Citus allows you to do so without necessarily leaving SQL, although the design decisions for this are still relevant.

There's two new features I want to talk about, and a change related to business model that I found quite interesting.

The first feature: Citus no longer blocks reads and writes on shard rebalancing. This is something that already existed (more on that later), but now it's open source and available for mere mortals. Blocking is not always bad, but on a large scale it can increase latency, which can affect your business. Shard rebalancing now uses logical replication to avoid this blocking.

The second feature: any Citus node can now do distributed reads and writes! This is awesome because, in older versions, Citus required you to have a "coordinator node" for distributed queries. It's a common pattern in distributed databases, like cassandra or dynamodb: A central point that monitors and knows about the other nodes is necessary to find the right node for a shard key or distribute the necessary queries, in Citus case. With Citus, the coordinator node is still required for schema changes, but this schema is now synchronized between all known nodes, which allows distributed reads AND writes. This is good because it alleviates pressure on a single node, while also removing the spof (single point of failure) of your cluster.

The last change, that is quite interesting: Citus is now 100% open source! In older versions, features like the non-blocking shard rebalancing were only available for the credit-card-willing people that wanted it, in the "enterprise" version. Now, all of the paid features are open source and available. This is an interesting shift, because it makes it easier for companies and people to use the tool more. I'm a big open source fan, so I see this as an amazing move from the Citus team.

What about you? Did you knew about Citus? Have you ever needed this type of capability for your SQL databases? Do you have any questions? Feel free to reach out to me in either Twitter or Linkedin!