Expiremental Highlighter [![Build Status](https://travis-ci.org/nik9000/expiremental-highlighter.svg?branch=master)](https://travis-ci.org/nik9000/expiremental-highlighter)
========================

Text highlighter for Java designed to be pluggable enough for easy
expirementation.  The idea being that it should be possible to play with how
hits are weighed or how they are grouped into snippets without knowing about
the guts of Lucene or Elasticsearch.

Comes in three flavors:
* Core: No dependencies jar containing most of the interesting logic
* Lucene: A jar containing a bridge between the core and lucene
* Elasticsearch: An Elasticsearch plugin


Elasticsearch value proposition
-------------------------------
This highlighter
* Doesn't need offsets in postings or term enums with offsets but can use
either to speed itself up.
* Can fragment like the Postings Highlighter, the Fast Vector Highlighter,
or it can highlight the entire field.
* Combine hits using multiple different fields (aka ```matched_field```
support).
* Boost matches that appear early in the document.

This highlighter does not (currently):
* Respect phrase matches at all (all phrases are reduced to terms)
* Support require_field_match

Elasticsearch installation
--------------------------

| Expiremental Highlighter Plugin |  ElasticSearch  |
|---------------------------------|-----------------|
| master                          | 1.0.1 -> master |

At this point nothing has been pushed to Elasticsearch's plugin repository so
you have to clone the plugin locally, build it by going to the cloned directory
and
```bash
mvn clean package
export ABSOLUTE_PATH_OF_CLONED_DIRECTORY=$(pwd)
```
then install by going to the root of the Elasticsearch installation and
```bash
./bin/plugin --url file:///$ABSOLUTE_PATH_OF_CLONED_DIRECTORY/expiremental-highlighter-elasticsearch-plugin/target/releases/expiremental-highlighter-elasticsearch-plugin-0.0.1-SNAPSHOT.zip  --install expiremental-highlighter-elasticsearch-plugin 
```

Then you can use it by searching like so:
```js
{
  "_source": false,
  "query": {
    "query_string": {
      "query": "hello world"
    }
  },
  "highlight": {
    "order": "score",
    "fields": {
      "title": {
        "number_of_fragments": 1,
        "type": "expiremental"
      }
    }
  }
}
```

Elasticsearch options
---------------------
The ```fragmenter``` field to defaults to ```scan``` but can also be set to
```sentence``` or ```none```.  ```scan``` produces results that look like the
Fast Vector Highlighter.  ```sentence``` produces results that look like the
Postings Highlighter.  ```none``` won't fragment on anything so it is cleaner
if you have to highlight the whole field.  Multi-valued fields will always
fragment between each value, even on ```none```.  Example:
```js
  "highlight": {
    "fields": {
      "title": {
        "type": "expiremental",
        "fragmenter": "sentence"
      }
    }
  }
```

The ```default_similarity``` option defaults to true for queries with more then
one term.  It will weigh each matched term using Lucene's default similarity
model similarly to how the Fast Vectory Highlighter weighs terms.  If can be
set to false to leave out that weighing.  If there is only a single term in the
query it will never be used.
```js
  "highlight": {
    "fields": {
      "title": {
        "type": "expiremental",
        "options": {
          "default_similarity": false
        }
      }
    }
  }
```

The ```hit_source``` option can force detecting matched terms from a particular
source.  It can be either ```postings```, ```vectors```, or ```analyze```.  If
set to ```postings``` but the field isn't indexed with ```index_options``` set
to ```offsets``` or set to ```vectors``` but ```term_vector``` isn't set to
```positions_offsets``` then the highlight throw back an error.  Defaults to
using the first option that wouldn't throw an error.
```js
  "highlight": {
    "fields": {
      "title": {
        "type": "expiremental",
        "options": {
          "hit_source": "analyze"
        }
      }
    }
  }
```

The ```boost_before``` option lets you set up boosts before positions.  For
example, this will multiply the weight of matches before the 20th position by
5 abd before the 100th position by 1.5.
```js
  "highlight": {
    "fields": {
      "title": {
        "type": "expiremental",
        "options": {
          "boost_before": {
            "20": 5,
            "100": 1.5
          }
        }
      }
    }
  }
```
Note that the position is not reset between multiple values of the same field
but is handled independently for each ```matched_field```.

The ```matched_fields``` field turns on combining matches from multiple fields,
just like the Fast Vector Highlighter.  See the [Elasticsearch documentation](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/search-request-highlighting.html#matched-fields)
for more on it.  The only real difference is that if ```hit_source``` is left
out then each field's HitSource is determined independently if .  If one field
is short feel free to leave out any special settings for ```index_options``` or
for ```term_vector```s.

If you aren't using Elasticsearch, you can combine hits from multiple sources
using:
```java
new OverlapMergingHitEnumWrapper(new MergingHitEnum(hitsToMerge, HitEnum.LessThans.OFFSETS));
```

Offsets in postings or term vectors
-----------------------------------
Since adding offsets to the postings (set ```index_options``` to ```offsets```
in Elasticsearch) and creating term vectors with offsets (set ```term_vector```
to ```positions_offsets``` in Elasticsearch) both act to speed up highligting
of this highlighter you have a choice which to use.  Unless you have a
compelling reason go with adding offsets to the postings.  That is faster (by
my tests) and uses much less space.