# Interacting with database

I often use [Prisma](https://prisma.io) with Next.js, which is a Node.js ORM. If you've never heard of Prisma before, **Prisma let you declare your database model and their relations in a single schema file, then it generates database migration, type-safe database client for you**. 

Install Prisma:

```bash
$ yarn add prisma @prisma/client

$ yarn prisma init
```

`yarn prisma init` will generate a default Prisma schema file in `prisma/schema.prisma`. 

By default, Prisma will use PostgreSQL as the provider. For convenience, we use SQLite in local environment. Modify the `prisma/schema.prisma` as below:

```diff
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

datasource db {
- provider = "postgresql"
- url      = env("DATABASE_URL")
+ provider = "sqlite"
+ url      = "file:../db.sqlite"
}

generator client {
  provider = "prisma-client-js"
}
```

It means Prisma will use the SQLite database in `../db.sqlite`. 

Imagine we are developing a blog system. We have three tables: `Post`, `Comment`, `Tag`:

- One `Post` has many `Comment`, but one `Comment` only belongs to one `Post`. They are one-to-many relation
- One `Post` has many `Tag`, and `Tag` can belong to many `Post`. They are many-to-many relation

Here is how we design the tables:

![image-20210612155159566](/assets/image-20210612155159566.png)



## Declarating data models

We need to write some sql to initialize our database. But if we use Prisma, all the things we need to do is just declare the data model and their relations in the schema file:

```graphql
datasource db {
  provider = "sqlite"
  url      = "file:../db.sqlite"
}

generator client {
  provider = "prisma-client-js"
}

model Post {
  id String @id

  content String

  createdAt DateTime @default(now())

  tags Tag[] @relation("post_tag")

  comments Comment[] @relation("post_comment")
}

model Comment {
  id String @id
  content String

  postId String
  post Post @relation(name: "post_comment", references: [id], fields: [postId])

  createdAt DateTime @default(now())
}

model Tag {
  label String @id

  posts Post[] @relation("post_tag")
}
```

Even though you've never learned Prisma Schema syntax, you can understand how the database is like through this file, because the syntax is super simple and straight forward. Especially the delcaration of the relation between to model. 

> [More about Prisma Schema syntax](https://www.prisma.io/docs/concepts/components/prisma-schema)

Now, run `yarn prisma db push`, Prisma will generate a database base on the schema. It knows how to create foreign key and how to create a many-to-many relation table.

Then run `yarn prisma generate`. This command will generate the database client base on the schema to `node_modules/@prisma/client`. So you can import the `PrismaClient` from `@prisma/client` in your code:

```js
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

export default (req, res) => {
  const posts = await prisma.post.findMany()
  res.json({
    posts
  })
}
```

![image-20210612162004027](/assets/image-20210612162004027.png)

(It's type-safe)

## Datebase Migration

`yarn prisma db push` is only for protoyping. It force apply the schema into your database, it may cause data lose. If you already have an online running database, you should generate a db migration file:

```
$ yarn prisma migrate dev
```

Prisma will generate migration files in `prisma/migrations` folder. When you want to deploy your change online, you should run:

```
$ yarn prisma migrate deploy
```

Prisma will apply the migrations to your database.

> More about [Prisma Migrate](https://www.prisma.io/docs/concepts/components/prisma-migrate#production-and-testing-environments)

## Best practice of using Prisma in Next.js

Initialize `PrismaClient` in Next.js will cause `too many connections` error. Because in dev environment, Next.js hot reload will clear cache and re-run the code. Every time a new `PrismaClient` is initialized, a new db connection will be created. 

We should keep the prisma client instance be singleton, by caching the instance in `global` object:

```js
import { PrismaClient } from '@prisma/client'

export const prisma = global.prisma || new PrismaClient()
if (process.env.NODE_ENV !== "production") global.prisma = prisma
```

Now, instead of initializing on each api routes, we can import `prisma` from a single source of truth.