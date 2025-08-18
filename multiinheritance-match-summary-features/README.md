# Try out multiple inheritance of summary and match features

Test this [issue](https://github.com/vespa-engine/vespa/issues/34189).

Start Vespa in Docker:
```shell
docker run \
--detach \
--rm \
--name vespa-multipleinheritance \
--hostname vespa-multipleinheritance \
--publish 0.0.0.0:8080:8080 \
--publish 0.0.0.0:19050:19050 \
--publish 0.0.0.0:19071:19071 \
vespaengine/vespa:8.565.17
```

Deploy the application package:
```shell
vespa deploy -t http://localhost:19071 
```

Feed in the document:
```shell
echo '{"id": "id:doc:doc::1", "fields": {"data_1": 1, "data_2": 10, "data_3": 100}}'\
| vespa feed - \
-t http://localhost:8080
```

Finally, query the index:
```shell
 vespa query 'select * from sources * where true' \
 'ranking.profile=final_1' \
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
        "id": "id:doc:doc::1",
        "relevance": 111.0,
        "source": "content",
        "fields": {
          "matchfeatures": {
            "attribute(data_1)": 1.0,
            "attribute(data_2)": 10.0,
            "attribute(data_3)": 100.0
          },
          "sddocname": "doc",
          "documentid": "id:doc:doc::1",
          "data_1": 1,
          "data_2": 10,
          "data_3": 100,
          "summaryfeatures": {
            "attribute(data_1)": 1.0,
            "attribute(data_2)": 10.0,
            "attribute(data_3)": 100.0,
            "vespa.summaryFeatures.cached": 0.0
          }
        }
      }
    ]
  }
}
```

Yes, the match and summary features are inherited from multiple ranking profiles.