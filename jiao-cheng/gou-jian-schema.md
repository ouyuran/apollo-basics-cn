---
title: '1. Build a schema'
description: 画出 graph 数据的蓝图
---

实现graph API的第一步是构造 **schema**。你可以把 schema 看做是 graph 所能访问的所有数据的蓝图。通过这个章节，你将学习到怎样在Apollo中构造 graph 的 schema。

## 搭建 Apollo 服务器

开始写 schema 之前，我们须要搭建 graph API 的服务器。**Apollo Server** 是一个帮你实现 graph API 的库。它可以连接到任何数据源，包括 REST API 和数据库，还可以无缝集成到 Apollo 开发者工具。

从头开始，让我们先安装一下依赖：

```bash
cd start/server && npm install
```

你需要 `apollo-server` 和 `graphql` 这两个包， 它们已经通过刚才的命令安装好了。现在，进入 `src/index.js` 这个文件，把下面的代码拷贝进去。

_src/index.js_

```js
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');

const server = new ApolloServer({ typeDefs });
```

为了实现我们的 graph API ，我们需要从 `apollo-server` 导入 `ApolloServer` 类。我们须要从 `src/schema.js` 导入自己写的 schema 。接下来，让我们创建一个 `ApolloServer` 的实例并把我们的 schema 作为配置对象的 `typeDefs` 属性传入。（译者：这里用到了ES6语法，不熟悉的同学移步[ES6 对象的扩展](http://es6.ruanyifeng.com/#docs/object)）

在启动服务器前，先让我们编写 schema 。

## 编写 graph 的 schema

每个 graph API 都围绕它的 schema 。这么解释 schema 吧，它就像一幅蓝图，定义了数据的类型和相互关系。它同时定义你能通过 query 获取哪些数据，通过 mutation 更新哪些数据。Schema 是强类型的，这带来了强大的开发者工具。

Schema 最好根据使用它的客户端的需求来定义。既然 schema 处于客户端和服务器之间，它是前端团队和后端团队合作的最好的中间地带。我们推荐团队实践 **Schema 优先** 的开发策略：先就 schema 达成一致再写代码。

现在，想想我们的应用需要哪些数据， 需求是：

- 获取所有即将到来的火箭发射
- 用 ID 获取某个特定的火箭发射
- 用户登录
- 登录用户预订一次星际旅行
- 登录用去取消一次星际旅行

我们的 schema 将基于以上这些功能定义。在 `src/schema.js` 文件中导入 `gql` ，然后为 schema 创建一个名为 `typeDefs` 的变量。

_src/schema.js_

```js
const { gql } = require('apollo-server');

const typeDefs = gql`
  # schema 将写在这里
`;

module.exports = typeDefs;
```

### Query 类型

让我们从 **Query 类型** 开始，它定义了我们能获取到什么数据。

我们用来编写 schema 的语言是 GraphQL 定义语言（SDL）。如果你用过 TypeScript ,它们看起来有点相似。把下面的 SDL 代码拷贝到 `gql` 函数后面的反引号中间，也就是上一段代码的注释位置。

_src/schema.js_

```graphql
type Query {
  launches: [Launch]!
  launch(id: ID!): Launch
  # Queries for the current user
  me: User
}
```

我们先定义一个 `launches` query 来获取所有的火箭发射。这个请求返回一个 `Launch` 数组，这个数组不能为空。GraphQL中所有类型的默认情况下都是可以为空的，如果我们希望我们的 query 必须返回一个非空结果，那么我们须要在返回类型后加上 `!`。然后我们再定义一个 query 来通过 ID 获取一个火箭发射 `launch`。这个 query 接受一个 `id` 参数，返回一个 `Launch`。最后，我们定义了一个 `me` query 来获取当前的用书数据。你会发现在 `launch` 和 `me` 之间有 `#` 打头的一行，没错这就是 SDL 中注释的写法。

那么我们怎么知道 `Launch` 和 `User` 都包含哪些属性呢？答案是我们需要定义 GraphQL 的对象类型。

### 对象类型和标量类型

让我们像创建一个 **对象类型** 来定义 `Launch` 的结构。再一次，把下面的 SDL 代码拷贝到刚才的代码后面。

_src/schema.js_

```graphql
type Launch {
  id: ID!
  site: String
  mission: Mission
  rocket: Rocket
  isBooked: Boolean!
}
```

`Launch` 类型有五个字段，一些是对象类型另一些是标量类型。标量类型是像 `ID` 、 `String` 、 `Boolean` 和 `Int` 这样的原始类型。Graph 解析的时候会相当于展开对象类型，所以对象类型展开成树状后，所有叶子节点都会是一个标量类型。GraphQL 有许多内置的标量类型，你也可以自定义，比如 `Date`。
;TODO， link to custom scalars

`Mission` 和 `Rocket` 是两个其它的对象类型，让我们来定义它们：

_src/schema.js_

```graphql
type Rocket {
  id: ID!
  name: String
  type: String
}

type User {
  id: ID!
  email: String!
  trips: [Launch]!
}

type Mission {
  name: String
  missionPatch(size: PatchSize): String
}

enum PatchSize {
  SMALL
  LARGE
}
```

你可能发现 `missionPatch` 这个字段可以接受入参  `size`。 GraphQL 是很灵活的，每个字段都可以包括入参，教程的后面将会解释这些入参如何起作用。`size` 入参是一个 **enum 类型**，这个类型在最底下定义为 `PatchSize`。

当你编写 schema 时，你还可能遇到一些不常见的类型。你可以在[类型全表](https://devhints.io/graphql#schema)找到所有的类型。

### Mutation 类型

现在，让我们定义 **Mutation 类型**。`Mutation` 是更新数据的入口，就像 `Query` 类型一样，`Mutation` 类型是一种特殊的对象类型。

_src/schema.js_

```graphql
type Mutation {
  # if false, booking trips failed -- check errors
  bookTrips(launchIds: [ID]!): TripUpdateResponse!

  # if false, cancellation failed -- check errors
  cancelTrip(launchId: ID!): TripUpdateResponse!

  login(email: String): String # login token
}
```

`bookTrips` 和 `cancelTrip` 这两个 mutation 都接受一个入参并返回一个 `TripUpdateResponse`。GraphQL mutation 返回的类型取决于你，但我们推荐专门定义一些回应类型来保证返回合适的结果给客户端。在大型项目中，你可能会把它抽象在一个接口（interface）类型中，但现在我们就直接定义 `TripUpdateResponse` 了：

_src/schema.js_

```graphql
type TripUpdateResponse {
  success: Boolean!
  message: String
  launches: [Launch]
}
```

我们的 mutation 回应类型包括一个 `success` 字段表示是否成功， 一个 `message` 字段表示一些额外的信息，一个 `launches` 字段表示我们更新的火箭发射。通常来说，你总应该返回你更新的内容，这样的话 Apollo 客户端就可以把更新过的内容自动地缓存下来。

## 让服务器跑起来

现在我们已经确定了 schema，我们调用 `server.listen()` 来让服务器跑起来。

_src/index.js_

```js
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');

const server = new ApolloServer({ typeDefs });

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

在你的终端执行 `npm start` 来启动你的服务器！🎉 Apollo 服务器现在工作在4000端口上了。

### 浏览你的 schema

默认情况下，Apollo 服务器支持 [GraphQL Playground](https://www.apollographql.com/docs/apollo-server/features/graphql-playground/)。Playground 是一个交互式的可以在浏览器运行的 GraphQL IDE，用来浏览的你的 schema 并测试你的请求。Apollo 服务器在开发模式下会自动地提供 Playground。

GraphQL Playground 让你可以重新审视你的 schema。点击右侧的 `schema` 按钮试试看。

![Schema 按钮](../assets/schematab.png)

你可以通过 `schema` 按钮快速访问到一个 GraphQL API 的细节。

You can quickly have access to the documentation of a GraphQL API via the `schema` button.

![Schema 细节](../assets/moredetailsonatype.png)

以上就是编写 schema 的全部内容了。让我们继续教程的下一章节。