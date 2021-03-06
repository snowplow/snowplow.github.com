---
layout: page
group: guides
subgroup: concepts
rank: 2
title: Event dictionaries and schemas
description: Understanding event dictionaries and schemas.
permalink: /guides/concepts/event-dictionaries-and-schemas/
redirect_from:
  - /analytics/concepts/event-dictionaries-and-schemas/
  - /analytics/event-dictionaries-and-data-models/event-dictionaries-and-schemas.html
  - /documentation/concepts/event-dictionaries-and-schemas/
---

# What is an event dictionary?

When we setup Snowplow, we need to make sure that we track all the events that are meaningful to our business, so that the data associated with those events is available in Snowplow for analysis.

When we come to analyse Snowplow data, we need to be able to look at the event data and understand, in an unambiguous way, what that data actually means i.e. what it represents.

An event dictionary is a crucial tool in both cases. It is a document that defines the universe of events that a company is interested in tracking. For each event, it defines:

1. What the event is. Often this might be illustrated e.g. with screenshots
2. What data is captured when the event occurs, that represents the event. This is a data schema for the event
3. Details on how the relevant Snowplow tracker has been setup to pass the event data into Snowplow

# What is the purpose of an event dictionary?

Event dictionaries serve a three purposes:

* They aid analysis, by making sure that everyone using the data understands what each line of data 'means'. This is especially important as companies get larger, and analysts need to crunch data that was defined and instrumented prior to the analyst joining the company.
* They aid technical setup: instrumentation (tracker setup) is driven by the event dictionary.
* They can assist both the product management and analytics development process, by ensuring that the anaytics instrumentation 'keeps up' with an evolving product.

# Examples

An example entry in an event dictionary might look like this:

## Event: perform_search

_Screenshot:_

![Perform search](http://snowplowanalytics.com/assets/img/analytics/basic-concepts/perform-search-mockup.png)

_Event schema:_

{% highlight json %}
{
    "$schema": "http://iglucentral.com/schemas/com.snowplowanalytics.self-desc/schema/jsonschema/1-0-0#",
    "description": "Schema for performing a search",
    "self": {
        "vendor": "com.acme_company",
        "name": "perform_search",
        "format": "jsonschema",
        "version": "1-0-0"
    },

    "type": "object",
    "properties": {
    	"query": {
            "type": "string",
            "maxLength": 1024
        }
    },
    "minProperties":1,
    "additionalProperties": false
}
{% endhighlight %}

_Example data:_

{% highlight javascript %}
window.snowplow('trackUnstructEvent', {
    schema: 'iglu:com.acme_company/perform_search/jsonschema/1-0-0',
    data: {
        query: 'Bruce Springsteen DVD'
    }
});
{% endhighlight %}


# Understand event dictionaries and schemas?

Then [learn about contexts](../contexts).
