
# Handling error in API routes

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
