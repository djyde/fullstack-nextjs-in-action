# Data fetching

Fetching data and updating data via HTTP API are not that hard. Here is a basic example with `axios`: 

```jsx
// pages/index.tsx

import axios from 'axios'
import React from 'react'

async function getPosts() {
  const result = await axios.get('https://jsonplaceholder.typicode.com/posts?_limit=5')
  return result.data
}

async function addPost(title, body) {
  await axios.post('https://jsonplaceholder.typicode.com/posts', {
    title, body
  })
}

function IndexPage() {

  const [posts, setPosts] = React.useState([])

  async function initial() {
    const result = await getPosts()
    setPosts(result)
  }

  React.useEffect(() => {
    initial()
  }, [])

  async function onClickAddPost() {
    await addPost('foo', 'bar')
  }

  return (
    <>
      <div>
        {posts.map(post => {
          return (
            <div key={post.id}>
              {post.title}
            </div>
          )
        })}
      </div>
      <button onClick={onClickAddPost}>add post</button>
    </>
  )
}

export default IndexPage
```

![image-20210612132751487](/assets/image-20210612132751487.png)

Every HTTP request has their own state:

- loading status: is loading or is not.
- if request success, the response data
- if request failed, the response error data

Managing these state for every HTTP request will cause us to write duplicated code. This is why we need `react-query`.

There are two major concepts in `react-query`:

{{< details title="Setting up react-query in Next.js" open=true >}}

> Install:
>
> ```bash
> $ yarn add react-query
> ```
>
> In `pages/_app.tsx`, add a `QueryClientProvider` at the top of the root component:
>
> ```jsx
> // pages/_app.tsx
> 
> import React from 'react'
> import { QueryClient, QueryClientProvider } from 'react-query'
> 
> export const queryClient = new QueryClient()
> 
> export default function MyApp({ Component, pageProps }) {
> 
> 	return (
>    <QueryClientProvider client={queryClient}>
>      <Component {...pageProps} />
>    </QueryClientProvider>
>  )
> }
> ```
{{< /details >}}

## Query

Fetching data from server is  `query`.

```jsx
// pages/index.tsx

import axios from 'axios'
import React from 'react'
import { useQuery } from 'react-query'

async function getPosts() {
  const result = await axios.get('https://jsonplaceholder.typicode.com/posts?_limit=5')
  return result.data
}

function IndexPage() {

  const getPostsQuery = useQuery('getPosts', getPosts)

  return (
    <>
      <div>
    		{getPostsQuery.isLoading && <div>Loading...</div>}
	      {getPostsquery.error && <div>Something error</div>}
        {getPostsQuery.data?.map(post => {
          return (
            <div key={post.id}>
              {post.title}
            </div>
          )
        })}
      </div>
    </>
  )
}

export default IndexPage
```

We use `useQuery()` to define a query. Every query has a key for being identify. `react-query` use this key to cache query result data. It means that if user has already queried `getPosts`, the other query with this key in other components will first get the cached data before fetching data again, and then refresh the cache and re-render the UI.

Imagin there are two pages would display posts list. They both have `useQuery('getPosts')`. When user comes to the first page, he will see a loading spinner. But when he navigate to the second page, he will instantly see the posts list without waiting to load because the data was cached.

As you can see in the above code example, we can use `isLoading` to know if the HTTP request is loading. And we can use `data` to access to the response data. 

## Mutation

Updating data (such as `POST`, `PUT`, `DELETE` request) is called mutation. To create a mutation, use `useMutation()`. This method accepts a mutation function. Then we can call `.mutate()` to muate a mutation with some variables. Let's rewrite the above example:

```jsx
// pages/index.tsx

import axios from 'axios'
import React from 'react'
import { useMutation, useQuery } from 'react-query'

async function getPosts() {
  const result = await axios.get('https://jsonplaceholder.typicode.com/posts?_limit=5')
  return result.data
}

async function addPost({ title, body }) {
  await axios.post('https://jsonplaceholder.typicode.com/posts', {
    title, body
  })
}

function IndexPage() {

  const getPostsQuery = useQuery('getPosts', getPosts)

  const addPostMutation = useMutation(addPost)

  function onClickAddPost() {
    addPostMutation.mutate({ title: 'foo', body: 'bar' })
  }

  return (
    <>
      <div>
        {getPostsQuery.isLoading && <div>Loading...</div>}
        {getPostsQuery.data?.map(post => {
          return (
            <div key={post.id}>
              {post.title}
            </div>
          )
        })}

	      {addPostMutation.isLoading && <div>Adding post...</div>}
        <button onClick={onClickAddPost}>Add post</button>
      </div>
    </>
  )
}

export default IndexPage
```

We can even do something when the mutation success or fail:

```jsx
function onClickAddPost() {
  addPostMutation.mutate({ title: 'foo', body: 'bar' }, {
    onSuccess(data) {
      // ...
    },
    onError(err) {
      // ...
    }
  })
}
```

In general, when user add new post, the `getPosts()` result is supposed to be outdated. To make a better user experience, we should refetch all the query that associated with fetching posts list.

Every query has a `refetch()` method, we can call it when the mutation is success:

```diff
function onClickAddPost() {
  addPostMutation.mutate({ title: 'foo', body: 'bar' }, {
    onSuccess(data) {
+			getPostsQuery.refetch()
    },
    onError(err) {
      // ...
    }
  })
}
```

But it's not a good idea. Because the queries asscociated with fetching posts list may appear in any where around the entire project. We should use an important feature in `react-query`  ——  `Query Invalidation`.

## Query Invalidation

You can call `invalidateQueries()` from `queryClient` to mark a query as stale, since every query has a unique key. In our example, we should mark `getPosts` query as stale:

```diff
+ import { queryClient } from './_app'
function onClickAddPost() {
  addPostMutation.mutate({ title: 'foo', body: 'bar' }, {
    onSuccess(data) {
-			getPostsQuery.refetch()
+			queryClient.invalidateQueries('getPosts')
    },
    onError(err) {
      // ...
    }
  })
}
```

When the query is marked as stale, `react-query` will refetch all the queries with the `getPosts`  key. The UI will be automatically updated with the up-to-date data.