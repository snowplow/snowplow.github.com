---
layout: post
title-short: Snowplow 88 Angkor Wat
title: "Snowplow 88 Angkor Wat released"
tags: [snowplow, duplicates, event duplication, de-dupe, de-duplication, dynamodb]
author: Anton
category: Releases
---

We are pleased to announce the release of [Snowplow 88 Angkor Wat][snowplow-release]. This release introduces event de-duplication functionality across different pipeline runs, powered by DynamoDB, along with an important refactoring of the batch pipeline configuration.

Read on for more information on R88 Angkor Wat, named after the [largest religious monument in the world][angkor-wat]:

1. [Cross-batch natural deduplication](/blog/2017/04/20/snowplow-r88-angkor-wat-released#crossbatch-natural-deduplication)
2. [New storage targets configuration](/blog/2017/04/20/snowplow-r88-angkor-wat-released#storage-targets)
3. [Upgrading](/blog/2017/04/20/snowplow-r88-angkor-wat-released#upgrading)
4. [Roadmap](/blog/2017/04/20/snowplow-r88-angkor-wat-released#roadmap)
5. [Getting help](/blog/2017/04/20/snowplow-r88-angkor-wat-released#help)

![angkor-wat][angkor-wat-img]

<!--more-->

<h2 id="synthetic-dedupe">1. Cross-batch natural deduplication</h2>

<h3 id="dedupe-history">1.1 The deduplication story so far</h3>

Event duplicates were problem in Snowplow pipeline since its origin and were described several times [in this blog][dupes-blog-post] and on our [discourse forum][dupes-discourse-thread]. 
As first step to solve the problem, in [R76][r76-changeable-hawk-eagle-release] we implemented in-batch natural deduplication which removed duplicates originated due to at-least-once delivery semantics in Snowplow pipeline. 
Next, in [R86][r86-petra-release] we introduced synthetic in-batch deduplication which one again drastically reduced amount of duplicates in our users' clusters and completely removed them from each particular load, but left them across distinct loads.

<h3 id="dedupe-dynamodb">1.2 Cross-batch deduplication using DynamoDB</h3>

Today we're going further and introducing new cross-batch deduplication that works with natural dupes across many loads, effectively eliminating duplicates problem.

To solve this problem across ETLs we're using [Amazon DynamoDB][amazon-dynamodb] storage, which allows to store information about events we processed in previous ETLs and incredibly fast check that event was (or was not) processed before. 
Mechanics of this process is quite simple and what is most important fast - you can find more technical information on dedicated [wiki page][shs-wiki]. 
However, unlike previous in-batch deduplications, it is not built-in into ETL job and needs to be explicitly enabled.

<h3 id="dedupe-usage">1.3 How to enable the new deduplication process</h3>

To start deduplicating events across batches you need to provide EmrEtlRunner a [duplicate storage configuration][duplicate-storage-config] via new `--targets` option. 
After EmrEtlRunner passed storage configuration to Scala Hadoop Shred job, ANTON TO EXPLAIN HOW THIS WORKS WITH SCREENSHOT OF DYNAMODB TABLE CONTENTS.

ANTON TO ADD NOTE THAT THIS IS TOTALLY OPTIONAL.

<h3 id="dedupe-usage">1.4 Cost impact of running this process</h3>

ANTON TO WRITE

<h3 id="dedupe-cold-start">1.5 Solving the cold start problem</h3>

To facilitate our users "cold start" deduplication problem we provide new [Event Manifest Populator][event-manifest-populator] Spark job, 
which loads old events into duplicate storage, which means you don't need to wait several months to get deduplication work at full power.

ANTON TO ADD SOME BASIC USAGE INFORMATION FOR COLD START

<h3 id="dedupe-roadmap">1.6 What's coming next for deduplication</h3>

ANTON TO ADD ABOUT ONE-OFF PYTHON SCRIPT TO CLEAN UP EXISTING DUPLICATES IN REDSHIFT.

ANTON TO MENTION CROSS-BATCH SYNTHETIC DUPES - NOT A PRIORITY BECAUSE...

<h2 id="storage-targets">2. New storage targets configuration</h2>

In order to simplify configuration of growing number of storage targets we replaced old all-in-one configuration approach with configuration through self-describing JSONs. This should reduce amount of mistakes made by our users due to similarity between different storages and alleviate adding new storage targets such as [Google BigQuery][bigquery] and [DashDB][dashdb-rfc].
With self-describing JSONs users now can see clear and detailed error message if they missed some property or used it incorrectly instead of huge contract violation traceback produced by EmrEtlRunner and StorageLoader before.

New storage configuration consists of separate self-describing JSON for each target.
You can find all supported targets grouped for [dedicated vendor][snowplow-storage-vendor] on Iglu Central and sample configs at [`4-storage/config`][sample-targets].

It also means that older `config.yml` is no longer valid and both EmrEtlRunner and StorageLoader need to accept `--targets` option specifying directory with storage configuration JSONs and `--resolver` option to validate these JSONs.

<h2 id="upgrading">3. Upgrading</h2>

<h3>3.1 Upgrading EmrEtlRunner and StorageLoader</h3>

The latest version of the EmrEtlRunner and StorageLoader are available from our Bintray [here][app-dl].

<h3>3.2 Creating new targets configuration</h3>

Storage targets configuration JSONs can be generated from your existing `config.yml`, using the [`3-enrich/emr-etl-runner/config/convert_targets.rb`][convert-targets] script. These files should be stored in a folder, for example called `targets`, alongside your existing `enrichments` folder.

When complete, your folder layout will look something like this:

{% highlight bash %}
snowplow_config
├── config.yml.tmpl
├── enrichments
│   ├── campaign_attribution.json
│   ├── ...
│   ├── user_agent_utils_config.json
├── iglu_resolver.json
├── targets
│   ├── duplicate_dynamodb.json.tmpl
│   ├── enriched_redshift.json.tmpl
{% endhighlight %}

<h3>3.3 Updating config.yml</h3>

1. Remove whole `storage.targets` section (leaving `storage.download.folder`) from your `config.yml` file
2. Update the `hadoop_shred` job version in your configuration YAML like so:

{% highlight yaml %}
versions:
  hadoop_enrich: 1.8.0        # UNCHANGED
  hadoop_shred: 0.11.0        # WAS 0.10.0
  hadoop_elasticsearch: 0.1.0 # UNCHANGED
{% endhighlight %}

For a complete example, see our [sample `config.yml` template][emretlrunner-config-yml] and [sample storage target templates][sample-targets].

<h3>3.4 Update EmrEtlRunner and StorageLoader scripts</h3>

1. Append the option `--targets $TARGETS_DIR` to both `snowplow-emr-etl-runner` and `snowplow-storage-loader` applications 
2. Append the option `--resolver $IGLU_RESOLVER` to `snowplow-storage-loader` application. This is required to validate the storage target configurations

<h3>3.5 Enabling cross-batch deduplication</h3>

In order to start deduplicating events you need to add a new [`dynamodb_config` target][duplicate_storage_config] to your newly created `targets` directory.

When Hadoop Shred runs, if the table doesn't exist then it will be automatically created with predefined [provisioned throughput][provisioned-throughput] 
For many cases it will just work, however, it is highly recommended to tweak provisioned throughput depending on your cluster size.

ANTON TO ADD MORE ON AUTO-CREATION OF TABLE

ANTON TO ADD MORE ON DEFAULT TTL, DEFAULT THROUGHPUTS

ANTON TO ADD MORE ON TUNING

<h2 id="roadmap">4. Roadmap</h2>

Upcoming Snowplow releases include:

* [R89 Plain of Jars][r89-plain-of-jars], which will port our Hadoop Enrich and Hadoop Shred jobs from Scalding to Apache Spark
* [R9x [HAD] 4 webhooks][r9x-webhooks], which will add support for 4 new webhooks (Mailgun, Olark, Unbounce, StatusGator)
* [R9x [HAD] GCP support pt. 1][r9x-gcp], the first phase of our support for running Snowplow real-time on Google Cloud Platform
* [R9x [HAD] EmrEtlRunner robustness][r9x-emretlrunner], continuing our work making EmrEtlRunner more reliable and modular
* [R9x [HAD] StorageLoader reboot][r9x-storageloader], which will port our StorageLoader app to Scala

<h2 id="help">5. Getting help</h2>

For more details on this release, please check out the [release notes] [snowplow-release] on GitHub.

If you have any questions or run into any problems, please [raise an issue] [issues] or get in touch with us through [the usual channels] [talk-to-us].

[angkor-wat]: https://en.wikipedia.org/wiki/Angkor_Wat
[angkor-wat-img]: /assets/img/blog/2017/02/angkor-wat.jpg

[snowplow-release]: https://github.com/snowplow/snowplow/releases/r88-angkor-wat

[amazon-dynamodb]: https://aws.amazon.com/dynamodb/
[dashdb-rfc]: http://discourse.snowplowanalytics.com/t/loading-enriched-events-into-ibm-dashdb/768
[bigquery]: https://cloud.google.com/bigquery/
[shs-wiki]: https://github.com/snowplow/snowplow/wiki/Scala-Hadoop-Shred#33-cross-batch-natural-de-duplication
[provisioned-throughput]: http://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ProvisionedThroughput.html

[snowplow-storage-vendor]: https://github.com/snowplow/iglu-central/tree/master/schemas/com.snowplowanalytics.snowplow.storage

[convert-targets]: https://github.com/snowplow/snowplow/blob/master/3-enrich/emr-etl-runner/config/convert_targets.rb
[sample-targets]: https://github.com/snowplow/snowplow/blob/master/4-storage/config/targets
[duplicate-storage-config]: https://github.com/snowplow/snowplow/wiki/Configuring-storage-targets#dynamodb
[event-manifest-populator]: https://github.com/snowplow/snowplow/blob/master/5-data-modeling/event-manifest-populator

[r76-changeable-hawk-eagle-release]: http://snowplowanalytics.com/blog/2016/01/26/snowplow-r76-changeable-hawk-eagle-released/
[r86-petra-release]: http://snowplowanalytics.com/blog/2016/12/20/snowplow-r86-petra-released/
[dupes-blog-post]: http://snowplowanalytics.com/blog/2015/08/19/dealing-with-duplicate-event-ids/
[dupes-discourse-thread]: http://discourse.snowplowanalytics.com/t/de-deduplicating-events-in-hadoop-and-redshift-tutorial/248

[app-dl]: http://dl.bintray.com/snowplow/snowplow-generic/snowplow_emr_r88_angkor_wat.zip
[emretlrunner-config-yml]: https://github.com/snowplow/snowplow/blob/master/3-enrich/emr-etl-runner/config/config.yml.sample

[r89-plain-of-jars]: https://github.com/snowplow/snowplow/milestone/137
[r9x-webhooks]: https://github.com/snowplow/snowplow/milestone/129
[r9x-gcp]: https://github.com/snowplow/snowplow/milestone/138
[r9x-emretlrunner]: https://github.com/snowplow/snowplow/milestone/141
[r9x-storageloader]: https://github.com/snowplow/snowplow/milestone/121

[issues]: https://github.com/snowplow/snowplow/issues/new
[talk-to-us]: https://github.com/snowplow/snowplow/wiki/Talk-to-us
