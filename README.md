# Custom analyzer

Since the version 4.0 Report Portal uses a new version of an analyzer. The implementation of the analyzer was moved to the
separate service. If the default implementation does not provide a solution it can be extended by a
custom implementation of an analyzer. The default analyzer can work together with a custom one. ReportPortal has a [client](https://github.com/reportportal/service-api/blob/develop/src/main/java/com/epam/ta/reportportal/core/analyzer/client/AnalyzerServiceClient.java) that communicates
with analyzers by HTTP and it has a pretty simple interface. 

### Supported endpoints:

All supported endpoints that could be implemented in a custom analyzer are:

[Analyze launch](#analyze)
```yaml
POST
http://analyzer_host:port/_analyze
```

[Index items](#analyzer-with-processing-previous-data). Optional
```yaml
POST
http://analyzer_host:port/_index
```

[Delete indexed project](#analyzer-with-processing-previous-data). Optional
```yaml
DELETE
http://analyzer_host:port/_index/{project}
```

[Clean indexed items](#analyzer-with-processing-previous-data). Optional
```yaml
PUT
http://analyzer_host:port/_index/delete
```


### Analyze
In a custom analyzer should be implemented at least [one endpoint](https://github.com/reportportal/example-custom-analyzer/blob/945b958ca02babb90e25e072e455a4bdb34f51da/src/main/java/by/pbortnik/analyzer/controller/AnalyzerController.java#L23) that consumes request from RP with a launch to be analyzed using the next endpoint:
```yaml
POST
http://analyzer_host:port/_analyze
```
It should consume requests in the next json format:

[Implementation in Java](https://github.com/reportportal/example-custom-analyzer/blob/677f749e4de7297e9d385ca9c033aa38e9f359bc/src/main/java/by/pbortnik/analyzer/model/IndexLaunch.java)

```yaml
[
  {
    "launchId": "5a0d84a8eff46f62cfd9cbe4",                   
    # Launch id in ReportPortal
    "launchName": "test-results",  
    # Launch name in ReportPortal. In default implementation issues from the launch with
    # the same launch name have a higher priority
    "project": "analyzer",                       
    # Project name. Default implementation categorises items by project  
    "testItems": [                                            
    # Test items to be analyzed
      {
        "autoAnalyzed": false,
        # Optional, used for marking items as analyzed. In default implementation it means 
        # that the item that was analyzed by human ('false') has a higher priority
        "testItemId": "5a0d84b6eff46f62cfd9cd9f",             
        # Test item id in ReportPortal       
        "issueType": "TI001",         
        # Only items with RP status 'TO_INVESTIGATE' can be analyzed        
        "logs": [
        # Test item's logs with level "ERROR" and higher          
          {
            "logLevel": 40000,
            "message": "java.lang.AssertionError: expected..."
          } 
        ],                                                    
        "uniqueId": "auto:f9377b76a3ebede22df09fa76400788b"   
        # Test Item's unique id in RP. In default implementation issue from item 
        # with the same unique id has a higher priority
      }
    ]
  }
]
```
In the current default realization it actually sends only one launch in any case. 


Report Portal accepts the analyzed items back as a response in the next json format:

[Implementation in Java](https://github.com/reportportal/example-custom-analyzer/blob/677f749e4de7297e9d385ca9c033aa38e9f359bc/src/main/java/by/pbortnik/analyzer/model/AnalyzedItemRs.java)
```yaml
[
    {
        "test_item": "5a0d84b6eff46f62cfd9cd9f",
        # Test item id in RP        
        "issue_type": "PB001",
        # Investigated issue type that will be applied to the item
        "relevant_item": "5a1be11deff46f8b838fc5e5"
        # The most relevant item id. It is used for taking a comment 
        # and an external system issue from the most relevant item in RP
    }
]
```

As a result items are updated in ReportPortal database with a new issue, comment and reference in BTS if any are found. Also new analyzed items are indexed.

### Analyzer with processing previous data

If an analyzing alghorithm is based on the previous results, the analyzer interface also provides possibility to collect information about updated items in RP. To have that logic there should be implemented [one more endpoint](https://github.com/reportportal/example-custom-analyzer/blob/945b958ca02babb90e25e072e455a4bdb34f51da/src/main/java/by/pbortnik/analyzer/controller/AnalyzerController.java#L51):

```yaml
POST
http://analyzer_host:port/_index
```

The request contains the same list of json objects as in "_analyze" higher except of the "issueType" sould be provided by user and it should be different from "TO_INVESTIGATE". In the current implementation RP doesn't really use the [response from index](https://github.com/reportportal/example-custom-analyzer/blob/677f749e4de7297e9d385ca9c033aa38e9f359bc/src/main/java/by/pbortnik/analyzer/model/IndexRs.java) so it could be ignored (or just 'null').

There are a few more optional endpoints. They are required for the default analyzer implementation. [The first one](https://github.com/reportportal/example-custom-analyzer/blob/945b958ca02babb90e25e072e455a4bdb34f51da/src/main/java/by/pbortnik/analyzer/controller/AnalyzerController.java#L59) is for deleting all the accumulated information about specified project: 

```yaml
DELETE
http://analyzer_host:port/_index/{project}
```
[The other](https://github.com/reportportal/example-custom-analyzer/blob/945b958ca02babb90e25e072e455a4bdb34f51da/src/main/java/by/pbortnik/analyzer/controller/AnalyzerController.java#L65) is for cleaning information about the specified test items from the specified project: 

```yaml
PUT
http://analyzer_host:port/_index/delete

{
  "ids": [
    "5a1be11deff46f8b838fc5e5"
  ],
  "project": "analyzer"
}
```

### Configuring service for Consul

For correct interaction with Report Portal the analyzer must have several required tags in it's consul service configuration. 

* `analyzer=custom` 

      Marks that the current service is analyzer implementation with name 'custom'

* `analyzer_priority=0` 

      If the service is going to work together with default analyzer there should be specified the serivce's priority. 
      For basic analyzer it is 10. So if the current analyzer's resutls are more important than default's the priority 
      parameter should be lower than 10. If the value is not specified or incorrect than the priority is the 
      lowest by default.
      
* `analyzer_index=true`

      If the service processes previous results it should have the tag 'analyzer_index=true'. It does indexing of logs.
      If value is not specified or incorrect than 'false' by default.

[Java configuration example](https://github.com/reportportal/example-custom-analyzer/blob/677f749e4de7297e9d385ca9c033aa38e9f359bc/src/main/resources/application.yaml)

All included analyzers could be found by sending a request to the next endpoint:

```yaml
GET
http://reportportal_host:port/composite/info
```

![composite/info](/CompositeInfo.png?raw=true)

### Implemntation example

This repository is an example of all endpoints' implementation using Java
