# Jayson

Jayson is a JSON-RPC 2.0 compliant (at the time of writing [version
2011-12-11][jsonrpc-spec]) server and client written in JavaScript that wants to
be as clean and simple to use as possible. 

[jsonrpc-spec]: http://jsonrpc.org/spec.html 
[node.js]: http://nodejs.org/

## Rationale / Why

Current implementations of JSON-RPC in node.js are either outdated, unmaintained or
not simple enough to use. The appropriate course of action when other libraries
do not do it properly is of course to reinvent the wheel and write an implementation
that does it right yourself.

## Example

A simple example of a JSON-RPC 2.0 server using HTTP:

```javascript
// server.js
var jayson = require('jayson');

// create a server
var server = jayson.server({
  add: function(a, b, callback) {
    callback(null, a + b);
  }
});

// let a http server listen to localhost:3000
server.http().listen(3000);
```

And a client invoking "add" on the above server:

```javascript
// client.js
var jayson = require('jayson');

// create a client
var client = jayson.client.http({
  port: 3000,
  hostname: 'localhost'
});

// invoke "add"
client.request('add', [1, 1], function(err, error, response) {
  if(err) throw err;
  console.log(response); // 2!
});
```


## Features

## Installation

Install the latest version of _jayson_ from [npm](https://github.com/isaacs/npm) by
executing `npm install jayson` in your shell. Do a global install with `npm install --global
jayson` if you want to use the jayson command line client from anywhere on your system.

## Requirements

Jayson does not have any dependencies that cannot be resolved with a simple `npm
install`.

- node.js (>= 0.5.0 < 0.7.0)

### Running tests

- Change directory to the repository root
- Install the testing framework
  ([mocha](https://github.com/visionmedia/mocha) together with
  [should](https://github.com/visionmedia/should.js)) by executing `npm install
  --dev`
- Run the tests with `make test` or `npm test`

## Usage

### Client

The client classes are all available by using `require('jayson').client`.

#### Client interfaces

* `client.http` Client that enables interfacing with HTTP protocols. See [http.request](http://nodejs.org/docs/v0.6.19/api/http.html#http_http_request_options_callback) for supported options.
* `client.jquery`

#### Notification requests

#### Batch requests

### Server

The server classes are all available with `require('jayson').server`. The server sports several interfaces that can be accessed as properties of an instance of `require('jayson').server` (long for `JaysonServer`). 

#### Server interfaces

* `server` - Base interface for a server that supports receiving JSON-RPC 2.0 requests.
* `server.http` - HTTP server that inherits from [http.Server](http://nodejs.org/docs/latest/api/http.html#http_class_http_server).
* `server.https` - HTTPS server that inherits from [https.Server](http://nodejs.org/docs/latest/api/https.html#https_class_https_server).
* `server.middleware` - Method that returns a [Connect](http://www.senchalabs.org/connect/)/[Express](http://expressjs.com/) compatible middleware.

##### Using many interfaces at the same time

A Jayson server can use many interfaces at the same time.

Example of a server that listens has both a `http` and a `https` interface:

```javascript
var jayson = require('jayson');

 var server = jayson.server({
   add: function(a, b, callback) { return callback(null, a + b); }
 });

 // http is now an instance of require('http').Server
 var http = server.http();

 // https is now an instance of require('https').Server
 var https = server.https({
   cert: require('fs').readFileSync('cert.pem'),
   key require('fs').readFileSync('key.pem')
 });

 http.listen(80); // let http listen to localhost:3000

 https.listen(443); // let https listen to localhost:443
```

#### Server methods

##### Server([methods[, [options]]])

Constructor for a server. Will return an instance of `Server` even if not
invoked with `new`.

* `methods` (Object) Object of name and method pairs (name -> func)
* `options` (Object) Object of settings that will propagate to all interfaces

##### Server.prototype.method(name[, definition])

Adds a method to the server.

* `name` (String) Name of method
* `definition` (Function|JaysonClient) Function definition that can be either a regular JavaScript function that must take a callback as the last argument, or an instance of `jayson.Client` for relay functionality.

##### Server.prototype.hasMethod(name)

Checks if a method with `name` exists on the server. Returns a `Boolean`.

* `name` (String) Name of method

##### Server.prototype.removeMethod(name)

Removes a method from the server. Returns void.

* `name` (String) Name of method

##### Server.prototype.error([code[, message[, data]]])

Returns a JSON-RPC error object. Can be used to generate a custom error inside a method or to intentionally return [one of the official JSON-RPC 2.0 errors](http://www.jsonrpc.org/specification#error_object). The error returned by this method can be used as the first argument to a RPC method callback.

* `code` (Number) Optional integer code
* `message` (String) Optional string description
* `data` (Object) Optional object with additional data

##### Server.prototype.call(request[, callback])

Calls a method on the server instance. Normally not used directly but can be used to pass raw JSON-RPC requests.

* `request` (Object|Array) JSON-RPC 2.0 request object or an array of batch requests
* `callback` (Function) Optional function that will be called on request completion

#### Using the server as a relay

Passing an instance of a client as a method (to the server) allows the server to relay incoming RPC
calls to the corresponding method on the server the client is pointing to. This
might be used to delegate computationally expensive functions into a separate
fork/thread/server or to abstract a cluster of servers behind a common
interface. Example:

```javascript
// server_public.js
var jayson = require('jayson');

// create a server where "add" will relay a localhost-only server
var server = jayson.server({
  add: jayson.client.http({
    hostname: 'localhost',
    port: 3001
  })
});

// let the server listen to *:3000
server.http().listen(3000, '0.0.0.0');
```

```javascript
// server_private.js
var jayson = require('jayson');

var server = jayson.server({
  add: function(a, b, callback) {
    callback(null, a + b);
  }
});

// let the private server listen to localhost:3001
server.http().listen(3001);
```

### Revivers and Replacers

JSON is a great data format, but it lacks support for representing types other than those defined in the (JSON specification)[http://www.json.org/] Fortunately the JSON methods in JavaScript (`JSON.parse` and
`JSON.stringify`) provides options for custom serialization/deserialization
routines. Jayson allows you to pass your own routines as options to both clients
and servers.

Simple example transferring the state of an object between a client and a server:

Shared code between the server and the client:

```javascript
// shared.js
var Counter = exports.Counter = function(value) {
  this.count = value || 0;
};

Counter.prototype.increment = function() {
  this.count += 1;
};

exports.replacer = function(key, value) {
  if(value instanceof Counter) {
    return {$class: 'counter', $props: {count: value.count}};
  }
  return value;
};

exports.reviver = function(key, value) {
  if(v && v.$class === 'counter') {
    var obj = new Counter;
    for(var prop in v.$props) obj[prop] = v.$props[prop];
    return obj;
  }
  return value;
};
```

The server:

```javascript
// server.js
var jayson = require('jayson');
var shared = require('./shared');

// Set the reviver/replacer options
var options = {
  reviver: shared.reviver,
  replacer: shared.replacer
};

// create a server
var server = jayson.server({
  increment: function(counter, callback) {
    counter.increment();
    callback(null, instance);
  }
}, options);

// let the server listen to for http connections on localhost:3000
server.http().listen(3000);
```

And a client invoking "increment" on the above server:

```javascript
// client.js
var jayson = require('jayson');
var shared = require('./shared');

// create a client with the shared reviver and replacer
var client = jayson.client.http({
  port: 3000,
  hostname: 'localhost',
  reviver: shared.reviver,
  replacer: shared.replacer
});

// the object
var instance = new shared.Counter(2);

// invoke "increment"
client.request('increment', [instance], function(err, error, response) {
  if(err) throw err;
  console.log(response instanceof shared.Counter); // true
  console.log(response.count); // 3!
});
```

Instead of using a replacer, it is possible to define a `toJSON` method for any JavaScript object. Unfortunately there is no corresponding method to define for reviving objects, so the _reviver_ always has to be set up manually.

### Contributing

Highlighting [issues](https://github.com/tedeh/jayson/issues) or submitting pull
requests on [Github](https://github.com/tedeh/jayson) is most welcome.
