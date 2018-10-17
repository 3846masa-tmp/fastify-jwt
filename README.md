# fastify-jwt

[![js-standard-style](https://img.shields.io/badge/code%20style-standard-brightgreen.svg?style=flat)](http://standardjs.com/)  [![Build Status](https://travis-ci.org/fastify/fastify-jwt.svg?branch=master)](https://travis-ci.org/fastify/fastify-jwt) [![Greenkeeper badge](https://badges.greenkeeper.io/fastify/fastify-jwt.svg)](https://greenkeeper.io/)

JWT utils for Fastify, internally uses [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken).

## Install
```
npm i fastify-jwt --save
```

## Usage
Register as a plugin. This will decorate your `fastify` instance with the standard [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) methods `decode`, `sign`, and `verify`; refer to their documentation to find how to use the utilities. It will also register `request.jwtVerify` and `reply.jwtSign`. You must pass a `secret` when registering the plugin.

```js
const fastify = require('fastify')
fastify.register(require('fastify-jwt'), {
  secret: 'supersecret'
})

fastify.post('/signup', (req, reply) => {
  // some code
  const token = fastify.jwt.sign({ payload })
  reply.send({ token })
})

fastify.listen(3000, err => {
  if (err) throw err
})
```

For verifying & accessing the decoded token inside your services, you can use a global `preHandler` hook to define the verification process like so:

```js
const fastify = require('fastify')
fastify.register(require('fastify-jwt'), {
  secret: 'supersecret'
})

fastify.addHook("preHandler", async (request, reply) => {
  try {
    await request.jwtVerify()
  } catch (err) {
    reply.send(err)
  }
})
```

Aftewards, just use `request.user` in order to retrieve the user information:

```js
module.exports = async function(fastify, opts) {
  fastify.get("/", async function(request, reply) {
    return request.user
  })
}
```

However, most of the time we want to protect only some of the routes in our application. To achieve this you can wrap your authentication logic into a plugin like

```js
const fp = require("fastify-plugin")

module.exports = fp(async function(fastify, opts) {
  fastify.register(require("fastify-jwt"), {
    secret: "supersecret"
  })

  fastify.decorate("authenticate", async function(request, reply) {
    try {
      await request.jwtVerify()
    } catch (err) {
      reply.send(err)
    }
  })
})
```

Then use the `beforeHandler` of a route to protect it & access the user information inside:

```js
module.exports = async function(fastify, opts) {
  fastify.get(
    "/",
    {
      beforeHandler: [fastify.authenticate]
    },
    async function(request, reply) {
      return request.user
    }
  )
}
```

Make sure that you also check [fastify-auth](https://github.com/fastify/fastify-auth) plugin for composing more complex strategies.

## API Spec

### fastify-jwt
`fastify-jwt` is a fastify plugin. You must pass a `secret` to the `options` parameter. The `secret` can be a primitive type String, a function that returns a String or an object `{ private, public }`.

In case of a private key with passphrase an object `{ private: { key, passphrase }, public }` or `{ private: key, public: passphrase }` can be used (based on [crypto documentation](https://nodejs.org/api/crypto.html#crypto_sign_sign_private_key_output_format)), in this case be sure you pass the `algorithm` option).

Function based `secret` is supported by the `request.jwtVerify()` and `reply.jwtSign()` methods and is called with `request`, `reply`, and `callback` parameters.
#### Example
```js
const { readFileSync } = require('fs')
const path = require('path')
const fastify = require('fastify')()
const jwt = require('fastify-jwt')
// secret as a string
fastify.register(jwt, { secret: 'supersecret' })
// secret as a function
fastify.register(jwt, {
  secret: function (request, reply, callback) {
    // do something
    callback(null, 'supersecret')
  }
})
// secret as an object of RSA keys (without passphrase)
// assuming the key files are inside a certs directory and loaded as strings
fastify.register(jwt, {
  secret: {
    private: readFileSync(`${path.join(__dirname, 'certs')}/private.key`, 'utf8')
    public: readFileSync(`${path.join(__dirname, 'certs')}/public.key`, 'utf8')
  },
  options: { algorithm: 'RS256' }
})
// secret as an object of RSA keys (with a passphrase)
// assuming the pem files are inside a certs directory and loaded as buffers
fastify.register(jwt, {
  secret: {
    private: {
      key: readFileSync(`${path.join(__dirname, 'certs')}/private.pem`),
      passphrase: 'super secret passphrase'
    },
    public: readFileSync(`${path.join(__dirname, 'certs')}/public.pem`)
  },
  options: { algorithm: 'RS256' }
})
```

### fastify.jwt.sign(payload [,options] [,callback])
The `sign` method is an implementation of [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken#jwtsignpayload-secretorprivatekey-options-callback) `.sign()`. Can be used asynchronously by passing a callback function; synchronously without a callback.

### fastify.jwt.verify(token, [,options] [,callback])
The `verify` method is an implementation of [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken#jwtverifytoken-secretorpublickey-options-callback) `.verify()`. Can be used asynchronously by passing a callback function; synchronously without a callback.
#### Example
```js
const token = fastify.jwt.sign({ foo: 'bar' })
// synchronously
const decoded = fastify.jwt.verify(token)
// asycnhronously
fastify.jwt.verify(token, (err, decoded) => {
  if (err) fastify.log.error(err)
  fastify.log.info(`Token verified. Foo is ${decoded.foo}`)
})
```

### fastify.jwt.decode(token [,options])
The `decode` method is an implementation of [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken#jwtdecodetoken--options) `.decode()`. Can only be used synchronously.
#### Example
```js
const token = fastify.jwt.sign({ foo: 'bar' })
const decoded = fastify.jwt.decode(token)
fastify.log.info(`Decoded JWT: ${decoded}`)
```

### fastify.jwt.secret
For your convenience, the `secret` you specify during `.register` is made available via `fastify.jwt.secret`. `request.jwtVerify()` and `reply.jwtSign()` will wrap non-function secrets in a callback function. `request.jwtVerify()` and `reply.jwtSign()` use an asynchronous waterfall method to retrieve your secret. It's recommended that your use these methods if your `secret` method is asynchronous.

### reply.jwtSign(payload, [options,] callback)
### request.jwtVerify([options,] callback)
These methods are very similar to their standard jsonwebtoken counterparts.
#### Example
```js
const fastify = require('fastify')()

fastify.register(jwt, {
  secret: function (request, reply, callback) {
    // do something
    callback(null, 'supersecret')
  }
})

fastify.post('/sign', function (request, reply) {
  reply.jwtSign(request.body.payload, function (err, token) {
    return reply.send(err || { 'token': token })
  })
})

fastify.get('/verify', function (request, reply) {
  request.jwtVerify(function (err, decoded) {
    return reply.send(err || decoded)
  })
})

fastify.listen(3000, function (err) {
  if (err) fastify.log.error(err)
  fastify.log.info(`Server live on port: ${fastify.server.address().port}`)

  // sign payload and get JWT
  request({
    method: 'POST',
    headers: {
      'Content-Type': 'application/json'
    },
    body: {
      payload: {
        foo: 'bar'
      }
    },
    uri: `http://localhost:${fastify.server.address().port}/sign`,
    json: true
  }, function (err, response, body) {
    if (err) fastify.log.error(err)
    fastify.log.info(`JWT token is ${body}`)

    // verify JWT
    request({
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        authorization: 'Bearer ' + sign.token
      },
      uri: 'http://localhost:' + fastify.server.address().port + '/verify',
      json: true
    }, function (err, response, body) {
      if (err) fastify.log.error(err)
      fastify.log.info(`JWT verified. Foo is ${body.bar}`)
    })
  })
})
```


## Acknowledgements

This project is kindly sponsored by:
- [LetzDoIt](http://www.letzdoitapp.com/)

## License

Licensed under [MIT](./LICENSE).
