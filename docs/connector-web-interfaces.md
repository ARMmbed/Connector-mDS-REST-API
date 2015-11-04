Web Interfaces
=================================

* [Introduction](#introduction)
* [mbed DS Data Model](#mbed-ds-data-model)
* [Checking mbed DS version](#checking-mbed-ds-version)
* [Detecting the versions of REST API](#detecting-the-versions-of-rest-api)
* [Version of REST](#version-of-rest)
* [Endpoint directory lookups](#endpoint-directory-lookups)
* [Notifications](#notifications)
* [Traffic limits](#traffic-limits)

Introduction
------------

The ARM mbed Device Connector is a hosted version of the mbed Device Server designed for fast development and prototyping. It gives you a quick, secure way of connecting your devices to the cloud without the overhead of building the infrastructure to host mbed Device Server. The service makes IoT device messaging, provisioning and updates available to enterprise software, web applications and cloud stacks through easily-integrated REST APIs.

Web applications interact with mbed Device Server (mbed DS) using a set of RESTful Web interfaces over HTTP. These interfaces provide administration, lookup and eventing services. 

All REST API calls are authenticated by using the API keys (notifications, subscription, pre-subscription). 

## mbed DS Data Model
The mbed DS data model is divided into: 

- mbed users. 
- endpoints (mbed Clients).
- Resources.

Resources can be perceived as individual URI paths, for example `/path`, `/longer/path`. Several resources can be  assigned for a single endpoint. Resources are typically used to provide access to sensors, actuators and configuration parameters on an IoT device. An endpoint is the software running on a device, and it belongs to a mbed user.

In short, an mbed user has endpoints that contain resources. Several mbed users can be configured in mbed DS.

mbed DS allows endpoints and resources to be associated with semantic naming. This means that during registration, naming metadata can be associated with them. An endpoint includes a host name, which uniquely identifies it for an mbed user. An endpoint can also be associated with a node type, for example _MotionDetector_. Finally resources can be associated with a resource type (for example _LightSensor_), an interface description and other metadata about the resource, such as its content types and observability.

## Checking mbed DS version

mbed DS version can be checked with call

        GET /

**Response**

|Response|Description|
|---|---|
|`200`|Successful response containing version of mbed DS and recent REST API version it supports.|

Content-Type: text/plain

## Detecting the versions of REST API

The most recent version of mbed DS is the last element in the result list.

	GET /rest-versions

The current REST API version of mbed DS is `v1`.

**Response**

|Response|Description |
|---|---|
|`200`|Successful response with a list of version(s) supported by the server.|

Acceptable content-types: application/json

*Endpoint list json structure*

    [
        "version value 1", 
        "version value 2",
        ...
    ]

*Example:*

    GET /rest-versions
    Accept: application/json
    
    HTTP/1.1 200 OK
    Content-Type: application/json
    
    [
        "v1",
        "v2"
    ]

### Calling the latest version of the REST API

To call the latest version of a resource, you can just use the existing URI without any changes or you can prefix the latest version of the above list to your URI as follows:

*Example:*

    /endpoints
    /v2/endpoints

### Calling an old version of the REST API

To call an old version of a resource, you must prefix the desired version of the above list to your URI as follows:

*Example:*

    /v1/endpoints


## Endpoint directory lookups

The directory lookup interface provides the possibility to browse and filter through resource directory of mbed DS.

### Listing all endpoints

	GET /endpoints

**Parameters**

|Name|Type|Description |
|---|---|---|
|`type`|`string`|Filter endpoints by endpoint-type.|

**Response**

|Response|Description|
|---|---|
|`200`|Successful response with a list of endpoints.|

Acceptable content-types: 
- application/json
- application/link-format

*Endpoint list json structure*

    [
      { "name": "{endpoint name}",
        "type": "{endpoint type}",
        "status": "{endpoint status: ACTIVE|STALE}"
        "q": "{queue mode: true|false (default: false)"}
    ]

*Example:*

    GET /endpoints
    Accept: application/json
    
    HTTP/1.1 200 OK
    Content-Type: application/json
    
    [
      { "name":"node-001", "type":"Light", "status":"ACTIVE"},
      { "name":"node-003", "type":"QueueModeNode", "status":"ACTIVE", "q": true}
    ]

### Listing endpoint's resource metainformation

	GET /endpoint/{endpoint-name}

**Response**

|Response|Description |
|---|---|
|`200`|Successful response with a list of metainformation.|
|`404`|Endpoint not found.|

Acceptable content-types:
- application/json
- application/link-format

*Example:*

    GET /endpoints/node-001
    Accept: application/json
    
    HTTP/1.1 200 OK
    Content-Type: application/json
    
    [
      { "uri":"/dev/temp", "rt":"ucum:C", "obs":"true", "type":"text/plain"},
      { "uri":"/dev/illu", "obs":"false", "type":"text/plain"}
    ]

### Endpoint's resource representation

This interface provides access to endpoint's resource representation.

To access the endpoint's resource representation:

	GET, PUT, POST, DELETE /endpoints/{endpoint-name}/{resource-path}

**Parameters**

|Name|Type|Description|
|---|---|---|
|`cacheOnly`|`boolean`|If true, the response will come only from cache. Optional. Default: `false`.|
|`noResp`|`boolean`|- If `true`, not waiting for response and no response is expected. Creates CoAP Non-Confirmable requests.<br>- If `false`, response is expected and CoAP request is confirmable.<br> Optional. Default: `false`|

**Response**

|Response|Description|
|---|---|
|`200`|Successful GET, PUT, DELETE operation.|
|`201`|Successful POST operation.|
|`202`|Accepted. Asynchronous response ID.|
|`205`|No cache available for resource.|
|`404`|Requested endpoint's resource is not found.|
|`409`|Conflict. Endpoint is in queue mode and synchronous request can not be made. If noResp=true, the request is not supported.|
|`410`|Gone. Endpoint not found.|
|`412`|Request payload has been incomplete.|
|`413`|Precondition failed.|
|`415`|Media type is not supported by the endpoint.|
|`429`|Cannot make a request at the moment, already ongoing other request for this endpoint or queue is full (for endpoints in queue mode).|
|`502`|TCP or TLS connection to endpoint is not established.|
|`503`|Operation cannot be executed because endpoint is currently unavailable.|
|`504`|Operation cannot be executed due to a time-out from the endpoint.|

Acceptable content types
- application/json (for `async response id`)
- `*/*` (depends on endpoint's resource)

**Asynchronous** 

The endpoint's response arrives in the notification channel. An HTTP request will return immediately with `async-response-id`, that is used to match the response. For GET requests, if the representation is available in the cache, it will return it.

*Example:*

    GET /endpoints/node-001/dev/temp
    
    HTTP/1.1 202 Accepted
    Content-Type: application/json
    
    {"async-response-id": "23471#node-001@test.domain.com/dev/temp"}

**No response:** 

Indicated with parameter `noResp=true` and as a result, mbed DS makes CoAP Non-confirmable requests. The REST response is with status code `204 No Content`. If the underlying protocol does not support it (HTTP) or the endpoint is registered in a queue mode, the response is with status code `409 Conflict`. 

<span style="background-color:lightyellow; color:black; display:block; height:100%; padding:10px">
**Note**: This type of requests are not guaranteed.
</span>

*Example:*

    GET /endpoints/node-001/dev/temp?noResp=true
    
    HTTP/1.1 204 No Content

## Notifications

The mbed DS eventing model consists of observable resources, which enables endpoints to deliver updated resource content, periodically or with a more sophisticated solution dependent logic. Applications can subscribe to every individual resource or can set a pre-subscription data to receive a notification update.

### Subscribing to a resource

	PUT /subscriptions/{endpoint-name}/{resource-path}

**Response**

|Response|Description|
|---|---|
|`200`|Successfully subscribed.|
|`202`|Accepted. Asynchronous response ID.|
|`404`|Endpoint or its resource not found.|
|`412`|Cannot make a subscription for a non-observable resource.|
|`413`|Cannot make a subscription due to failed precondition.|
|`415`|Media type is not supported by the endpoint.|
|`429`|Cannot make subscription request at the moment due to already ongoing other request for this endpoint or (for endpoints in queue mode) queue is full or queue was cleared because endpoint made full registration.|
|`502`|Subscription failed.|
|`503`|Subscription could not be established because endpoint is currently unavailable.|
|`504`|Subscription could not be established due to a time-out from the endpoint.|

**Asynchronous** 

A subscription response will arrive in the notification channel. An HTTP request will return immediately with `asyn-response-id`, that is used to match the response. If the subscription can be established immediately, without contacting the endpoint, this call will return the subscription result.

*Example:*

    PUT /subscriptions/node-001/dev/temp
    
    HTTP/1.1 202 Accepted
    Content-Type: application/json
    
    {"async-response-id": "5734979#node-001@test.domain.com/path1"}

### Removing a subscription

	DELETE /subscriptions/{endpoint-name}/{resource-path}

**Response**

|Response|Description |
|---|---|
|`204`|Successfully removed subscription.|
|`404`|Endpoint or endpoint's resource not found.|

### Removing all subscriptions

	DELETE /subscriptions

**Response**

|Response|Description|
|---|---|
|`204`|Successfully removed subscriptions.|

### Reading subscription status

	GET /subscriptions/{endpoint-name}/{resource-path}

**Response**

|Response|Description|
|---|---|
|`200`|Resource is subscribed.|
|`404`|Resource is not subscribed.|

### Reading endpoint's subscriptions

	GET /subscriptions/{endpoint-name}

**Response**

|Response|Description |
|---|---|
|`200`|List of subscribed resources.|
|`404`|Endpoint not found or there are no subscriptions for that endpoint.|

Acceptable content-types:
- text/uri-list

*Example:*

    GET /subscriptions/node-001
    
    HTTP/1.1 200 OK
    Content-Type: text/uri-list
    
    /subscriptions/node-001/dev/temp
    /subscriptions/node-001/dev/power

### Removing endpoint's subscriptions

	DELETE /subscriptions/{endpoint-name}

**Response**

|Response|Description|
|---|---|
|`204`|Successfully removed.|
|`404`|Endpoint not found.|

### Setting pre-subscription data

Pre-subscription is a set of rules and patterns put by the application. When an endpoint registers and its name, type and registered resources match the pre-subscription data, mbed DS sends subscription requests to the device automatically. 

The pattern may include the endpoint name (optionally having an `*` character at the end), endpoint type, a list of resources or expressions with an `*` character at the end. The pre-subscription concerns all the endpoints that are already registered and the server sends subscription requests to the devices immediately when the patterns are set. The speed of the requests can be limited by the configuration parameters preventing the underlying network to be flooded or overloaded.

Changing the pre-subscription data removes all the previous subscriptions. To remove the pre-subscription data, put an empty array as a rule.

*Example*

    PUT /subscriptions
    Content-Type: application/json
    
    [
       {
         endpoint-name: "node-001",
         resource-path: ["/dev"]
       },
       {
         endpoint-type: "Light",
         resource-path: ["/sen/*"]
       },
       {
         endpoint-name: "node*"
       },
       {
         endpoint-type: "Sensor"
       },
       {
         resource-path: ["/dev/temp","/dev/hum"]
       }
    ]
    
    -->
    HTTP/1.1 200 OK

### Getting pre-subscription data

	GET /subscriptions
	
The server returns with the same JSON structure as described above. If there are no pre-subscribed resources, it returns with an empty array.

### Receiving notifications

Notifications can be delivered with the callback mechanism using a subscription server where mbed DS actively sends the notifications. 

The notifications are delivered in JSON format.

**Callback URL**

Notifications are delivered as PUT messages to the HTTP server defined by the web application with a subscription server message. The given URL should be accessible and respond to the PUT request with response code of `200` or `204`.

The web application needs the following REST request (subscription):

	PUT /notification/callback

**Response**

|Response|Description |
|---|---|
|`204`|Successfully subscribed.|
|`400`|Given URL is not accessible.|

The request body contains a JSON object including URL and headers. The headers are optional. If the header is set, the notifications will include the given headers.

The total length of url and values of headers should not be more than 400 chars, otherwise your request will fail with error code `bad request - 400`

*Example:*

    PUT /notification/callback HTTP/1.1
    Content-Type: application/json
    Content-Length: 87
    
    {"url" : "http://example.com/notification?x=123", "headers" : {"Authorization" : "auth", "test-header" : "test"}}
    
    HTTP/1.1 204 No Content

**Checking callback URL**

	GET /notification/callback

**Response**

|Response|Description|
|---|---|
|`200`|URL found.|
|`404`|Callback URL does not exist.|

*Example:*

    GET /notification/callback HTTP/1.1
    Content-Type: application/json
    Content-Length: 87
    
    {"url" : "example.com", "headers" : {"Authorization" : "auth", "test-header" : "test"}}


**Deleting callback URL**

To delete the Callback URL and remove all subscriptions:

	DELETE /notification/callback

**Response**

|Response|Description |
|---|---|
|`204`|Successfully removed.|
|`404`|Callback URL does not exist.|


### Notification data structure

A notification message contains the following server event types:

|Notification type|Description|
|---|---|
|notifications|Contains resource notifications.|
|registrations|List of endpoints that have registered (with resources).|
|reg-updates|List of endpoints that have updated registration.|
|de-registrations|List of endpoints that are removed in controlled manner.|
|registrations-expired|List of endpoints that are removed because the registration has expired.|
|async-responses|Responses to asynchronous proxy request.|

The notification message is delivered in JSON format as follows:

    {
       "notifications":[
         {
            "ep":"{endpoint-name}",
            "path":"{uri-path}",
            "ct":"{content-type}",
            "payload":"{base64-encoded-payload}",
            "timestamp":"{timestamp}"
            "max-age":"{max-age}"
         }
       ],
    
       "registrations":[
         {
           "ep": "{endpoint-name}",
           "ept": "{endpoint-type}",
           "q": "{queue-mode, default: false}",
           "resources": [ {
             "path": "{uri-path}",
             "if": "{interface-description}",
             "rf": "{resource-type}"
             "ct": "{content-type}",
             "obs": "{is-observable (true|false) }"
           } ]
         }
       ],
    
       "reg-updates":[
         {
           "ep": "{endpoint-name}"
           "ept": "{endpoint-type}"
           "q": "{queue-mode, default: false}",
           "resources": [ {
             "path": "{uri-path}",
             "if": "{interface-description}",
             "rf": "{resource-type}"
             "ct": "{content-type}",
             "obs": "{is-observable (true|false) }"
           } ]
         }
       ],
    
       "de-registrations":[
         {
           "{endpoint-name}",
           "{endpoint-name2}"
         }
       ],
    
       "registrations-expired":[
         {
           "{endpoint-name}",
           "{endpoint-name2}"
         }
       ],
    
       "async-responses":[
         {
            "id": "{async-response-id}",
            "status":  {http-status-code},
            "error":  {error-message},
            "ct":  "{content-type}",
            "max-age": {max-age},
            "payload":  "{base64-encoded-payload}"
         }
       ]
    }

## Traffic limits
The number of transactions and registered endpoints per mbed user is limited. The counter for the number of transactions includes both incoming (endpoint registrations, registration updates and removals, notification) and outgoing (proxy and subscription requets). The current status can be fetched by using the REST API.

To read the current status of limits:

	GET /limits

**Response**

|Response|Description|
|---|---|
|`200`|OK.|

*Example:*

    GET /limits
    
    HTTP/1.1 200 OK
    Content-Type: application/json
    
    { 
	  transaction-quota: 10000, 
	  transaction-count: 7845, 
	  endpoint-quota: 100, 
	  endpoint-count: 50 
	}
