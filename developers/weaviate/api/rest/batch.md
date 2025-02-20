---
title: REST - /v1/batch
sidebar_position: 13
image: og/docs/api.jpg
# tags: ['RESTful API', 'references', 'batching']
---
import Badges from '/_includes/badges.mdx';

<Badges/>

## Batch data objects

For sending data objects to Weaviate in bulk.

### Performance

💡 Import speeds, especially for large datasets, will drastically improve when using the batching endpoint. 

A few points to bear in mind:

1. If you use a vectorizer that improves with GPU support, make sure to enable it if possible, it will drastically improve import.
1. Avoid duplicate vectors for multiple data objects.
1. Handle your errors, if you ignore them, it might lead to significant delays on import.
1. If import slows down after a particular number of objects (e.g. 2M), check to see if the [`vectorCacheMaxObjects`](../../configuration/indexes.md#how-to-configure-hnsw) in your schema is larger than the number of objects. Also, see [this example](https://github.com/weaviate/semantic-search-through-wikipedia-with-weaviate/blob/d4711f2bdc75afd503ff70092c3c5303f9dd1b3b/step-2/import.py#L58-L59).
1. There are ways to improve your setup when using vectorizers. Like in the Wikipedia demo dataset. We will keep publishing about this, sign up for our [Slack channel](https://join.slack.com/t/weaviate/shared_invite/zt-goaoifjr-o8FuVz9b1HLzhlUfyfddhw) to keep up to date.

### Method and URL

```js
POST /v1/batch/objects
```

### Parameters

The body requires the following field:

| name | type | required | description |
| ---- | ---- | ---- | ---- |
| `objects` | list of data objects | yes | a list of data objects, which correspond to the data [object body](./objects.md#parameters-1) |

### Example request

import BatchObjects from '/_includes/code/batch.objects.mdx';

<BatchObjects/>

## Batching objects with the Python Client

Specific documentation for the Python client

### Additional documentation

* Additional documentation can be found on [weaviate-python-client.readthedocs.io](https://weaviate-python-client.readthedocs.io/en/stable/weaviate.batch.html)
* Additional documentation on different types of batching and tip &amp; tricks can be found [here](/developers/weaviate/client-libraries/python.md)

## Batch references

For batching cross-references between data objects in bulk.

### Method and URL

```js
POST /v1/batch/references
```

### Parameters

The body of the data object for a new object is a list of objects containing:

| name | type | required | description |
| ---- | ---- | ---- | ---- |
| `from` | Weaviate Beacon (long-form) | yes | The beacon, in the form of `weaviate://localhost/{Classname}/{id}/{cref_property_name}` |
| `to` | Weaviate Beacon (regular) | yes | The beacon, in the form of `weaviate://localhost/{className}/{id}` |

*Note: In the beacon format, you need to always use `localhost` as the host,
rather than the actual hostname. `localhost` refers to the fact that the
beacon's target is on the same Weaviate instance, as opposed to a foreign
instance.*

*Note: For backward compatibility, you can omit the class name in the
short-form beacon format that is used for `to`. You can specify it as
weaviate://localhost/&lt;id&gt;. This is, however, considered deprecated and will be
removed with a future release, as duplicate IDs across classes could mean that
this beacon is not uniquely identifiable. For the long-form beacon - used as part
of `from` - you always need to specify the full beacon, including the reference
property name.*

### Example request

import BatchReferences from '/_includes/code/batch.references.mdx';

<BatchReferences/>


For detailed information and instructions of batching in Python, click [here](https://weaviate-python-client.readthedocs.io/en/v3.0.0/weaviate.batch.html#weaviate.batch.Batch).

## Batch Delete By Query

You can use the `/v1/batch/objects` endpoint with the HTTP Verb `DELETE` to delete all objects that match a particular expression. To determine if an object is a match [a where-Filter is used](../graphql/filters.md#where-filter). The request body takes a single filter, but will delete all objects matched. It returns the number of matched objects as well as any potential errors. Note that there is a limit to how many objects can be deleted using this filter at once which is explained below.

### Maximum number of deletes per query

There is an upper limit to how many objects can be deleted using a single query. This protects against unexpected memory surges and very-long running requests which would be prone to client-side timeouts or network interruptions. If a filter matches many objects, only the first `n` elements are deleted. You can configure the amount `n` by setting `QUERY_MAXIMUM_RESULTS` in Weaviate's config. The default value is 10,000. Objects are deleted in the same order that they would be returned using the same filter in a a Get-Query. To delete more objects than the limit, run the same query multiple times, until no objects are matched anymore.

### Dry-Run before Deletion

You can use the dry-run option to see which objects would be deleted using your specified filter without deleting any objects yet. Depending on the verbosity set, you will either receive the total count of affected objects or a list of the affected IDs.

### Method and URL

```js
DELETE /v1/batch/objects
```

### Parameters

The body requires the following field:

| name | type | required | description |
| ---- | ---- | ---- | ---- |
| `match` | object | yes | an object outlining how to find the objects to be deleted, see an example below |
| `output` | string | no | optional, controls the verbosity of the output, see possible values below. Defaults to `"minimal"` |
| `dryRun` | bool | no | optional, if true, objects will not be deleted yet, but merely listed. Defaults to `false` |

#### A request body in detail

```yaml
{
  "match": {
    "class": "<ClassName>", # required
    "where": { /* where filter object */ },  # required
  },
  "output": "<output verbosity>" # Optional, one of "minimal", "verbose". Defaults to "minimal"
  "dryRun": <bool> # Optional, if true, objects will not be deleted yet, but merely listed. Defaults to "false"
}
```

Possible values for `output`

| value | effect |
| --- | --- |
| `minimal` | The result only includes counts. Information about objects is omitted if the deletes were successful. Only if an error occurred will the object be described. |
| `verbose` | The result lists all affected objects with their ID and deletion status, including both successful and unsuccessful deletes |


#### A response body in detail

```yaml
{
  "match": {
    "class": "<ClassName>",      # matches the request
    "where": { /* where filter object */ },  # matches the request
  },
  "output": "<output verbosity>" # matches the request
  "dryRun": <bool>
  "results": {
    "matches": "<int>"           # how many objects were matched by the filter
    "limit": "<int>"             # the most amount of objects that can be deleted in a single query, matches QUERY_MAXIMUM_RESULTS
    "successful": "<int>"        # how many objects were successfully deleted in this round
    "failed": "<int>"            # how many objects should have been deleted but could not be deleted
    "objects": [{                # one json object per weaviate object
      "id": "&lt;id&gt;",              # This successfully deleted object would be ommitted with output=minimal
      "status": "SUCCESS",       # Possible status values are: "SUCCESS", "FAILED", "DRYRUN"
      "error": null
    }, {
      "id": "&lt;id&gt;",              # This error'd object will always be listed, even with output=minimal
      "status": "FAILED",
      "errors": {
         "error": [{
             "message": "<error-string>"
         }]
      }
    ]}
  }
}
```


### Example request

import BatchDeleteObjects from '/_includes/code/batch.delete.objects.mdx';

<BatchDeleteObjects/>

### Error handling

When sending a batch request to your Weaviate instance, it could be the case that an error occurs. This can be caused by several reasons, for example that the connection to Weaviate is lost or that there is a mistake in a single data object that you are trying to add.

You can check if an error and what kind has occurred. 

A batch request will always return a HTTP `200` status code when a the batch request was successful. That means that the batch was successfully sent to Weaviate, and there were no issues with the connection or processing of the batch and no malformed request (`4xx` status code). However, with a `200` status code, there might still be individual failures of the data objects which are not contained in the response. Thus, a `200` status code does not guarantee that each batch item is added/created. An example of an error on an individual data object that might be unnoticed by sending a batch request without checking the individual results is: Adding an object to the batch that is in conflict with the schema (for example a non existing class name).

The following Python code can be used to handle errors on individual data objects in the batch. 

```python
import weaviate

client = weaviate.Client("http://localhost:8080")


def check_batch_result(results: dict):
  """
  Check batch results for errors.

  Parameters
  ----------
  results : dict
      The Weaviate batch creation return value, i.e. returned value of the client.batch.create_objects().
  """

  if results is not None:
    for result in results:
      if 'result' in result and 'errors' in result['result']:
        if 'error' in result['result']['errors']:
          print(result['result']['errors']['error'])

object_to_add = {
    "name": "Jane Doe",
    "writesFor": [{
        "beacon": "weaviate://localhost/f81bfe5e-16ba-4615-a516-46c2ae2e5a80"
    }]
}

with client.batch(batch_size=100, callback=check_batch_result) as batch:
  batch.add_data_object(object_to_add, "Author", "36ddd591-2dee-4e7e-a3cc-eb86d30a4303")
```

This can also be applied to adding references in batch. Note that sending batches, especially references, skips some validation on object and reference level. Adding this validation on single data objects like above makes it less likely for errors to pass without discovering. 


## More Resources

import DocsMoreResources from '/_includes/more-resources-docs.md';

<DocsMoreResources />
