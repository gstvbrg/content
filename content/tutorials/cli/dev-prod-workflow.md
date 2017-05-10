---
alias: ex4wo4zaep
path: /docs/tutorials/cli-dev-prod-workflow
layout: TUTORIAL
description: Learn best practices to manage a development-production-workflow with the Graphcool CLI
tags:
  - cli
related:
  more:
  further:
---


# Dev-Production-Workflow with the Graphcool CLI

## Overview

In most real-world project, developers maintain (at least) two different environments:

- **Development (_dev_)**: Used during the development phase 
- **Production (_prod_)**: Used when the app gets deployed, this is where real user data is stored and managed

In this tutorial, we want to show a typical workflow for how these two different environments can be properly maintained using the [Graphcool CLI](https://www.npmjs.com/package/graphcool).

## Getting Started

Let's assume we're starting from scratch with a brandnew Instagram application that has the following simple schema:

```graphql
type Post {
  description: String!
  imageUrl: String!
}
```

The workflow we are going to simulate in this tutorial looks as follows:

1. Create new Graphcool project that is used in dev
2. Prepare a clone of the dev project for prod
3. Iterate on the application by adjusting the schema of the dev project 
4. Move all changes that happened in dev to prod

## 1. Creating development project

Let's go ahead and create the development environment with `graphcool init`:

```bash
graphcool init --schema https://graphqlbin.com/instagram.graphql --name InstagramDev -alias insta-dev --output dev.graphcool
```

This will create a new project called `InstagramDev` in our Graphcool account. In future commands, we can further refer to the project by using the _alias_ `insta-dev` instead of using the project ID. The schema is fetched from the remote URL `https://graphqlbin.com/instagram.grapqhl` and the project file will be written to `dev.graphcool`.

This what the project `dev.graphcool` file look like:

```graphql
# project: insta-dev
# version: 1

type File implements Node {
  contentType: String!
  createdAt: DateTime!
  id: ID! @isUnique
  name: String!
  secret: String! @isUnique
  size: Int!
  updatedAt: DateTime!
  url: String! @isUnique
}

type Post implements Node {
  createdAt: DateTime!
  description: String!
  id: ID! @isUnique
  imageUrl: String!
  updatedAt: DateTime!
}

type User implements Node {
  createdAt: DateTime!
  id: ID! @isUnique
  updatedAt: DateTime!
}
```

This is the project that we'll use for our development environment. It will never actually store any data from our end users.

## 2. Cloning for production

Fast forward a bit and assume we're now done with all the frontend work and ready to deploy version 1.0 of the app. Before we do so, we need to create the _production environment_ with a schema that's identical to the one from dev. This can be done by calling the `init` command again and passing in the `--copy` and `--copy-options` arguments:

```
graphcool init --copy insta-dev --name Instagram --alias insta-prod --copy-options mutation-callbacks --output prod.graphcool 
```

This will create the same project file as above excepted for the first line which now is:

```
# project: insta-prod 
```

Depending on the client technology that was used, we now only have to adjust the API endpoints in the code that we wrote in the actual application and are ready to deploy 🚀


## 3. Another product iteration

After version 1.0 of our app was deployed, we collected some feedback and did another iteration on the schema. In particular, there are two changes that we added incrementally to our app (and schema):

1. Create a relation between `Post` and `User` so that it's clear which user created a post. We thus update the `Post` and the `User` type by adding a new _one-to-many_-relation:

```graphql
type Post implements Node {
  ...
  postedBy: User @relation(name: "PostsByUser")
}

type User implements Node {
  ...
  posts: [Post!] @relation(name: "PostsByUser")
}
```

The changes were submitted by using `graphcool push dev.graphcool`.

2. Add a new type `Comment` including a _many-to-many_-relation to `Post` and a _one-to-many_-relation to `User` so that users can comment on Posts.

```graphql
type Comment implements Node {
  author: User @relation(name: "CommentsByUser")
  post: Post @relation(name: "CommentsOnPost")
}

type Post implements Node {
  ...
  comments: [Comment!] @relation(name: "CommentsOnPost")
}

type User implements Node {
  ...
  comments: [Comment!] @relation(name: "CommentsByUser")
}
```

Again, `graphcool push dev.graphcool` was used to make these changes take effect.

After we've performed these migrations, our project file has now grown to look as follows:

```graphql
# project: insta-dev
# version: 3

type Comment implements Node {
  author: User @relation(name: "CommentsByUser")
  createdAt: DateTime!
  id: ID! @isUnique
  post: Post @relation(name: "CommentsOnPost")
  updatedAt: DateTime!
}

type File implements Node {
  contentType: String!
  createdAt: DateTime!
  id: ID! @isUnique
  name: String!
  secret: String! @isUnique
  size: Int!
  updatedAt: DateTime!
  url: String! @isUnique
}

type Post implements Node {
  comments: [Comment!]! @relation(name: "CommentsOnPost")
  createdAt: DateTime!
  description: String!
  id: ID! @isUnique
  imageUrl: String!
  postedBy: User @relation(name: "PostsByUser")
  updatedAt: DateTime!
}

type User implements Node {
  comments: [Comment!]! @relation(name: "CommentsByUser")
  createdAt: DateTime!
  id: ID! @isUnique
  posts: [Post!]! @relation(name: "PostsByUser")
  updatedAt: DateTime!
}
```

## 4. Update production environment

To make sure our changes are transferred to the production environment, we have to update the schema in `prod.graphcool` manually and then performing the migration as before. 

The manual process can also be replaced by a script where the schema from `dev.graphcool` are written automatically into `prod.graphcool`. Assuming we either used such a script or pasted the schema over manually, our `prod.graphcool` file should now look similar to `dev.graphcool` - except for the frontmatter:

```graphql
# project: insta-prod
# version: 1

type Comment implements Node {
  author: User @relation(name: "CommentsByUser")
  createdAt: DateTime!
  id: ID! @isUnique
  post: Post @relation(name: "CommentsOnPost")
  updatedAt: DateTime!
}

type File implements Node {
  contentType: String!
  createdAt: DateTime!
  id: ID! @isUnique
  name: String!
  secret: String! @isUnique
  size: Int!
  updatedAt: DateTime!
  url: String! @isUnique
}

type Post implements Node {
  comments: [Comment!]! @relation(name: "CommentsOnPost")
  createdAt: DateTime!
  description: String!
  id: ID! @isUnique
  imageUrl: String!
  postedBy: User @relation(name: "PostsByUser")
  updatedAt: DateTime!
}

type User implements Node {
  comments: [Comment!]! @relation(name: "CommentsByUser")
  createdAt: DateTime!
  id: ID! @isUnique
  posts: [Post!]! @relation(name: "PostsByUser")
  updatedAt: DateTime!
}
```

We can apply the changes by calling `graphcool push prod.graphcool` and are ready to deploy our next product iteration. 😎






