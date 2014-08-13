
## Gofer - API

The module exports one function, `buildGofer`.
In addition it exposes `gofer/hub` which exports [`Hub`](#hub).

#### buildGofer(serviceName: String, serviceVersion: String) -> Gofer

Create a new gofer class for a given service.
`serviceName` is used in multiple places:

* It is the key under which instances of the class will be looking for service-level [configuration](#configuration)
* It is injected into the fetch options, see [events and logging](#events-and-logging)
* It is used to build the `User-Agent` header

### Gofer

#### new Gofer(config = {}, hub = new Hub()) -> gofer

* `config`: A config object as described in [configuration](#configuration)
* `hub`: An instance of [Hub](#hub)

#### Static methods

##### Gofer.addOptionMapper(mapFn)

Add a new option mapper to *all* instances that are created afterwards. This
can also be called on an instance which doesn't have a global effect.

* `mapFn`: An option mapper, see [option mappers](#option-mappers)

##### Gofer.clearOptionMappers()

Clear the option mapper chain for all instances that are created afterwards.
It can also be called on an instance which does not have a global effect.

##### Gofer.registerEndpoints(endpointMap)

Registers "endpoints". Endpoints are convenience methods for easier
construction of API calls and can also improve logging/tracing. The following
conditions are to be met by `endpointMap`:

1. It maps a string identifier that is a valid property name to a function
2. The function takes one argument which is `request`
3. `request` works like `gofer.request` only that it's aware of [endpoint defaults](#configuration)

Whatever the function returns will be available as a property on instances of the Gofer class.
Reasonable variants are a function or a nested objects with functions.

```js
MyService.registerEndpoints({
  simple: function(request) {
    return function(cb) {
      return request('/some-path', cb);
    };
  },
  complex: function(request) {
    return {
      foo: function(qs, cb) {
        return request('/foo', { qs: qs }, cb);
      },
      bar: function(entity, cb) {
        return request('/bar', { json: entity, method: 'PUT' }, cb);
      }
    }
  }
});
var my = new MyService();
my.simple(function(err, body) {});
my.complex.foo({ limit: 1 }, function(err, body) {});
my.complex.bar({ name: 'Jordan', friends: 231 }, function(err, body) {});
```

#### Instance methods

##### gofer.clone()

Creates a new instance with the exact same settings and referring to the same `hub`.

##### gofer.with(overrideConfig)

Returns a copy with `overrideConfig` merged into both the endpoint- and the
service-level defaults.
Useful if you know that you'll need custom timeouts for this one call or you want to add an accessToken.

##### gofer.request(uri: String, options, cb)

* `uri`: A convenient way to specify `options.uri` directly
* `options`: Anything that is valid [configuration](#configuration)
* `cb`: A callback function that receives the following arguments:
  - `error`: An instance of `Error`
  - `body`: The, in most cases parsed, response body
  - `meta`: Stats about the request
  - `response`: The response object with headers and statusCode

The return value is the same as the one of `request`.

If an HTTP status code outside of the accepted range is returned,
the error object will have the following properties:

* `type`: 'api_response_error'
* `httpHeaders`: The headers of the response
* `body`: The, in most cases parsed, response body
* `statusCode`: The http status code
* `minStatusCode`: The lower bound of accepted status codes
* `maxStatusCode`: The upper bound of accepted status codes

The accepted range of status codes is part of the [configuration](#configuration).
It defaults to accepting 2xx codes only.

If there's an error that prevents any response from being returned,
you can look for `code` to find out what happened.
Possible values include:

* `ECONNECTTIMEDOUT`: It took longer than `options.connectTimeout` allowed to establish a connection
* `ETIMEDOUT`: Request took longer than `options.timeout` allowed
* `ESOCKETTIMEDOUT`: Same as `ETIMEDOUT` but signifies that headers were received
* `EPIPE`: Writing to the request failed
* `ECONNREFUSED`: The remote host refused the connection, e.g. because nothing was listening on the port
* `ENOTFOUND`: The hostname failed to resolve

##### gofer.applyBaseUrl(baseUrl: String, options)

Takes `options.uri`, discards everything but the `pathname` and appends it to the specified `baseUrl`.

```js
applyBaseUrl('http://api.example.com/v2', { uri: '/foo' })
  === 'http://api.example.com/v2/foo';
applyBaseUrl('http://api.example.com/v2', { uri: '/foo?x=y' })
  === 'http://api.example.com/v2/foo?x=y';
applyBaseUrl('http://api.example.com/v2', { uri: { pathname: '/zapp' } })
  === 'http://api.example.com/v2/zapp';
```

##### gofer\[httpVerb\](uri: String, options, cb)

Convenience methods to make requests with the specified http method.
Just lowercase the http verb and you've got the method name,
only exception is `gofer.del` to avoid collision with the `delete` keyword.

### Option mappers

All service-specific behavior is implemented using option mappers.
Whenever an request is made, either via an endpoint or directly via `gofer.request`,
the options go through the following steps:

1. The endpoint defaults are applied if the request was made through an endpoint
3. `options.serviceName` and `options.serviceVersion` is added
4. `options.methodName` and `options.endpointName` is added. The former defaults to the http verb but can be set to a custom value (e.g. `addFriend`). The latter is only set if the request was made through an endpoint method
5. The service-specific and global defaults are applied
6. For every registered option mapper `m` the `options` are set to `m(options)`
7. A `User-Agent` header is added if not present already
8. `null` and `undefined` values are removed from `qs`. If you want to pass empty values, you should use an empty string

Step 6 implies that every option mapper is a function that takes one argument `options` and returns transformed options.
Inside of the mapper `this` refers to the `gofer` instance.
The example contains an option mapper that handles access tokens and a default base url.

By default every `gofer` class starts of with one option mapper.
It just calls `gofer.applyBaseUrl` if `options.baseUrl` is passed in.

#### Options

In addition to the options mentioned in the [request docs](https://github.com/mikeal/request#requestoptions-callback), `gofer` offers the following options:

* `connectTimeout`: How long to wait until a connection is established
* `baseUrl`: See `applyBaseUrl` above
* `parseJSON`: The `json` option offered by request itself will silently ignore when parsing the body fails. This option on the other hand will forward parse errors. It defaults to true if the response has a json content-type and is non-empty
* `minStatusCode`: The lowest http status code that is acceptable. Everything below will be treated as an error. Defaults to `200`
* `maxStatusCode`: The highest http status code that is acceptable. Everything above will be treated as an error. Defaults to `299`
* `requestId`: Useful to track request through across services. It will added as an `X-Request-ID` header. See [events and logging](#events-and-logging) below
* `serviceName`: The name of the service that is talked to, e.g. "github". Used in the user-agent
* `serviceVersion`: By convention the client version. Used in the user-agent

In addition the following options are added that are useful for instrumentation but do not affect the actual HTTP request:

* `endpointName`: The name of the "endpoint" or part of the API, e.g. "repos"
* `methodName`: Either just an http verb or something more specific like "repoByName". Defaults to the http verb (`options.method`)

#### Configuration

All parts of the configuration end up as options.
There are three levels of configuration:

* `globalDefaults`: Used for calls to any service
* `[serviceName].*`: Only used for calls to one specific service
* `[serviceName].endpointDefaults[endpointName].*`: Only used for calls using a specific endpoint

```js
var buildGofer = require('gofer');

var config = {
  "globalDefaults": { "timeout": 100, "connectTimeout": 55 },
  "a": { "timeout": 1001 },
  "b": {
    "timeout": 99,
    "connectTimeout": 70,
    "endpointDefaults": {
      "x": { "timeout": 23 }
    }
  }
};

var GoferA = buildGofer('a'), GoferB = buildGofer('b');
GoferB.registerEndpoints({
  x: function(request) {
    return function(cb) { return request('/something', cb); }
  }
});

var a = new GoferA(config), b = new GoferB(config);
a.request('/something'); // will use timeout: 1001, connectTimeout: 55
b.request('/something'); // will use timeout: 99, connectTimeout: 70
b.x(); // will use timeout: 23, connectTimeout: 70
```

### Hub

Every `gofer` instance has a reference to a "hub".
The hub is used to make all calls to request and exposes a number of useful events.
The following snippet shows how to share a hub across multiple gofer instances:

```js
var buildGofer = require('gofer');
var GoferA = buildGofer('a'); // client for service "a"
var GoferB = buildGofer('b'); // client for service "b"

var hub = require('gofer/hub')(); // create a new hub
var goferA = new GoferA({ /* config */ }, hub);
var goferB = new GoferB({ /* config */ }, hub);

hub.on('success', function() {}); // this will fire for every successful
                                  // request using either goferA or goferB
```

#### Events and Logging

There are a couple of things `gofer` does that are opinionated but may make your life easier.

1.  It assumes you are using `x-request-id` headers.
    These can be very useful when tracing a request through multiple levels in the stack.
    Heroku has a [nice description](https://devcenter.heroku.com/articles/http-request-id).
2.  It uses unique `x-fetch-id` headers for each http request.
3.  All timings are reported in seconds with microtime precision.


##### start

A service call is attempted.
Event data:

```js
{ fetchStart: Float, // time in seconds
  requestOptions: options, // options passed into request
  requestId: UUID, // id of the overall transaction
  fetchId: UUID } // id of this specific http call
```


##### socketQueueing

Waiting for a socket. See [`http.globalAgent.maxSockets`](http://nodejs.org/api/http.html#http_agent_maxsockets).
Event data:

```js
{ maxSockets: http.globalAgent.maxSockets,
  queueReport: Array[String] } // each entry contains "<host>: <queueLength>"
```


##### connect

Connected to the remote host, transfer may start.
Event contains the data from `start` plus:

```js
{ connectDuration: Float } // time in seconds to establish a connection
```


##### success

All went well.
Event data:

```js
{ statusCode: Integer, // the http status code
  uri: String,
  method: String, // uppercase http verb, PUT/GET/...
  connectDuration: Float,
  fetchDuration: Float,
  requestId: UUID,
  fetchId: UUID }
```


##### fetchError

A transport error occured (e.g. timeouts).
Event contains the data from `success` plus:

```js
{ statusCode: String, // the error code (e.g. ETIMEDOUT)
  syscall: String, // the syscall that failed (e.g. getaddrinfo)
  error: error } // the raw error object
```


##### failure

An invalid status code was returned.
Event contains the data from `success`.