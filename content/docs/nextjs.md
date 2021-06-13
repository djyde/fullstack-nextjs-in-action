# Next.js

## Using middleware in API route

First, let me explain what is middleware and why we need middleware layer in Next.js.

In Express/Koa, we can use middlewares to access the `request` object and `response` object in a request-response cycle. A middleware is a function that take three params: `req`, `res` and `next`:

```js
function exampleMiddleware(req, res, next) {
  if (/** ...*/) {
    req.foo = 'bar'
  	next()
  } else {
    res.statusCode = 403
    res.send('forbiddon')
  }
}
```

>  Since this middleware pattern is first introduced by a Node.js framework [connect](https://github.com/senchalabs/connect), we always call these middlewares `connect-like` middleware.

In a middleware, we can both:

- Inject some properties on the `req` object, which can be used on the next middleware or route hadler (controller).
- Suspend the request by not calling `next()` and modify the response (such as status code, response body).

This is very usable for reusing logic between routes. Imagine we need to get current signed in user's info in some routes, we can write this logic into a middleware, then apply it before the routes that need to get user's info. Here is an example using middleware in Express:

```js
function authMiddleware(req, res, next) {
  // suppose we saved a `token` in cookies after user signing in
  if (req.cookies.token) {
    const user = getUserByToken(req.cookies.token)
    req.user = user
    next()
  } else {
    // user not signed in
    res.statusCode = 403
    res.send('please sign in first')
  }
}

// route that doesn't need user's information
app.get('/', (req, res) => {
  res.send('hello world')
})

// route that need user's information
app.get('/profile', authMiddleware, (req, res) => {
  res.send(`welcome! ${req.user.name}`)
})
```

Middleware makes code very reusable. But in Next.js, there isn't a middleware layer like it. Imagine we need to do the same thing in some of Next.js API route handlers:

```ts
// pages/api/example.ts

function auth(req, res) {
  if (req.cookies.token) {
    const user = getUserByToken(req.cookies.token)
    return user
  } else {
    // user not signed in
    res.status(403)
    res.send('please sign in first')
  }
}

export default (req, res) => {
  if (req.method === 'GET') {
    res.send('hello')
  } else if (req.method === 'POST') {
    const user = auth(req, res)
    console.log('do other things')
	  res.send(`welcome! ${user.name}`)
  }
}
```

It seems good but what if `const user = auth(req, res)` return nothing? `res.status(403)` would be executed but it doesn't suspend the whole request, `console.log('do other things')` is gonna be executed unexpectedly.

### Introducing `next-connect`

To make use of connect-like middlewares in Next.js, we can use an amazing library —— [next-connect](https://github.com/hoangvvo/next-connect). 



> Install `next-connect`:
>
> ```bash
> $ yarn add next-connect
> ```



Now, let's see how we can use `next-connect` in a Next.js api route handler:

```js
// pages/api/example.ts

import nc from 'next-connect'

function authMiddleware(req, res, next) {
  // just make a 403 response
  res.status(403)
  res.send('please sign in first')
}

// First, we use `nc()` to create a api route handler`
const handler = nc()
  .get((req, res) => {
    res.send('hello')
  })
  .post(authMiddleware, (req,res) => {
    res.send('hello')
  })

export default handler
```

In the above code example, you can see we can now use connect-like middleware in api route, just like how we can do in Express. The `authMiddleware` change the response status code to 403 and send request body, but didn't call `next()`, so the `res.send('hello')` code in the POST handler wouldn't be executed.

Another greate benefit of using next-connect is we can use `.get()`, `.post()`, `put()`, etc, rather than using `if (req.method === XXX)`. It makes code more readable.

Another example is to enable [CORS]() in an API route. We can just use the Express `cors` middleware:

> Install `cors`:
>
> ```bash
> $ yarn add cors
> ```

```diff
// pages/api/example.ts

import nc from 'next-connect'
+ import * as cors from 'cors'

const corsOptions = {
  origin: 'http://example.com',
  optionsSuccessStatus: 200 // some legacy browsers (IE11, various SmartTVs) choke on 204
}

// First, we use `nc()` to create a api route handler`
const handler = nc()
+	.use(cors(corsOptions))
  .get((req, res) => {
    res.send('hello')
  })
  .post(authMiddleware, (req,res) => {
    res.send('hello')
  })

export default handler
```

## Handling error in API routes

In a practical HTTP API design, we need to set corresponding http code and message in the corresponding scenario. For example, when the resource is not found, we should response a 404 status; when an unauthorized user call a protected API, we should response a 403 status.

Let's see how do we handle error in Express: it comes with a default [error handler](https://expressjs.com/en/guide/error-handling.html):

> - The `res.statusCode` is set from `err.status` (or `err.statusCode`). If this value is outside the 4xx or 5xx range, it will be set to 500.
> - The `res.statusMessage` is set according to the status code.
> - The body will be the HTML of the status code message when in production environment, otherwise will be `err.stack`.
> - Any headers specified in an `err.headers` object.

In other word, the error handler will read the error object that being thrown. According to the error object's properties, the corresponding response status information would be set automatically.

This makes it very easy to throw the "right" HTTP error. For example, we can throw a 404 error every where, without manually modify the `res` object:

```js
function getProjectById(id) {
  const project = db.project.find(id)
  if (!project) {
    throw {
      status: 404
    }
  }
  
  return project
}

app.get('/project/:id', (req, res) => {
  const project = getProjectById(req.params.id)
  res.json({ data: project })
})
```

Another famous web framework [hapi](https://hapi.dev/) chooses a more elegant approach: it provides a[ `@hapi/boom`](https://hapi.dev/module/boom/api/?v=9.1.2) module , which is a set of utility functions for returning HTTP erros. For example, `Boom.forbidden('Please sign in first')` will return:

```json
{
    "statusCode": 403,
    "error": "Forbidden",
    "message": "Please sign in first"
}
```

We can throw this error object in hapi's route handler:

```js
throw Boom.forbidden('Please sign in first')
```

The error handler in `hapi` will catch the error and response the corresponding HTTP status and message body. 

Next.js, in contrast, it's quite hard to handle error in API route because no matter what error was thrown in API routes, Next.js always response a same structure —— status code `500`  with a  ` Internal Servre Error` body. It doesn't have a way to customize the error handler.

Fortunately, `next-connect`, the library we introduce in the previous section, has a catch-all error handler. All the errors thrown in middlewares or api route handlers (they are middlewares too, actually) will be catched in a single place:

```js
import nc from 'next-connect'

const handler = () => nc({
  onError(err, req, res) {
    console.log(err.message) // => 'oops'
    res.send(err.message)
  }
})

export default handler()
	.get((req, res) => {
		throw new Error('oops')
	})
```

![image-20210612044906647](/assets/image-20210612044906647.png)

But why not leverage the power of `@hapi/boom` in Next.js? We can throw Boom error in every where and handle them on the `onError` handler:



> Install `@hapi/boom`:
>
> ```bash
> $ yarn add @hapi/boom
> ```



```js
// pages/api/example.ts

import nc from "next-connect";
import * as Boom from '@hapi/boom'

const handler = () =>
  nc({
    onError(err, req, res) {
      if (Boom.isBoom(err)) {
        res.status(err.output.payload.statusCode);
        res.json({
          error: err.output.payload.error,
          message: err.output.payload.message,
        });
      } else {
        res.status(500);
        res.json({
          message: "Unexpected error",
        });
        console.error(err);
        // unexcepted error
      }
    },
  });

function getProjects() {
  throw Boom.forbidden('Please sign in first')
}

export default handler()
  .get(async (req, res) => {
    const projects = getProjects()
    res.json({
      projects
    })
  });

```

In the error handler, we firstly check if the error object is a `Boom` error object. If it is, set the response to corresponding status code and message from the `Boom` error object.

![image-20210612045920643](/assets/image-20210612045920643.png)

# 