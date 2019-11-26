<h1 align="center">Fastify</h1>

## Plugins
Fastify allows the user to extend its functionality with plugins. A plugin can be a set of routes, a server [decorator](https://github.com/fastify/fastify/blob/master/docs/Decorators.md) or anything else, really. The `register` API is provided to use plugins.

By default, `register` creates a *new scope*. This means that if you make some changes to the Fastify instance (via `decorate`), this change will only be reflected in the children of the current context. This feature allows us to achieve plugin *encapsulation* and *inheritance*, in this way we create a *direct acyclic graph* (DAG) and will not have issues caused by cross dependencies.

Assuming you read the [Getting Started](https://github.com/fastify/fastify/blob/master/docs/Getting-Started.md#register) guide, you probably already noticed how the usage of this API is very straightforward. 

```
fastify.register(plugin, [options])
```

<a name="plugin-options"></a>
### Plugin Options
The optional `options` parameter for `fastify.register` supports a predefined set of options that Fastify itself will use, except when the plugin has been wrapped with [fastify-plugin](https://github.com/fastify/fastify-plugin). This options object will also be passed to the plugin upon invocation, regardless of whether or not the plugin has been wrapped. The currently supported list of Fastify specific options is:

+ [`logLevel`](https://github.com/fastify/fastify/blob/master/docs/Routes.md#custom-log-level)
+ [`prefix`](https://github.com/fastify/fastify/blob/master/docs/Plugins.md#route-prefixing-option)

Fastify may directly support other options in the future. Thus, to avoid collisions, a plugin should consider namespacing its options. For example, a plugin `foo` might be registered like so:

```js
fastify.register(require('fastify-foo'), {
  prefix: '/foo',
  foo: {
    fooOption1: 'value',
    fooOption2: 'value'
  }
})
```

If collisions are not a concern, the plugin may simply accept the options object as-is:

```js
fastify.register(require('fastify-foo'), {
  prefix: '/foo',
  fooOption1: 'value',
  fooOption2: 'value'
})
```

The `options` parameter can also be a `Function` which will be evaluated at the time the plugin is registered while giving access to the fastify instance via the first positional argument:

```js
const fp = require('fastify-plugin')

fastify.register(fp((fastify, opts, done) => {
  fastify.decorate('foo_bar', { hello: 'world' })

  done()
}))

// The opts argument of fastify-foo will be { hello: 'world' }
fastify.register(require('fastify-foo'), parent => parent.foo_bar)
```

The fastify instance passed on to the function is the latest state of the **external fastify instance** the plugin was declared on, allowing access to variables injected via [`decorate`](https://github.com/fastify/fastify/blob/master/docs/Decorators.md) by preceding plugins according to the **order of registration**. This is useful in case a plugin depends on changes made to the Fastify instance by a preceding plugin (e.g. an existing database connection to wrap around).

Keep in mind that the fastify instance passed on to the function is the same as the one that will be passed into the plugin, a copy of the external fastify instance rather than a reference. Any usage of the instance will behave the same as it would if called within the plugin's function i.e. if `decorate` is called, the decorated variables will be available within the plugin's function unless it was wrapped with [`fastify-plugin`](https://github.com/fastify/fastify-plugin).

<a name="route-prefixing-option"></a>
#### Route Prefixing option
If you pass an option with the key `prefix` with a `string` value, Fastify will use it to prefix all the routes inside the register, for more info check out the [Routes](https://github.com/fastify/fastify/blob/master/docs/Routes.md#route-prefixing) documentation.<br>
Be aware that if you use [`fastify-plugin`](https://github.com/fastify/fastify-plugin) this option won't work.

<a name="error-handling"></a>
#### Error handling
Errors are handled by [avvio](https://github.com/mcollina/avvio#error-handling).<br>
As a general rule, it is highly recommended that you handle your errors in the next `after` or `ready` block, otherwise you will get them inside the `listen` callback.

```js
fastify.register(require('my-plugin'))

// `after` will be executed once
// the previously declared `register` has finished
fastify.after(err => console.log(err))

// `ready` will be executed once all the declared registers
// have executed
fastify.ready(err => console.log(err))

// `listen` is a special ready,
// so it behaves in the same way
fastify.listen(3000, (err, address) => {
  if (err) console.log(err)
})
```

*async-await* is supported only by `ready` and `listen`.
```js
fastify.register(require('my-plugin'))

await fastify.ready()

await fastify.listen(3000)
```
<a name="create-plugin"></a>
### Create a plugin
Creating a plugin is very easy, you just need to create a function that takes three parameters; the `fastify` instance, an `options` object and the `done` callback.<br>
Example:
```js
module.exports = function (fastify, opts, done) {
  fastify.decorate('utility', () => {})

  fastify.get('/', handler)

  done()
}
```
You can also use `register` inside another `register`:
```js
module.exports = function (fastify, opts, done) {
  fastify.decorate('utility', () => {})

  fastify.get('/', handler)

  fastify.register(require('./other-plugin'))

  done()
}
```
Sometimes, you will need to know when the server is about to close, for example because you must close a connection to a database. To know when this is going to happen, you can use the [`'onClose'`](https://github.com/fastify/fastify/blob/master/docs/Hooks.md#on-close) hook.

Do not forget that `register` will always create a new Fastify scope, if you don't need that, read the following section.

<a name="handle-scope"></a>
### Handle the scope
If you are using `register` only for extending the functionality of the server with  [`decorate`](https://github.com/fastify/fastify/blob/master/docs/Decorators.md), it is your responsibility to tell Fastify to not create a new scope, otherwise, your changes will not be accessible by the user in the upper scope.

You have two ways to tell Fastify to avoid the creation of a new context:
- Use the [`fastify-plugin`](https://github.com/fastify/fastify-plugin) module
- Use the `'skip-override'` hidden property

We recommend using the `fastify-plugin` module, because it solves this problem for you, and you can pass a version range of Fastify as a parameter that your plugin will support.
```js
const fp = require('fastify-plugin')

module.exports = fp(function (fastify, opts, done) {
  fastify.decorate('utility', () => {})
  done()
}, '0.x')
```
Check out the [`fastify-plugin`](https://github.com/fastify/fastify-plugin) documentation to learn more about how use this module.

If you don't use the `fastify-plugin` module, you can use the `'skip-override'` hidden property, but we do not recommend it. If the Fastify API changes in the future it will be your responsibility to update the module, while if you use `fastify-plugin`, you don't have to worry about backwards compatibility.
```js
function yourPlugin (fastify, opts, done) {
  fastify.decorate('utility', () => {})
  done()
}
yourPlugin[Symbol.for('skip-override')] = true
module.exports = yourPlugin
```
