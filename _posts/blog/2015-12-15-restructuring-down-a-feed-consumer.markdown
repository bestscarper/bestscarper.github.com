---
layout: post          #important: don't change this
title: "Rearchitecting a smelly enterprise pattern"
date: 2015-12-16 09:16:00
author: Ashley
categories:
- blog                #important: leave this here
- enterprise
- dropwizard
- ...
img: post01.jpg       #place image (850x450) with this name in /assets/img/blog/
thumb: thumb01.jpg    #place thumbnail (70x70) with this name in /assets/img/blog/thumbs/
---

Writing this post is intended to act as a spur for myself to, as much as an exposition on a sane decomposition and decoupling of a small monolithic.

The background is a pattern which had emerged in two entirely separate enterprise systems at two very large UK-based media companies (you could take a wild guess, just looking at our landing page).
Both systems were intended to firewall a high-volume client-facing service from a large, metadata database.

Future posts will (unless you're watching on "Dave") detail a skeleton implementation and microservice-y reimplementation, which should prove to decouple the systems more effective and provide a more obvious roadmap.

That doesn't sound too zingy.... I ought to start with some graphics.

<!--more-->

## The Problem

I’ve worked at two large UK-based media companies, providing OTT VoD solutions; I worked at one for several years, the other for several weeks. I was still surprised to find the same smelly architectural pattern a the second that we had encountered at the first. 

The two companies manifested the same painful system
 * media metadata, in a remote ‘behemoth’ database which was not designed for web-scale use
  * the elements of the metadata relate to VoD programming (time-based, with a rolling 'availability' window)
 * data is exposed to downstream systems via a feed of ‘current’ media elements, which is intended to be a rolling window
 * feed is ingested by system adjacent to the web service, and stored in a NoSQL db
 * updates are by deltas (changes since the last poll), but sometimes the whole feed must be reingested

## So what went wrong?

What goes wrong is...
 * the feed grows linearly as
  * (a) scope for media increases, 
  * (b) it include data with indef availability which doesn’t go away
   => the full feed ingest becomes a major issue
 * the system is cron-driven, so only a single instance should ever be run : in one case an NFS partition was used to hold a lock - prone to errors, when the NFS connection died; in the other case, the app was deployed to a single instance, against the shape of the platform
 * the polling client runs in a single thread, so throughput is potentially reduced, esp for the large feed
 * delays in propagation of data from source could be noticeable if media was not published in advance
 * issues with the spec for the ‘delta’ feed whereby data arriving late to the source database was missed - custom APIs and code are a smell here

# ...and what can we do about it?

Ideally, we’d have a message-driven solution whereby changes to the source data were subscribed to by the web service database.
Although it’s not possible to architect this, it should be possible to design an ideal system with adapters to support the semantic gaps.

 * separate concerns of the polling client from the database
 * polling client generates idempotent messages for each item to be ingested, and queues them via a message broker (eg. AMQ/SQS)
 * the complexity of making the polling client HA can be handled later
 * competing consumer which polls the messages to update them web service database idempotently
 * avoid ‘apples and oranges’ - assuming most changes are in one "unit" (but that's inevitably what happened)
 * support migration - great scope for sourcing messages from elsewhere, /en route/ to ditching the polling client
 * provides a faster, less risky, migration path when the source changes…. do you really want to poll 2 systems during migration, or the idempotent messaging?
 
In separating the polling client from the message ingested, we have to define the contract between the systems, which forces us to design the schema for the messages to be ingested.

Playing devil’s advocate, there would be reasonable concerns for not making architectural changes:
 * “it ain’t broke” … ok, that’s a lame story - maintainability, scalability, availability are of proven value for a corporate customer-facing service
 * "the RoI isn’t great for the lifetime of the system”: this is fair enough -
 * “the original system is in maintenance-only - we’ll never get pub-sub” - the decoupling and maintainability are still strong cases for change.

