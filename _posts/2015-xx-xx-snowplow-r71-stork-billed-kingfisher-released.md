---
layout: post
shortenedlink: Snowplow 71 released
title: Snowplow 71 Stork-Billed Kingfisher released
tags: [snowplow, hadoop, enrichments, timestamps]
author: Fred
category: Releases
---

We are pleased to announce the release of Snowplow version 71 Stork-Billed Kingfisher. Among other things, this release adds six new fields to the atomic.events table.

The rest of this post will cover the following topics:

1. [Combined configuration](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#tstamps)
2. [New event definition fields](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#events)
3. [New CloudFront access log fields](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#access-log)
4. [The event fingerprint](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#fingerprint)
5. [Faster failure for missing schemas](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#missing-schemas)
6. [Other changes](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#other-changes)
7. [Using sslmode in the StorageLoader](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#sslmode)
8. [New approach to table upgrades](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#table-upgrades)
9. [Upgrading](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#upgrading)
10. [Getting help](/blog/2015/xx/xx/snowplow-r71-stork-billed-kinfisher-released#help)

![stork-billed-kingfisher][stork-billed-kingfisher]

<!--more-->

<h2 id="tstamps">1. Derived Timestamp and True Timestamp</h2>

Sometimes the timestamp reported by a client device clock can be very innaccurate. The derived_tstamp field is an attempt to work around this innaccuracy with the help of the new `dvce_sent_tstamp` field. The idea is that as well as the timestamp at which an event occurred (the `dvce_created_tstamp`), trackers attach a timestamp for when the event was sent (the `dvce_sent_tstamp`). If we assume that the time between the tracker sending the event and the collector receiving the event is negligible, and that the accuracy of the device clock does not significantly change between the event being created and the event being sent, then the following calculation gives a good approximation for the time at which the event was created:

`derived_tstamp = collector_tstamp + dvce_created_tstamp - dvce_sent_tstamp`

At the moment no tracker supports the `dvce_sent_tstamp`, so the `derived_tstamp` defaults to being equal to the `collector_tstamp`.

We also intend to allow tracker users who set event timestamps manually (without having to use an innaccurate client device clock) to specify that an event timestamp is a "true timestamp". True timestamps will directly populate both the `derived_tstamp` field and the new `true_tstamp` field without any intermediate processing.

For a deeper look at our approach to event timestamps, check out Alex's post on [Improving Snowplow's understanding of time][timestamps-post].

<h2 id="events">2. New event definition fields</h2>

Thanks to Dani Solà  ([@danisola][danisola] on GitHub), the Scala Hadoop Shred validation code has been moved to Scala Common Enrich. This means that Scala Hadoop Enrich can now validate unstructured events and custom contexts. In addition, Dani has added `event_vendor`, `event_name`, `event_format`, and `event_version` fields to `atomic.events`. Thanks a lot Dani!

These are the values of those fields for our five legacy event types:

| event_name       | event_vendor                   | event_format | event_version |
|------------------|--------------------------------|--------------|:-------------:|
| page_view        | com.snowplowanalytics.snowplow | jsonschema   | 1-0-0         |
| page_ping        | com.snowplowanalytics.snowplow | jsonschema   | 1-0-0         |
| transaction      | com.snowplowanalytics.snowplow | jsonschema   | 1-0-0         |
| transaction_item | com.snowplowanalytics.snowplow | jsonschema   | 1-0-0         |
| event            | com.google.analytics           | jsonschema   | 1-0-0         |

<h2 id="access-log">3. New CloudFront access log fields</h2>

In July, an [AWS update][access-logs] added four new CloudFront access log fields. The Snowplow CloudFront access log adapter now supports these new fields. You can use [this migration script][cloudfront-access-log-migration] to upgrade your Redshift table accordingly.

<h2 id="combinedConfiguration">4. Event fingerprint</h2>

The new event_fingerprint enrichment creates a fingerprint from a hash of the tracker protocol fields set in an event's querystring (for GET requests) or body (for POST requests). You can configure a list of tracker protocol fields to exclude from the process which creates the hash. For example, it is probably sensible to exclude "stm", which is the tracker protocol field for the `dvce_sent_tstamp`, since this field could change between two different attempts to send the same event.

<h2 id="missing-schemas">5. Faster failure for missing schemas</h2>

The enrichment process used to take a long time if lots of events had schemas which couldn't be found in an Iglu repository. This was because Iglu Scala Client cached successfully found schemas but didn't remember schemas which it had already failed to find. The latest release fixes this problem.

<h2 id="sslmode">6. Using sslmode in the StorageLoader</h2>

Snowplow community member Dennis Waldron ([@dennisatspaceape][dennisatspaceape]) has contributed the ability to connect to Postgres and Redshift using SSL. To do this, add an "ssl_mode" field to each target in your configuration YAML:

{% highlight yaml %}
  targets:
    - name: "My Redshift database"
      type: redshift
      host: ADD HERE # The endpoint as shown in the Redshift console
      database: ADD HERE # Name of database
      port: 5439 # Default Redshift port
      ssl_mode: disable # One of disable (default), require, verify-ca or verify-full
      table: atomic.events
      username: ADD HERE
      password: ADD HERE
      maxerror: 1 # Stop loading on first error, or increase to permit more load errors
      comprows: 200000 # Default for a 1 XL node cluster. Not used unless --include compupdate specified
{% endhighlight %}

Thanks Dennis!

<h2 id="table-upgrades">7. New approach to table upgrades</h2>

Starting in this release, we are taking a new approach to upgrading the `atomic.events` table. Previous upgrades would rename the existing table as "atomic.events_$OLD_VERSION" and create a new table with the new schema. We will now be directly mutating the old table using `ALTER` statements.

To prevent confusion about the version of a particular `atomic.events` table, the table creation and migration scripts now add the version to the table as a comment using the [COMMENT][postgres-comment] statement.

<h2 id="combinedConfiguration">8. Other improvements</h2>

We have also:

* Upgraded Scala Hadoop Shred to use Hadoop version 2.4 [#1720][1720]
* Added validation for `v_collector` and `collector_tstamp` [#1611][1611]
* Upgraded to version 0.2.4 of the referer-parser [#1839][1839]
* Upgraded to version 1.16 of [user-agent-utils][uau] [#1905][1905]
* Changed the BadRow class to use ProcessingMessages rather than Strings [#1936][1936]
* Added an exception handler around the whole of Scala Common Enrich [#1954][1954]
* Updated web-incremental so failure is recoverable [#1974][1974]
* Fixed a bug where Scala Hadoop Shred didn't correctly add original LZO-encoded event strings to bad rows [#1950][1950]

<h2 id="upgrading">9. Upgrading</h2>

The latest version of the EmrEtlRunner and StorageLoadeder are available from our Bintray [here][app-dl].

You should update the versions of the Enrich and Shred jars in your configuraiton file:

{% highlight yaml %}
    hadoop_enrich: 1.1.0 # Version of the Hadoop Enrichment process
    hadoop_shred: 0.5.0 # Version of the Hadoop Shredding process
{% endhighlight %}

You should also update the AMI version field:

{% highlight yaml %}
    ami_version: 3.7.0
{% endhighlight %}

Use the appropriate update your version of the atomic.events table to the latest schema:

* [The Redshift migration script] [redshift-migration]
* [The PostgreSQL migration script] [postgres-migration]

If you are ingesting Cloudfront access logs with Snowplow, use the [Cloudfront access log migration script][cloudfront-migration] to update your "com_amazon_aws_cloudfront_wd_access_log_1.sql" table.

If you wish to use the new event fingerprint enrichment, write a configuration JSON and add it to your enrichments folder. An example JSON can be found [here][example-event-fingerprint].

<h2 id="help">10. Getting help</h2>

For more details on this release, please check out the [R71 Stork-Billed Kingfisher release notes][r71-release] on GitHub. 

If you have any questions or run into any problems, please [raise an issue][issues] or get in touch with us through [the usual channels][talk-to-us].

[timestamps-post]: http://snowplowanalytics.com/blog/2015/09/15/improving-snowplows-understanding-of-time/
[stork-billed-kingfisher]: /assets/img/blog/2015/09/stork-billed-kingfisher.jpg
[danisola]: https://github.com/danisola
[dennisatspaceape]: https://github.com/dennisatspaceape
[access-logs]: http://aws.amazon.com/releasenotes/CloudFront/6827606387084636
[uau]: https://github.com/HaraldWalker/user-agent-utils
[example-event-fingerprint]: https://github.com/snowplow/snowplow/blob/master/3-enrich/config/enrichments/event_fingerprint_enrichment.json
[postgres-comment]: http://www.postgresql.org/docs/9.1/static/sql-comment.html
[cloudfront-access-log-migration]: https://github.com/snowplow/snowplow/blob/master/4-storage/redshift-storage/sql/com.amazon.aws.cloudfront/migrate_wd_access_log_1_r3_to_r4.sql
[redshift-migration]: https://github.com/snowplow/snowplow/blob/master/4-storage/redshift-storage/sql/migrate_0.6.0_to_0.7.0.sql
[postgres-migration]: https://github.com/snowplow/snowplow/blob/master/4-storage/postgres-storage/sql/migrate_0.5.0_to_0.6.0.sql
[cloudfront-migration]: https://github.com/snowplow/snowplow/blob/master/4-storage/redshift-storage/sql/com.amazon.aws.cloudfront/migrate_wd_access_log_1_r3_to_r4
[app-dl]: http://dl.bintray.com/snowplow/snowplow-generic/snowplow_emr_r71_stork_billed_kingfisher.zip

[1611]: https://github.com/snowplow/snowplow/issues/1611
[1720]: https://github.com/snowplow/snowplow/issues/1720
[1669]: https://github.com/snowplow/snowplow/issues/1839
[1905]: https://github.com/snowplow/snowplow/issues/1905
[1936]: https://github.com/snowplow/snowplow/issues/1936
[1954]: https://github.com/snowplow/snowplow/issues/1954
[1974]: https://github.com/snowplow/snowplow/issues/1974
[1950]: https://github.com/snowplow/snowplow/issues/1950

[r71-release]: https://github.com/snowplow/snowplow/releases/tag/r71-stork-billed-kingfisher
[issues]: https://github.com/snowplow/snowplow/issues
[talk-to-us]: https://github.com/snowplow/snowplow/wiki/Talk-to-us