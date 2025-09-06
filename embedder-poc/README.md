# Embed documents using ranking framework

A demo off an ONNX model based embeddings calculation using Vespa's ranking framework.

Is it achievable to get similar outcome in the document processing chain? Or even in the indexing language?

## Setup

Start Vespa in Docker:
```shell
docker run \
--detach \
--rm \
--name vespa-embedder \
--hostname vespa-embedder \
--publish 0.0.0.0:8080:8080 \
--publish 0.0.0.0:19050:19050 \
--publish 0.0.0.0:19071:19071 \
vespaengine/vespa:8.575.21
```

Deploy the application package:

```shell
vespa deploy -t http://localhost:19071
```

The ONNX model is shamelessly taken from [here](https://github.com/vespa-engine/sample-apps/blob/master/custom-embeddings/models/).

Feed the dummy document:
```shell
echo '{"id": "id:doc:doc::1", "fields": {"embedding": [1], "double_value": 2.0}}'\
| vespa feed - \
-t http://localhost:8080
```

Finally, query the index:
```shell
vespa query 'select matchfeatures from sources * where true' \
  'ranking.profile=embed' \
  'format.tensors=short' \
  -t http://localhost:8080
```

Which returns:

```json
{
  "root": {
    "id": "toplevel",
    "relevance": 1.0,
    "fields": {
      "totalCount": 1
    },
    "coverage": {
      "coverage": 100,
      "documents": 1,
      "full": true,
      "nodes": 1,
      "results": 1,
      "resultsFull": 1
    },
    "children": [
      {
        "id": "index:content/0/c4ca42388ce70a10b392b401",
        "relevance": 0.4992060363292694,
        "source": "content",
        "fields": {
          "matchfeatures": {
            "onnx_embedding": {
              "type": "tensor<float>(d0[1],d1[1])",
              "values": [
                [
                  0.4992060363292694
                ]
              ]
            }
          }
        }
      }
    ]
  }
}
```

The `onnx_embedding` has type `tensor<float>(d0[1],d1[1])` but it could be of different tensor type.

### Convert a number to some tensor shape

Check the `embed2` ranking profile where `double` value is wrapped into a tensor of the required type.

```shell
vespa query 'select matchfeatures from sources * where true' \
  'ranking.profile=embed2' \
  'format.tensors=short' \
  -t http://localhost:8080
```

```json
{
    "root": {
        "id": "toplevel",
        "relevance": 1.0,
        "fields": {
            "totalCount": 1
        },
        "coverage": {
            "coverage": 100,
            "documents": 1,
            "full": true,
            "nodes": 1,
            "results": 1,
            "resultsFull": 1
        },
        "children": [
            {
                "id": "index:content/0/c4ca42388ce70a10b392b401",
                "relevance": 0.5046800971031189,
                "source": "content",
                "fields": {
                    "matchfeatures": {
                        "onnx_embedding": {
                            "type": "tensor<float>(d0[1],d1[1])",
                            "values": [
                                [
                                    0.5046800971031189
                                ]
                            ]
                        }
                    }
                }
            }
        ]
    }
}
```

## Ideas

ONNX model can have many more inputs, and they can all be passed with query params and do not use any document feature.

## Helpers

Let's check what inputs model expects:

```shell
curl http://localhost:8080/model-evaluation/v1/custom_similarity
```

Unfortunately, the model output type is not visible in that output.

The output type can be checked using [`vespa-analyze-onnx-model`](https://docs.vespa.ai/en/operations/tools.html#vespa-analyze-onnx-model).

```shell
docker run -v `pwd`:/w \
  --entrypoint /opt/vespa/bin/vespa-analyze-onnx-model \
  vespaengine/vespa \
  /w/models/custom_similarity.onnx
```

E.g. for this model output is: 
```text
unspecified option[0](optimize model), fallback: true
vm_size: 167244 kB, vm_rss: 42752 kB, malloc_peak: 0 kb, malloc_curr: 1056 (before loading model)
vm_size: 173972 kB, vm_rss: 57224 kB, malloc_peak: 0 kb, malloc_curr: 7784 (after loading model)
model meta-data:
  input[0]: 'query' float[1][384]
  input[1]: 'document' float[1][384]
  output[0]: 'similarity' float[1][1]
test setup:
  input[0]: tensor<float>(d0[1],d1[384]) -> float[1][384]
  input[1]: tensor<float>(d0[1],d1[384]) -> float[1][384]
  output[0]: float[1][1] -> tensor<float>(d0[1],d1[1])
unspecified option[1](max concurrent evaluations), fallback: 1
vm_size: 173972 kB, vm_rss: 57352 kB, malloc_peak: 0 kb, malloc_curr: 7784 (no evaluations yet)
vm_size: 173972 kB, vm_rss: 57480 kB, malloc_peak: 0 kb, malloc_curr: 7784 (concurrent evaluations: 1)
estimated model evaluation time: 0.0145994 ms
```
