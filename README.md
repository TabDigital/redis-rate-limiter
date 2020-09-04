# redis-rate-limiter

[![NPM](http://img.shields.io/npm/v/redis-rate-limiter.svg?style=flat)](https://npmjs.org/package/redis-rate-limiter)
[![License](http://img.shields.io/npm/l/redis-rate-limiter.svg?style=flat)](https://github.com/Tabcorp/redis-rate-limiter)

[![Build Status](http://img.shields.io/travis/Tabcorp/redis-rate-limiter.svg?style=flat)](http://travis-ci.org/Tabcorp/redis-rate-limiter)
[![Dependencies](http://img.shields.io/david/Tabcorp/redis-rate-limiter.svg?style=flat)](https://david-dm.org/Tabcorp/redis-rate-limiter)
[![Dev dependencies](http://img.shields.io/david/dev/Tabcorp/redis-rate-limiter.svg?style=flat)](https://david-dm.org/Tabcorp/redis-rate-limiter)
[![Known Vulnerabilities](https://snyk.io/package/npm/redis-rate-limiter/badge.svg)](https://snyk.io/package/npm/redis-rate-limiter)

Rate-limit any operation, backed by Redis.

- Inspired by [ratelimiter](https://www.npmjs.org/package/ratelimiter)
- But uses a fixed-window algorithm
- Great performance (>10000 checks/sec on local redis)
- No race conditions

Very easy to plug into `Express` or `Restify` to rate limit your `Node.js` API.

## Usage

Step 1: create a Redis connection

```js
var redis = require('redis');
var client = redis.createClient(6379, 'localhost', {enable_offline_queue: false});
```

Step 2: create your rate limiter

```js
var rateLimiter = require('redis-rate-limiter');
var limit = rateLimiter.create({
  redis: client,
  key: function(x) { return x.id },
  rate: '100/minute'
});
```

And go

```js
limit(request, function(err, rate) {
  if (err) {
    console.warn('Rate limiting not available');
  } else {
    console.log('Rate window: '  + rate.window);  // 60
    console.log('Rate limit: '   + rate.limit);   // 100
    console.log('Rate current: ' + rate.current); // 74
    if (rate.over) {
      console.error('Over the limit!');
    }
  }
});
```

## Options

### `redis`

A pre-created Redis client.
Make sure offline queueing is disabled.

```js
var client = redis.createClient(6379, 'localhost', {
  enable_offline_queue: false
});
```

### `key`

The key is how requests are grouped for rate-limiting.
Typically, this would be a user ID, a type of operation.

You can also specify any custom function:

```js
// rate-limit each user separately
key: function(x) { return x.user.id; }

// rate limit per user and operation type
key: function(x) { return x.user.id + ':' + x.operation; }

// rate limit everyone in the same bucket
key: function(x) { return 'single-bucket'; }
```

You can also use the built-in `ip` shorthand, which gets the remote address from an HTTP request.

```js
key: 'ip'
```

### `window`

This is the duration over which rate-limiting is applied, in seconds.

```js
// rate limit per minute
window: 60

// rate limit per hour
window: 3600
```

Note that this is **not a rolling window**.
If you specify `10 requests / minute`, a user would be able
to execute 10 requests at `00:59` and another 10 at `01:01`.
Then they won't be able to make another request until `02:00`.


### `limit`

This is the total number of requests a unique `key` can make during the `window`.

```js
limit: 100
```

### `rate`

Rate is a shorthand notation to combine `limit` and `window`.

```js
rate: '10/second'
rate: '100/minute'
rate: '1000/hour'
```

Or the even shorter

```js
rate: '10/s'
rate: '100/m'
rate: '100/h'
```

*Note:* the rate is parsed ahead of time, so this notation doesn't affect performance.

### `deleteImmediatelyIfRaceCondition`
If a race condition occurs, redis-rate-limiter sets a expire date to corresponding entry.
You can set this option to true, if you want to delete this corrupted entry immediately after occurring a possible
race condition.

Default is `false`

```js
deleteImmediatelyIfRaceCondition: true
```

### `onPossibleRaceCondition`
A callback function called after occurring a possible race condition.

Definition: `(key: string): void`

```js
onPossibleRaceCondition: function(key) {
    console.log(`Race condition detected for ${key}`);
}
```

## HTTP middleware

This package  contains a pre-built middleware,
which takes the same options


```js
var rateLimiter = require('redis-rate-limiter');

var middleware = rateLimiter.middleware({
  redis: client,
  key: 'ip',
  rate: '100/minute'
});

server.use(middleware);
```

It rejects any rate-limited requests with a status code of `HTTP 429`,
and an empty body.

*Note:* if you want to rate limit several routes individually, don't forget to use the route name as part of the `key`, for example using Restify:

```js
function ipAndRoute(req) {
  return req.connection.remoteAddress + ':' + req.route.name;
}

server.get(
  {name: 'routeA', path: '/a'},
  rateLimiter.middleware({redis: client, key: ipAndRoute, rate: '10/minute'}),
  controllerA
);

server.get(
  {name: 'routeB', path: '/b'},
  rateLimiter.middleware({redis: client, key: ipAndRoute, rate: '20/minute'}),
  controllerB
);
```
