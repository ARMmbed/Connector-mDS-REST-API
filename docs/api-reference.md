## API Reference

If you're unfamiliar with mbed Device Connector please read our short [introduction](/docs/v5.4/device-connector-api/index.html) to general concepts.

The mbed Device Connector Web API offers functions that manage:

* [Versions](#versions)
* [Endpoints](#endpoints)
* [Resources](#resources)
* [Notifications](#notifications)
* [Subscriptions](#subscriptions)
* [Traffic limits](#traffic-limits)

The [Device Connector developer tools](#developer-tools) can be used to:

* [Create a new device certificate](#create-a-certificate)
* [Create a new access key](#create-an-access-key)

__A note about asynchronous functions__

A number of functions in the mbed Device Connector API are asynchronous, because it's not guaranteed that an action (such as writing to a device) will happen straight away, as the device might be in deep sleep. These APIs are marked with '(async)' in the API reference. For information about handling asynchronous functions, see: [Asynchronous requests](/docs/restructure/legacy-products/mbed-device-connector-web-api.html#asynchronous-requests).

### Versions

#### Detecting the versions of REST API

To check the version of the mbed Device Connector API:
```
GET /rest-versions
```

The most recent version of the API is the last element in the result list.

**Response**

Response|Description
---|---
200|Successful response with a list of versions supported by the server.

Acceptable content-types:

- application/json

**Example**

```
GET /rest-versions
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

[
    "v1",
    "v2"
]
```

#### Calling the latest version of the REST API

To call the latest version of the API, you can either use the existing URI without any changes, or you can prefix the latest version of the above list to your URI:

**Example**

```
/some-url
/v2/some-url
```

#### Calling an old version of the REST API

To call an old version of the API, you must prefix the desired version of the above list to your URI:

**Example**

```
/v1/some-url
```

### Endpoints

Endpoints are physical devices running [mbed Client](https://www.mbed.com/en/development/software/mbed-client/). For more information about mbed Device Connector’s data model, see the [REST API introduction](/docs/restructure/legacy-products/mbed-device-connector-web-api.html#the-mbed-device-connector-data-model).

#### Listing all endpoints

```
GET /v2/endpoints
```

**Query string parameters**

Name|Type|Description
---|---|---
type|`string`|Filter endpoints by endpoint-type.

**Response**

Response|Description
---|---
200|Successful response with a list of endpoints.

Acceptable content-types:

- application/json
- application/link-format

*Endpoint list JSON structure*

```json
[{
  "name": "{endpoint name, set by the client in M2MInterfaceFactory::create_interface}",
  "type": "{endpoint type, set by the client in M2MInterfaceFactory::create_interface}",
  "status": "{endpoint status: always ACTIVE, but reserved for future use}",
  "q": "{queue mode: true|false (default: false)}"
}]
```

**Example**

```
GET /v2/endpoints
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

[
  { "name":"node-001", "type":"Light", "status":"ACTIVE"},
  { "name":"node-003", "type":"QueueModeNode", "status":"ACTIVE", "q": true}
]
```

##### Queue mode

When an endpoint is in queue mode, messages sent to the endpoint will not wake up the physical device. Rather, the messages are queued and delivered when the device wakes up and connects to mbed Device Connector itself. Queue mode can also be used when the device is behind a NAT and cannot be reached directly by mbed Device Connector.

#### Listing an endpoint's resources

```
GET /v2/endpoints/{endpoint-name}
```

The list of resources are cached by mbed Device Connector as long as the device remains registered, and this call does not wake up the device.

<span class="notes">**Note:** `endpoint-name` needs to be an exact match of the name of the endpoint. It is not possible to use wildcards here.</span>

**Response**

Response|Description
---|---
200|Successful response with a list of resources.
404|Endpoint not found.

Acceptable content-types:

- application/json
- application/link-format

*Endpoint resources JSON structure*

```json
[{
  "uri": "{URL for this resource}",
  "rt": "{resource type}",
  "obs": "{observable, whether you can subscribe to changes for this resource: true|false}",
  "type": "{content type}"
}]
```

**Example**

```
GET /v2/endpoints/node-001
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

[
  { "uri":"/actuator/0/speed", "rt":"5851", "obs":"true", "type":"text/plain"},
  { "uri":"/led/1/state", "rt":"5850", "obs":"false"}
]
```

*You are encouraged to use the [LwM2M](http://technical.openmobilealliance.org/Technical/technical-information/omna/lightweight-m2m-lwm2m-object-registry) resource types.*

### Resources

All APIs related to device resources are [asynchronous](/docs/restructure/legacy-products/mbed-device-connector-web-api.html#asynchronous-requests). Be aware that these APIs will only respond if the device is turned on and connected to mbed Device Connector. You can also receive [notifications](#notifications) when a resource changes by subscribing to the resource.

#### Non-confirmable requests

All resource APIs have the parameter `noResp`. If a request is made with `noResp=true`, mbed Device Connector makes a CoAP non-confirmable requests to the device. This means that these requests are not guaranteed to arrive at the device, nor do you get an `async-response-id` back.

If calls with this parameter enabled succeed, they return with the status code `204 No Content`. If the underlying protocol does not support non-confirmable requests, or if the endpoint is registered in queue mode, the response is status code `409 Conflict`.

#### Reading from a resource ([async](/docs/restructure/legacy-products/mbed-device-connector-web-api.html#asynchronous-requests))

```
GET /v2/endpoints/{endpoint-name}/{resource-path}
```

**Query string parameters**

|Name|Type|Description|
|---|---|---|
|cacheOnly|`boolean`|If true, the response will come only from cache. Optional. Default: `false`.|
|noResp|`boolean`|See [Non-confirmable requests](#non-confirmable-requests)|

**Response**

|Response|Description|
|---|---|
|200|Successful GET operation.|
|202|Accepted. Returns an asynchronous response ID.|
|205|No cache available for resource.|
|404|Requested endpoint's resource is not found.|
|409|Conflict. Endpoint is in queue mode and synchronous request can not be made. If noResp=true, the request is not supported.|
|410|Gone. Endpoint not found.|
|412|Request payload was not valid.|
|413|Precondition failed.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|

Acceptable content types:

- application/json

*Resource JSON structure*

```
[{
  "id": "{async response ID}",
  "status": "{HTTP status code, for example 200 for OK}",
  "payload": "{requested data, base64 encoded}",
  "ct": "{content type}",
  "max-age": {how long this value will be valid, in seconds}
}]
```

**Example**

```
GET /v2/endpoints/node-001/actuator/0/speed
Accept: application/json

HTTP/1.1 200 OK
Content-Type: application/json

{"async-response-id": "1073741829#521f9d17-c5d7-4769..."}
```

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "1073741829#521f9d17-c5d7-4769...",
    "status": 200,
    "payload": "MTI=",
    "ct": "text/plain",
    "max-age": 0
  }]
}
```

#### Writing to a resource ([async](/docs/restructure/legacy-products/mbed-device-connector-web-api.html#asynchronous-requests))

This API can be used to write new values to existing resources, or to create new resources on the device. The `resource-path` does not have to exist - it can be created by the call.

```
PUT /v2/endpoints/{endpoint-name}/{resource-path}
```

**Query string parameters**

|Name|Type|Description|
|---|---|---|
|noResp|`boolean`|See [Non-confirmable requests](#non-confirmable-requests)|


**Response**

|Response|Description|
|---|---|
|202|Accepted. Returns an asynchronous response ID.|
|409|Conflict. Endpoint is in queue mode and synchronous request can not be made. If noResp=true, the request is not supported.|
|410|Gone. Endpoint not found.|
|412|Request payload was not valid.|
|413|Precondition failed.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|

Acceptable content types:

- `*/*` (depends on endpoint's resource)

*Resource JSON structure*

```
[{
  "id": "{async response ID}",
  "status": "{HTTP status code, for example 200 for OK}",
  "payload": "",
  "ct": "{content type}",
  "max-age": {how long this value will be valid, in seconds}
}]
```

**Example**

```
    PUT /v2/endpoints/node-001/actuator/0/speed
    Accept: application/json
    Content-Type: text/plain

    95

    HTTP/1.1 200 OK
    Content-Type: application/json

    {"async-response-id": "1073741827#521f9d17-c5d7-4769..."}
```

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "1073741827#521f9d17-c5d7-4769...",
    "status": 200,
    "payload": "",
    "ct": "text/plain",
    "max-age": 60
  }]
}
```

**Handling the update in mbed Client**

The [mbed Client overview](/docs/restructure/legacy-products/features.html) contains information on how to process updates on the device side.

#### Executing a function on a resource ([async](/docs/restructure/legacy-products/mbed-device-connector-web-api.html#asynchronous-requests))

This API can be used to execute functions on existing resources on the device.

```
POST /v2/endpoints/{endpoint-name}/{resource-path}
```

**Query string parameters**

|Name|Type|Description|
|---|---|---|
|noResp|`boolean`|See [Non-confirmable requests](#non-confirmable-requests)|

The body of the request  is passed in as a `char*` to the function in mbed Client.

**Response**

|Response|Description|
|---|---|
|202|Accepted. Returns an asynchronous response ID.|
|404|Requested endpoint's resource is not found.|
|409|Conflict. Endpoint is in queue mode and synchronous request cannot be made. If noResp=true, the request is not supported.|
|410|Gone. Endpoint not found.|
|412|Request payload was not valid.|
|413|Precondition failed.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling ongoing another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|


Acceptable content types:

- `*/*` (depends on endpoint's resource)

*Resource JSON structure*

```
[{
  "id": "{async response ID}",
  "status": "{HTTP status code, for example 200 for OK}",
  "payload": "",
  "ct": "{content type}",
  "max-age": {how long this value will be valid, in seconds}
}]
```

**Example**

```
    POST /v2/endpoints/node-001/actuator/0/drive
    Accept: application/json
    Content-Type: text/plain

    Data that will be passed to the function on the device

    HTTP/1.1 202 OK
    Content-Type: application/json

    {"async-response-id": "1073741827#521f9d17-c5d7-4769..."}
```

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "1073741827#521f9d17-c5d7-4769...",
    "status": 200,
    "payload": "",
    "ct": "text/plain",
    "max-age": 60
  }]
}
```

**Handling the execute operation in mbed Client**

The [mbed Client overview](/docs/restructure/legacy-products/features.html) contains information on how to handle the execute operation on the device side.

#### Deleting a resource ([async](/docs/restructure/legacy-products/mbed-device-connector-web-api.html#asynchronous-requests))

A request to delete a resource must be handled by both mbed Client and mbed Device Connector. The resource is not deleted from mbed Device Connector until the delete is handled by mbed Client.

```
DELETE /v2/endpoints/{endpoint-name}/{resource-path}
```

|Name|Type|Description|
|---|---|---|
|noResp|`boolean`|See [Non-confirmable requests](#non-confirmable-requests)|


**Response**

|Response|Description|
|---|---|
|202|Accepted. Returns an asynchronous response ID.|
|404|Requested endpoint's resource is not found.|
|409|Conflict. Endpoint is in queue mode and synchronous request can not be made. If noResp=true, the request is not supported.|
|410|Gone. Endpoint not found.|
|412|Request payload was not valid.|
|413|Precondition failed.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling ongoing another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|


Acceptable content types:

- application/json

*Resource JSON structure*

```
[{
  "id": "{async response ID}",
  "status": "{HTTP status code, for example 200 for OK}",
  "payload": "",
  "ct": "{content type}",
  "max-age": {how long this value will be valid, in seconds}
}]
```

**Example**

```
DELETE /v2/endpoints/node-001/logs/0/debug
Accept: application/json
Content-Type: text/plain

HTTP/1.1 202 OK
Content-Type: application/json

{"async-response-id": "1073741827#521f9d17-c5d7-4769..."}
```

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "1073741827#521f9d17-c5d7-4769...",
    "status": 200,
    "payload": "",
    "ct": "text/plain",
    "max-age": 60
  }]
}
```

**Handling the delete operation in mbed Client**

The [mbed Client overview](/docs/restructure/legacy-products/features.html#the-device-management-and-service-enabler-feature) contains information on how to handle the delete operation on the device side.

### Notifications

Notifications are created when certain events happen on mbed Device Connector:

- A response for an asynchronous request is ready.
- A device reports about a change in its resource state (if a subscription exists for the resource).
- A device registers, re-registers or unregisters, or the registration expires.

A notification is created for each asynchronous request when the response is ready, for example when the device sends a response for a read request. Another case is when you [subscribe](#subscriptions) to a resource, mbed Device Connector starts creating notifications when the resource state changes. The device registration related notifications are always created automatically.

#### Receiving notifications

There are two ways of receiving notifications:

* [Register a notification callback](#registering-a-notification-callback).
* [Use long polling](#long-polling).

mbed Device Connector tries to deliver each notification event for up to 15 minutes after which the event is discarded if it is not delivered. 

We'll review the notification data structure before looking at the registration and notification functions.

##### Notification data structure

A notification message contains the following server event types:

|Notification type|Description|
|---|---|
|notifications|Contains resource notifications. **Note that the payload is base64-encoded.**|
|registrations|List of new endpoints that have registered with mbed Device Connector (with resources).|
|reg-updates|List of endpoints that have updated registration.|
|de-registrations|List of endpoints that were removed in a controlled manner.|
|registrations-expired|List of endpoints that were removed because the registration has expired.|
|async-responses|Responses to asynchronous proxy request. **Note that the payload is base64-encoded.**|

The notification message is delivered in JSON format:

```
    {
       "notifications":[
         {
            "ep":"{endpoint-name}",
            "path":"{uri-path}",
            "ct":"{content-type}",
            "payload":"{base64-encoded-payload}",
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
```

#### Registering a notification callback

##### Registering a callback URL

Register a URL to which the server should deliver notifications.

```
PUT /v2/notification/callback
```

<span class="notes">**Note:** Only one URL per access key can be active. If you register a new URL when another is already active, the old URL is replaced by the new.</span>

**Body**

The body of the request needs to be a JSON object with the following format:

|Name|Type|Description|
|---|---|---|
|url|`string`|The URL to which notifications need to be sent. We recommend that you serve this URL over HTTPS.|
|headers|`object`|Headers (key/value) that need to be sent with the request. Optional.|

<span class="notes">**Note:** The total length of the URL and header values must be no more than 400 characters.</span>

The values in the `headers` field are sent as HTTP headers with every request that mbed Device Connector makes to your URL. You could use this to verify that a request originated from mbed Device Connector.

**Response**

|Response|Description |
|---|---|
|204|Successfully subscribed.|
|400|Given URL is not accessible.|

When you register a URL, mbed Device Connector sends a PUT request to that URL immediately.

The server responds with `400 Bad Request` if:
* The URL you passed in is not reachable.
* The total length of the URL and header values is more than 400 characters.

<span class="tips">**Tip:** The response body includes information on why the request failed.</span>

**Example:**

```
PUT /v2/notification/callback HTTP/1.1
Content-Type: application/json
Content-Length: 87

{"url" : "http://example.com/notification?x=123", "headers" : {"Authorization" : "auth", "test-header" : "test"}}

HTTP/1.1 204 No Content
```

##### Checking the notification callback

Returns the notification callback URL and the headers that have been set in [Registering a notification callback](#registering-a-notification-callback).

```
GET /v2/notification/callback
```

**Response**

|Response|Description|
|---|---|
|200|URL found.|
|404|No callback URL is registered.|

**Example**

```
GET /v2/notification/callback 

HTTP/1.1
Content-Type: application/json
Content-Length: 87

{"url" : "example.com", "headers" : {"Authorization" : "auth", "test-header" : "test"}}
```

##### Deleting the notification callback

To delete the Callback URL and remove all subscriptions:

```
DELETE /notification/callback
```

**Response**

|Response|Description |
|---|---|
|204|Successfully removed.|
|404|No callback URL is registered.|

#### Long polling

As an alternative to the notification callback, you can use HTTP long-poll requests to receive notifications. You open a request from the client to mbed Device Connector, and the request will be kept open until either an event notification is delivered to the client or the request times out (after 30 seconds). In both cases, the client should open a new polling request after the previous one closes.

```
GET /v2/notification/pull
```

<span class="notes">**Note:** It is mandatory to create a persistent connection, by sending the `Connection: keep-alive` header, to avoid excessive TLS handshakes.</span>

**Response**

|Response|Description |
|---|---|
|200|OK.|
|204|No new notifications.|

### Subscriptions

When you have set up a [notification callback](#registering-a-notification-callback), your application can subscribe to either individual resources or pre-subscribe to resources.

When you subscribe to an observable resource, the server notifies your application when the resource changes. Your application can either [subscribe to an individual resource](#subscriptions), or use [pre-subscriptions](#automatically-subscribe-to-resources) to define rules that subscribe automatically to any resources that match those rules.

Whether or not a resource is observable is determined by [mbed Client](/docs/restructure/legacy-products/features.html), not by mbed Device Connector.

#### Subscribing to an individual resource ([async](/docs/restructure/legacy-products/mbed-device-connector-web-api.html#asynchronous-requests))

This function subscribes to an individual resource.

```
PUT /v2/subscriptions/{endpoint-name}/{resource-path}
```

**Response**

|Response|Description|
|---|---|
|200|Successfully subscribed.|
|202|Accepted. Asynchronous response ID.|
|404|Endpoint or its resource not found.|
|412|Cannot make a subscription for a non-observable resource.|
|413|Cannot make a subscription due to failed precondition.|
|415|Media type is not supported by the endpoint.|
|429|Cannot make a request at the moment; already handling another request for this endpoint or the queue is full (for endpoints in queue mode).|
|502|TCP or TLS connection to endpoint cannot be established.|
|503|Operation cannot be executed because endpoint is currently unavailable.|
|504|Operation cannot be executed due to a time-out from the endpoint.|

**Example:**

```
PUT /v2/subscriptions/node-001/piezo/0/heart-rate

HTTP/1.1 202 Accepted
Content-Type: application/json

{"async-response-id": "5734979#node-001@test.domain.com/path1"}
```

When the response is available, on your notification channel:

```json
{
  "async-responses": [{
    "id": "5734979#node-001@test.domain.com/path1",
    "status": 200,
    "payload": "",
    "ct": "text/plain",
    "max-age": 60
  }]
}
```

#### Removing a subscription

```
DELETE /subscriptions/{endpoint-name}/{resource-path}
```

**Response**

|Response|Description |
|---|---|
|204|Successfully removed subscription.|
|404|Endpoint or endpoint's resource not found.|

#### Removing all subscriptions

```
DELETE /v2/subscriptions
```

**Response**

|Response|Description|
|---|---|
|204|Successfully removed subscriptions.|

#### Checking resource subscription status

```
GET /v2/subscriptions/{endpoint-name}/{resource-path}
```

**Response**

|Response|Description|
|---|---|
|200|Resource is subscribed.|
|404|Resource is not subscribed.|

Acceptable content-types:

- text/uri-list

**Example:**

```
GET /v2/subscriptions/node-001/dev/power

HTTP/1.1 200 OK
Content-Type: text/uri-list

/subscriptions/node-001/dev/power
```

#### Reading endpoint's subscriptions

```
GET /v2/subscriptions/{endpoint-name}
```

**Response**

|Response|Description|
|---|---|
|200|List of subscribed resources.|
|404|Endpoint not found or there are no subscriptions for that endpoint.|

Acceptable content-types:

- text/uri-list

**Example:**

```
GET /v2/subscriptions/node-001

HTTP/1.1 200 OK
Content-Type: text/uri-list

/subscriptions/node-001/dev/temp
/subscriptions/node-001/dev/power
```

#### Removing endpoint's subscriptions

```
DELETE /v2/subscriptions/{endpoint-name}
```

**Response**

|Response|Description|
|---|---|
|204|Successfully removed.|
|404|Endpoint not found.|

#### Automatically subscribe to resources

Besides manually subscribing to resources, you can also automatically subscribe (a pre-subscription) to resources based on their endpoint name, endpoint type, or listed resources. mbed Device Connector will automatically send a subscription request when it encounters an endpoint or resource that matches these definitions.

```
PUT /v2/subscriptions
```

**Body**

The body has to be a JSON array that contains objects that may have the following fields:

|Name|Type|Description|
|---|---|---|
|endpoint-name|`string`|The endpoint name (optionally having an `*` character at the end).|
|endpoint-type|`string`|The endpoint type.|
|resource-path|`Array`|A list of resources to subscribe to. Allows wildcards to subscribe to multiple resources at once.|

If you send an empty array, the pre-subscription data will be removed.

<span class="notes">**Note:** Changing the pre-subscription data removes *all* the existing subscriptions, this includes individual subscriptions.</span>

**Response**

|Response|Description|
|---|---|
|204|Successfully set pre-subscription data.|
|400|Malformed content.|

**Example**

```
    PUT /v2/subscriptions
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
```

#### Getting automatic subscriptions

```
GET /v2/subscriptions
```

The server returns with the same JSON structure as described above. If there are no pre-subscribed resources, it returns with an empty array.

### Traffic limits

The number of transactions and registered endpoints for each mbed user is limited per 24 hours. The counter for the number of transactions includes both incoming (endpoint registrations, registration updates and removals, notifications) and outgoing (proxy and subscription requests).

To read the current value of the limit counter:

```
GET /v2/limits
```

**Response**

|Response|Description|
|---|---|
|200|OK|

**Example**

```
GET /v2/limits

HTTP/1.1 200 OK
Content-Type: application/json

{
    "transaction-quota": 10000,
    "transaction-count": 7845,
    "endpoint-quota": 100,
    "endpoint-count": 50
}
```
    
### Developer tools

The REST methods described below help automation of applications. Access key authorization is required.

<span class="notes">**Note:** A different base URL is required to use these tools. The services can be accessed only from URL https://connector.mbed.com.</span>

#### Create a certificate

##### Creating a new certificate for an endpoint

Besides requesting for the `security.h` file from the mbed Connector webapp, the security credentials can be generated programmatically using CSR process. For such request, you need to have the Elliptic Curve private-public keys created. Secp256r1 is recommended, but other 256 bit curves work as well. The public key is sent along with your domain ID in the JSON body as a PEM format string. When you have received the domain and endpoint names and the server and device certificates along with the successful response, you must formulate the `security.h` file manually. The file needs to include the private key that pairs with the public key sent in CSR request.

To create the endpoint credentials:

```
POST /api/cert
```

**Body**

The body needs to be a JSON object that contains the following fields:

|Name|Type|Description|
|---|---|---|
|domain|`string`| The ID of your domain.|
|publickey|`string`| The public key of your Elliptic Curve key pair, in PEM format.|

Both fields are mandatory.

**Response**

Response|Description
---|---
201|Created, successful response with credential details.
401|Unauthorized, provided access key is not valid for domain.
400|Bad Request, error parsing JSON payload or error reading the public key.

Acceptable content-types:

- application/json

**Example**

```
    POST /api/cert
    Accept: application/json
    Content-Type: application/json

    {
        "domain": "012345-6789-0123-45678901",
        "publickey": "-----BEGIN PUBLICKEY-----
        MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAryQICCl6NZ5gDKrnSztO
        3Hy8PEUcuyvg/ikC+VcIo2SFFSf18a3IMYldIugqqqZCs4/4uVW3sbdLs/6PfgdX
        7O9D22ZiFWHPYA2k2N744MNiCD1UE+tJyllUhSblK48bn+v1oZHCM0nYQ2NqUkvS
        OrUZ/wK69Dzu4IvrN4vs9Nes8vbwPa/ddZEzGR0cQMt0JBkhk9kU/qwqUseP1QRJ
        5I1jR4g8aYPL/ke9K35PxZWuDp3U0UPAZ3PjFAh+5T+fc7gzCs9dPzSHloruU+gl
        FQIDAQAB
        -----END PUBLIC KEY-----"
    }

    HTTP/1.1 201 Created
    Content-Type: application/json

    {
        "domain": "012345-6789-0123-45678901",
        "endpoint": "987654-3210-9876-54321098",
        "server_certificate": "-----BEGIN CERTIFICATE-----
        MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAryQICCl6NZ5gDKrnSztO
        3Hy8PEUcuyvg/ikC+VcIo2SFFSf18a3IMYldIugqqqZCs4/4uVW3sbdLs/6PfgdX
        7O9D22ZiFWHPYA2k2N744MNiCD1UE+tJyllUhSblK48bn+v1oZHCM0nYQ2NqUkvS
        3Hy8PEUcuyvg/ikC+VcIo2SFFSf18a3IMYldIugqqqZCs4/4uVW3sbdLs/6PfgdX
        7O9D22ZiFWHPYA2k2N744MNiCD1UE+tJyllUhSblK48bn+v1oZHCM0nYQ2NqUkvS
        3Hy8PEUcuyvg/ikC+VcIo2SFFSf18a3IMYldIugqqqZCs4/4uVW3sbdLs/6PfgdX
        7O9D22ZiFWHPYA2k2N744MNiCD1UE+tJyllUhSblK48bn+v1oZHCM0nYQ2NqUkvS
        OrUZ/wK69Dzu4IvrN4vs9Nes8vbwPa/ddZEzGR0cQMt0JBkhk9kU/qwqUseP1QRJ
        5I1jR4g8aYPL/ke9K35PxZWuDp3U0UPAZ3PjFAh+5T+fc7gzCs9dPzSHloruU+gl
        FQIDAQAB
        -----END CERTIFICATE-----",
        "device_certificate": "-----BEGIN CERTIFICATE-----
        MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAryQICCl6NZ5gDKrnSztO
        3Hy8PEUcuyvg/ikC+VcIo2SFFSf18a3IMYldIugqqqZCs4/4uVW3sbdLs/6PfgdX
        7O9D22ZiFWHPYA2k2N744MNiCD1UE+tJyllUhSblK48bn+v1oZHCM0nYQ2NqUkvS
        3Hy8PEUcuyvg/ikC+VcIo2SFFSf18a3IMYldIugqqqZCs4/4uVW3sbdLs/6PfgdX
        7O9D22ZiFWHPYA2k2N744MNiCD1UE+tJyllUhSblK48bn+v1oZHCM0nYQ2NqUkvS
        3Hy8PEUcuyvg/ikC+VcIo2SFFSf18a3IMYldIugqqqZCs4/4uVW3sbdLs/6PfgdX
        7O9D22ZiFWHPYA2k2N744MNiCD1UE+tJyllUhSblK48bn+v1oZHCM0nYQ2NqUkvS
        OrUZ/wK69Dzu4IvrN4vs9Nes8vbwPa/ddZEzGR0cQMt0JBkhk9kU/qwqUseP1QRJ
        5I1jR4g8aYPL/ke9K35PxZWuDp3U0UPAZ3PjFAh+5T+fc7gzCs9dPzSHloruU+gl
        FQIDAQAB
        -----END CERTIFICATE-----"
    }
```

#### Create an access key

##### Creating a new access key

To create your first access key, log in to the mbed Connector webapp. After that, new access keys can be created programmatically. 

To create a new access key:

```
POST /api/api_tokens
```

**Body**

The body, if present, needs to be a JSON object that contains the wanted display name in field 'name'. If no body is given, a random alphanumeric string will be generated for name.


|Name|Type|Description|
|---|---|---|
|name|`string`| Optional. The display name for the new key.|

**Response**

Response|Description
---|---
201|Created, successful response with new access key.
401|Unauthorized, provided access key is not valid for domain.
400|Bad Request, error parsing JSON payload.

Acceptable content-types:

- application/json

**Example**

```
    POST /api/api_tokens
    Accept: application/json
    Content-Type: application/json

    {
        "name": "Access key name #1"
    }

    HTTP/1.1 201 Created
    Content-Type: application/json

    {
        "access_key": "pkDJumPX65wZXrqjyQrd42skwRomzolglEzgrjrntSC8g11UmJgL2Oq5NhQ7MFpmb6EPKkxEKV67ZUxc1A0kygAGHz5stChxMFY8",
        "name" : "Access key name #1"
    }
```

**Example**

```
    POST /api/api_tokens
    Accept: application/json

    HTTP/1.1 201 Created
    Content-Type: application/json

    {
        "access_key": "F2x59HUmziqRI6w3UUj8PmYOMLdMAkpUsHCEMUVym8stcC4TQJa8sjOm2pqf5PGnzobXTW5h7NM7MoxCY0wZcwiWKr5mtxAEkjkw",
        "name" : "lSUXJA9jTP"
    }
```
