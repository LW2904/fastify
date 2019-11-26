<h1 align="center">Fastify</h1>

## Routes
You have two ways to declare a route with Fastify, the shorthand method and the full declaration.
<a name="full-declaration"></a>
### Full declaration
```js
fastify.route(options)
```
* `method`: currently supports `'DELETE'`, `'GET'`, `'HEAD'`, `'PATCH'`, `'POST'`, `'PUT'` and `'OPTIONS'`. Can also be an array of methods.

* `url`: the path to match against the requested url (alias: `path`).
* `schema`: an object containing the schemas for the request and response which needs to be in [JSON Schema](http://json-schema.org/) format. Check out the [Validation and Serialization](https://github.com/fastify/fastify/blob/master/docs/Validation-and-Serialization.md) docs for more info.

  * `body`: validates the body of the request if it is a POST or a
    PUT.
  * `querystring` or `query`: validates the querystring. This can be a complete JSON
  Schema object, with the property `type` of `object` and `properties` object of parameters, or
  simply the values of what would be contained in the `properties` object as shown below.
  * `params`: validates the params.
  * `response`: filter and generate a schema for the response, setting a
    schema allows us to have 10-20% more throughput.
* `attachValidation`: if there is a schema validation error, attach a `validationError` instead of sending the error to the error handler.
* `onRequest(request, reply, done)`: a [function](https://github.com/fastify/fastify/blob/master/docs/Hooks.md#route-hooks) that is called as soon as a request is received. Can also be an array of functions.
* `preValidation(request, reply, done)`: a [function](https://github.com/fastify/fastify/blob/master/docs/Hooks.md#route-hooks) that is called after the shared `preValidation` hooks, useful if you need to perform authentication at route level, for example. Can also be an array of functions.
* `preHandler(request, reply, done)`: a [function](https://github.com/fastify/fastify/blob/master/docs/Hooks.md#route-hooks) that is called just before the request handler. Can also be an array of functions.
* `preSerialization(request, reply, payload, done)`: a [function](https://github.com/fastify/fastify/blob/master/docs/Hooks.md#route-hooks) that is called just before the serialization. Can also be an array of functions.
* `handler(request, reply)`: the function that will handle this request.
* `schemaCompiler(schema)`: the function that builds the schema for the validation. See the [Validation and Serialization](https://github.com/fastify/fastify/blob/master/docs/Validation-and-Serialization.md#schema-compiler) docs for more information.
* `bodyLimit`: prevents the default JSON body parser from parsing request bodies larger than this number of bytes. Must be an integer. You may also set this option globally when first creating the Fastify instance with `fastify(options)`. Defaults to `1048576` (1 MiB).
* `logLevel`: set the log level for this route. Check out the [Custom Log Level](#custom-log-level) section for more information.
* `config`: an object used to store any custom configuration.
* `version`: a [semver](http://semver.org/) compatible string that defines the version of the endpoint. See the [Version](#version) subsection.
* `prefixTrailingSlash`: string used to determine how to handle the `/` route when using [route prefixing](#route-prefixing).
  * `both` (default): Will register both `/prefix` and `/prefix/`.
  * `slash`: Will register only `/prefix/`.
  * `no-slash`: Will register only `/prefix`.

  `request` is defined in [Request](https://github.com/fastify/fastify/blob/master/docs/Request.md).

  `reply` is defined in [Reply](https://github.com/fastify/fastify/blob/master/docs/Reply.md).

```js
fastify.route({
  method: 'GET',
  url: '/',
  schema: {
    querystring: {
      name: { type: 'string' },
      excitement: { type: 'integer' }
    },
    response: {
      200: {
        type: 'object',
        properties: {
          hello: { type: 'string' }
        }
      }
    }
  },
  handler: function (request, reply) {
    reply.send({ hello: 'world' })
  }
})
```

<a name="shorthand-declaration"></a>
### Shorthand declaration
The above route declaration is more *Hapi*-like, but if you prefer an *Express/Restify* approach, we support it as well:<br>
`fastify.get(path, [options], handler)`<br>
`fastify.head(path, [options], handler)`<br>
`fastify.post(path, [options], handler)`<br>
`fastify.put(path, [options], handler)`<br>
`fastify.delete(path, [options], handler)`<br>
`fastify.options(path, [options], handler)`<br>
`fastify.patch(path, [options], handler)`

```js
const opts = {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          hello: { type: 'string' }
        }
      }
    }
  }
}
fastify.get('/', opts, (request, reply) => {
  reply.send({ hello: 'world' })
})
```

`fastify.all(path, [options], handler)` will add the same handler to all supported methods.

The handler may also be supplied via the `options` object:
```js
const opts = {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          hello: { type: 'string' }
        }
      }
    }
  },
  handler (request, reply) {
    reply.send({ hello: 'world' })
  }
}
fastify.get('/', opts)
```

> Note that if the handler is specified in both the `options` and as the third parameter to the shortcut method, an error is thrown.

<a name="url-building"></a>
### Url building
Fastify supports both static and dynamic urls.<br>
To register a **parametric** path, use a *colon* before the parameter name. For **wildcards** use an *asterisk*.
*Remember that static routes are always checked before routes using parametric or wildcard paths.*

```js
// parametric
fastify.get('/example/:userId', (request, reply) => {}))
fastify.get('/example/:userId/:secretToken', (request, reply) => {}))

// wildcard
fastify.get('/example/*', (request, reply) => {}))
```

Regular expression routes are supported as well. Do note that RegExp routes are very expensive in terms of performance!
```js
// parametric with regexp
fastify.get('/example/:file(^\\d+).png', (request, reply) => {}))
```

It's possible to define more than one parameter within the same pair of slashes (`/`). Note that dashes have to be used to seperate parameters, in this case.
```js
fastify.get('/example/near/:lat-:lng/radius/:r', (request, reply) => {}))
```

Finally, it's possible to have multiple parameters with RegExp. In this case it's possible to use any character that is not matched by the regular expression as separator.
```js
fastify.get('/example/at/:hour(^\\d{2})h:minute(^\\d{2})m', (request, reply) => {}))
```

Having a route with multiple parameters may negatively affect the performance of your app. Always prefer a single parameter approach, especially on routes which are on the hot path of your application.

If you are interested in how we handle the routing, take a look at [find-my-way](https://github.com/delvedor/find-my-way).

<a name="async-await"></a>
### Async Await
Are you an `async/await` user? We have you covered!
```js
fastify.get('/', options, async function (request, reply) {
  const data = await getData()
  const processed = await processData(data)
  return processed
})
```
**Warning:** You can't return `undefined`. For more details see [Promise resolution](#promise-resolution).

As you can see, `reply.send` is not used to send the data back to the user. You just need to return the body and you are done!

If needed you can also still use `reply.send` however. The following example is equivalent to the one shown above.
```js
fastify.get('/', options, async function (request, reply) {
  const data = await getData()
  const processed = await processData(data)
  reply.send(processed)
})
```
**Warning:**
* If you use `return` and `reply.send` at the same time, the first one that happens takes precedence, the second value will be discarded, a *warn* log will also be emitted because you tried to send a response twice.

<a name="promise-resolution"></a>
### Promise resolution

If your handler is an `async` function or returns a promise, you should be aware of special behaviour which is necessary to support both the callback and promise control-flow. If the handler's promise is resolved with `undefined`, it will be ignored causing the request to hang and an *error* log to be emitted.

1. If you want to use `async/await` or promises:
    - **Don't** use `reply.send`.
    - **Don't** return `undefined`.
2. If you want to use `async/await` or promises but respond to a value with `reply.send`:
    - **Don't** `return` any value.
    - **Don't** forget to call `reply.send`.

In this way, we can support both `callback-style` and `async-await`, with very little trade-offs. In spite of so much freedom we highly recommend to go with only one of the two styles since errors should always be handled in a consistent way within your application.

**Notice**: Every async function returns a promise by itself.

<a name="route-prefixing"></a>
### Route Prefixing
Sometimes you need to maintain two or more different versions of the same api, a classic approach is to prefix all the routes with the api version number, `/v1/user` for example.
Fastify offers you a fast and smart way to create different version of the same api without changing all the route names by hand, *route prefixing*.

```js
// server.js
const fastify = require('fastify')()

fastify.register(require('./routes/v1/users'), { prefix: '/v1' })
fastify.register(require('./routes/v2/users'), { prefix: '/v2' })

fastify.listen(3000)
```
```js
// routes/v1/users.js
module.exports = function (fastify, opts, done) {
  fastify.get('/user', handler_v1)
  done()
}
```
```js
// routes/v2/users.js
module.exports = function (fastify, opts, done) {
  fastify.get('/user', handler_v2)
  done()
}
```
Fastify will not complain that you are using the same name for two different routes, because at compilation time it will handle the prefix automatically *(this also means that the performance will not be affected at all!)*.

Now your clients will have access to the following routes:
- `/v1/user`
- `/v2/user`

You can do this as many times as you want, nested `register`s and route parameters are supported as well.

**Warning:** If you use [`fastify-plugin`](https://github.com/fastify/fastify-plugin), this option won't work.

#### Handling of `/` route inside prefixed plugins

The `/` route has different behavior depending on whether or not the prefix ends with `/`. As an example, consider the prefix `/something/`, where adding a `/` route will only match `/something/`. With the the prefix `/something`, adding a `/` route will match both `/something` 
and `/something/`.

See the `prefixTrailingSlash` route option above to change this behaviour.

<a name="custom-log-level"></a>
### Custom Log Level
It could happen that you need different log levels for routes on a case by case basis. With Fastify, you can achieve this in a very straightforward way by passing the `logLevel` option to a given plugin or route, with the [level](https://github.com/pinojs/pino/blob/master/docs/api.md#level-string) that you need.<br/>

Be aware that if you set the `logLevel` at plugin level, the [`setNotFoundHandler`](https://github.com/fastify/fastify/blob/master/docs/Server.md#setnotfoundhandler) and [`setErrorHandler`](https://github.com/fastify/fastify/blob/master/docs/Server.md#seterrorhandler) will also be affected.

```js
// server.js
const fastify = require('fastify')({ logger: true })

fastify.register(require('./routes/user'), { logLevel: 'warn' })
fastify.register(require('./routes/events'), { logLevel: 'debug' })

fastify.listen(3000)
```
Or you can directly pass it to a route:
```js
fastify.get('/', { logLevel: 'warn' }, (request, reply) => {
  reply.send({ hello: 'world' })
})
```

*Note that the custom log level is applied only to the routes, and not to the global Fastify Logger, accessible with `fastify.log`*

<a name="routes-config"></a>
### Config
When registering a new handler, you can pass a configuration object to it and retrieve it in the handler.

```js
// server.js
const fastify = require('fastify')()

function handler (req, reply) {
  reply.send(reply.context.config.output)
}

fastify.get('/en', { config: { output: 'hello world!' } }, handler)
fastify.get('/it', { config: { output: 'ciao mondo!' } }, handler)

fastify.listen(3000)
```

<a name="version"></a>
### Version
#### Default
If needed you can provide a version option, which will allow you to declare multiple versions of the same route. The versioning should follow the [semver](http://semver.org/) specification.<br/>
Fastify will automatically detect the `Accept-Version` header and route the request accordingly (advanced ranges and pre-releases currently are not supported).<br/>
*Be aware that using this feature will cause a degradation in the overall performance of the router.*
```js
fastify.route({
  method: 'GET',
  url: '/',
  version: '1.2.0',
  handler: function (request, reply) {
    reply.send({ hello: 'world' })
  }
})

fastify.inject({
  method: 'GET',
  url: '/',
  headers: {
    'Accept-Version': '1.x' // it could also be '1.2.0' or '1.2.x'
  }
}, (err, res) => {
  // { hello: 'world' }
})
```
If you declare multiple versions with the same major or minor number, Fastify will always choose the highest version compatible with the `Accept-Version` header value.<br/>
If the request does not have the `Accept-Version` header, a 404 error will be returned.
#### Custom
It's possible to define a custom versioning logic. This can be done through the [`versioning`](https://github.com/fastify/fastify/blob/master/docs/Server.md#versioning) configuration, when creating a fastify server instance.
