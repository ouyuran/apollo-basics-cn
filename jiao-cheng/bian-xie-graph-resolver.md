---
title: "3. Write your graph's resolvers"
description: 学习 GraphQL query 如何获取数据
---

预计阅读时间: _15 分钟_

到目前为止，我们的 graph API 还不能工作。我们可以查看我们定义的 graph schema，但我们还不能用它真正地跑一个 query。既然我们已经编写好了 schema 和数据源，是时候展现它们的用处了。在我们的 graph API resolver 函数中调用数据源来触发业务逻辑、获取/更新数据吧。

## Resolver是什么？

**Resolver** 是将 GraphQL 操作（例如 query、mutation 或 subscription）转换为数据的操作指南。它可以返回 schema 中定义的数据结构或是一个那些数据结构的 promise。

在我们开始编写 resolver 之前，我们需要先看看 resolver 函数长什么样子。Resolver 函数接受四个入参：

```js
fieldName: (parent, args, context, info) => data;
```

- **parent**: 父类型的 resolver 返回结果的对象
- **args**: 包含所有入参的对象
- **context**: 所有 GraphQL 操作的 resolver 共享的对象，我们使用 context 来保存之前请求的状态（例如鉴权信息），我们还用它来访问数据源
- **info**: 操作的执行状态信息，仅在高阶用例中使用

还记得我们在前一章节实现的 `LaunchAPI` 和 `UserAPI`吗？我们把他们传入了 `ApolloServer` 的 `context` 属性。我们准备通过 `context` 入参在 resolver 中调用他们了。

这可能一开始听起来有点乱，但仔细看一个具体的例子就会明白了。让我们开始吧！

### 将 resolver 连接到 Apollo 服务器

首先，让我们把我们的 resolver 连接到 Apollo 服务器。目前为止，它还是一个空对象，但我们还是先把它加到 `ApolloServer` 实例中以免待会忘记。打开 `src/index.js` 文件添加以下代码：

_src/index.js_
```js
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');
const { createStore } = require('./utils');
const resolvers = require('./resolvers'); // highlight-line

const LaunchAPI = require('./datasources/launch');
const UserAPI = require('./datasources/user');

const store = createStore();

const server = new ApolloServer({
  typeDefs,
  resolvers, // highlight-line
  dataSources: () => ({
    launchAPI: new LaunchAPI(),
    userAPI: new UserAPI({ store })
  })
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

Apollo 服务器会自动地将 `launchAPI` 和 `userAPI` 添加到 resolver 的上下文（context）以便我们能调用它们。

## 编写 Query resolver

首先，让我们开始为 `Query` 类型中的 `launches`、`launch` 和 `me` 字段编写 resolver 吧。我们的 resolver 是一个 map，键值代表 schema 中定义的字段。如果你不记得一个类型中有哪些字段了，就回去看看 schema 的定义吧。

打开 `src/resolvers.js` 并添加以下代码：

```js
module.exports = {
  Query: {
    launches: (_, __, { dataSources }) =>
      dataSources.launchAPI.getAllLaunches(),
    launch: (_, { id }, { dataSources }) =>
      dataSources.launchAPI.getLaunchById({ launchId: id }),
    me: (_, __, { dataSources }) => dataSources.userAPI.findOrCreateUser()
  }
};
```

上面这段代码是 `launches`、`launch` 和 `me` 的 resolver 函数。顶级 resolver 的第一个入参 `parent` 永远是空，因为它指向我们 graph 的根（root）。第二个入参指向任何传入 query 的入参,在 `launch` query 中我们传入的是火箭发射的班次号。最后我们从第三个入参 `context` 中解构出数据源，这样我们就能在 resolver 中调用他们了。（译者：解构同样为ES6语法）

有没有发现我们的 resolver 写得简单明了，那是因为复杂逻辑都被包装到 `LaunchAPI` 和 `UserAPI` 这两个数据源中了。保持 resolver 轻量化是我们推荐的最佳实践，这样你就能安全地重构代码而不用担心破坏了你的 API 了。

### 在 playground 中运行 query

Apollo 服务器提供了 GraphQL Playground 让你运行 query 和查看 schema。用 `npm start` 命令启动你的服务器，然后用浏览器访问 `http://localhost:4000/` 来打开 playground。

将下面的 GraphQL query 拷贝到 playground 的左侧，然后中间的点击 play 按钮来获得结果。

```graphql
query GetLaunches {
  launches {
    id
    mission {
      name
    }
  }
}
```

当你编写一个 GraphQL query 时，你总是应该用**操作关键字**（例如 query 或 mutation）和它的名字（例如 `GetLaunches`）开始。给你的 query 起一个表义的名字很重要，这样你在 Apollo 开发者工具中能更容易地找到它。下面，在名字后面我们用一对大括号把 query 的 body 部分括起来。接下来，我们指定 `launches` 字段（它定义在 `Query` 类型中），然后再跟上一对大括号把**选择集合**括起来。选择集合表示我们期望 query 的回应包含哪些字段。

GraphQL 很棒的一点就是 query 想要什么，回应就会给什么。试着在 query 中添加或删除一些字段，看看回应的内容会发生什么变化。

现在，让我们编写一个 query，它接受一个入参，用来获取单个火箭发射。把下面的 query 拷贝到 playground 中并点击 play 运行。

```graphql
query GetLaunchById {
  launch(id: 60) {
    id
    rocket {
      id
      type
    }
  }
}
```

注意到上面的入参 `60` 是硬编码在 query 中的，你可以将它改写成下面这样，然后在 playground 的左下角填写参数，这样就可以用不同的参数跑同一个 query 了：

```graphql
query GetLaunchById($id: ID!) {
  launch(id: $id) {
    id
    rocket {
      id
      type
    }
  }
}
```

试着将 `{ "id": 60 }` 拷贝到左下角的变量区然后再运行一次你的 query，在进行下一小节之前多试试这个功能吧。

### 分页 query

直接运行 `launches` query 返回的数据量很大，这可能拖累我们的应用。我们怎么保证一次获取的数据量不会太大呢？

**分页（pagination）**是解决这个问题的一种方案，它保证了服务器每次只发送一小片数据。基于游标的分页（Curson-based pagination）是我们推荐的做法，因为这种做法既不会漏掉一些条目也不会将同一个条目显示多次。在基于游标的分页中，一个常量指针（cursor）用来追踪接下来该从哪里开始获取数据。

下面，让我们试试基于游标的分页。打开 `src/schema.js` 文件，更新 `Query` 类型中的 `launches` 并增加一个新的 `LaunchConnection` 类型：

_src/schema.js_

```graphql
type Query {
  launches( # 将当前的 launches query 替换为这个。
    """
    默认的结果个数，应该 >= 1。默认值 20。
    """
    pageSize: Int
    """
    如果在这里添加了游标，那么之后游标之后的结果会返回。
    """
    after: String
  ): LaunchConnection!
  launch(id: ID!): Launch
  me: User
}

"""
对 launches 结果的简单包装，包含一个游标 cursor 指向当
前结果的最后一项，hasMore 标示是否有更多的结果了。
将 cursor 传入下一次的 launches query 来获取之后的结果。
"""
type LaunchConnection { # 将这个额外的类型添加在 Query 类型之后。
  cursor: String!
  hasMore: Boolean!
  launches: [Launch]!
}
...
```

你可能注意到了，我们使用一组 `"""` 来在 shcema 中引入多行注释。现在 `launches` query 接受两个入参： `pageSize` 和 `after`，返回 `LaunchConnection`。`LaunchConnection` 类型包括 launches 的一个列表，一个 `cursor` 标示我们当前获取到全部结果的哪里了，一个 `hasMore` 标示是否还有更多的结果。

打开  `src/utils.js` 文件看看 `paginateResults` 函数。这个函数是在服务器侧将数据分页的帮助函数。现在，让我们更新必要的 resolver 函数来完成分页功能。

让我们引入 `paginateResults` 并将 `src/resolvers.js` 文件中的 `launches` resolver 函数替换为下面的代码：

_src/resolvers.js_

```js{1,5-26}
const { paginateResults } = require('./utils');

module.exports = {
  Query: {
    launches: async (_, { pageSize = 20, after }, { dataSources }) => {
      const allLaunches = await dataSources.launchAPI.getAllLaunches();
      // we want these in reverse chronological order
      allLaunches.reverse();

      const launches = paginateResults({
        after,
        pageSize,
        results: allLaunches
      });

      return {
        launches,
        cursor: launches.length ? launches[launches.length - 1].cursor : null,
        // if the cursor of the end of the paginated results is the same as the
        // last item in _all_ results, then there are no more results after this
        hasMore: launches.length
          ? launches[launches.length - 1].cursor !==
            allLaunches[allLaunches.length - 1].cursor
          : false
      };
    },
    launch: (_, { id }, { dataSources }) =>
      dataSources.launchAPI.getLaunchById({ launchId: id }),
     me: async (_, __, { dataSources }) =>
      dataSources.userAPI.findOrCreateUser(),
  }
};
```

让我们测试一下分页功能。先杀掉服务器程序，然后用 `npm start` 重新启动，在 playground 上运行下面这个 query：

```graphql
query GetLaunches {
  launches(pageSize: 3) {
    launches {
      id
      mission {
        name
      }
    }
  }
}
```

分页功能生效了，你现在应该只能看到三个结果了。

## 为你的类型编写 resolver

值得注意的是，你可以为定义在 schema 中的任何类型编写 resolver，不仅仅是 query 和 mutation。这让 GraphQL 非常灵活。

你可能发现我们还没有给任何一个类型编写 resolver，但是我们的 query 还是能正确的运行。这是因为 GraphQL 有默认的 resolver，因此如果一个字段在父对象中的属性名和本身的名字相同，那么我们就不比为它编写 resolver。

看个具体的例子，假设说我们打算为 `Mission` 类型编写一个 resolver。打开 `src/resolvers.js` 文件并把下面的 resolver 拷贝到 `Query` 属性下面：

_src/resolvers.js_

```js
Mission: {
  // 设置默认的 size 为 'large'
  missionPatch: (mission, { size } = { size: 'LARGE' }) => {
    return size === 'SMALL'
      ? mission.missionPatchSmall
      : mission.missionPatchLarge;
  },
},
```

_src/schema.js_
```js
  type Mission {
    # ... with rest of schema
    missionPatch(mission: String, size: PatchSize): String
  }
```

传入 resolver 的第一个入参是父对象，指向 mission。第二个入参是我们希望的 `missionPatch` 的 size，用来决定我们会用 mission 对象的哪个字段来填充 `missionPatch` 字段。

我们已经知道怎么在除了 `Query` 和 `Mission` 之外的类型上添加 resolver，让我们再往 `Launch` 和 `User` 类型上添加 resolver 吧。将下面的代码拷贝到对应的位置：

_src/resolvers.js_

```js
Launch: {
  isBooked: async (launch, _, { dataSources }) =>
    dataSources.userAPI.isBookedOnLaunch({ launchId: launch.id }),
},
User: {
  trips: async (_, __, { dataSources }) => {
    // 通过用户获取班次号
    const launchIds = await dataSources.userAPI.getLaunchIdsByUser();

    if (!launchIds.length) return [];

    // 通过班次号查询班次信息
    return (
      dataSources.launchAPI.getLaunchesByIds({
        launchIds,
      }) || []
    );
  },
},
```

你可能想知道，我们怎么获取用户信息，然后才能查询他们的运行班次。我们需要用户鉴权！在继续学习 `Mutatuion` resolver 之前，让我们一些来学习怎么样鉴权用户，并把他们的信息添加到上下文。

## 用户鉴权

访问控制几乎是每一个应用都需要的一个功能。在本教程中，我们将重点放在介绍用户鉴权的一些基本知识而不是某一个具体实现。

下面是我们将要学习的内容：

1. `ApolloServer` 实例中的 context 函数，它将在你的 API 每次收到一个 GraphQL 操作请求的时候被调用。Context 函数可以访问请求对象（request object），使用请求对象来读 authorization 消息头（header）。
2. 在 context 函数中鉴权用户。
3. 一旦用户被鉴权，将用户信息添加到 context 函数的返回对象中。这让我们可以在数据源和 resolver 中读取到用户信息，然后可以判断用户是否有权限访问特定的数据。

打开 `src/index.js` 文件并更新 `ApolloServer` 中的 `context` 函数：

_src/index.js_

```js{1,4,8,10}
const isEmail = require('isemail');

const server = new ApolloServer({
  context: async ({ req }) => {
    // simple auth check on every request
    const auth = req.headers && req.headers.authorization || '';
    const email = Buffer.from(auth, 'base64').toString('ascii');

    if (!isEmail.validate(email)) return { user: null };

    // 通过邮箱查找用户
    const users = await store.users.findOrCreate({ where: { email } });
    const user = users && users[0] || null;

    return { user: { ...user.dataValues } };
  },
  // ....
```

在上面的代码中，我们检查了求情的 authorization 消息头，通过在数据库中查找他们的证书（credential）来做鉴权，并把用户信息添加到了 `context`。当然我们并不建议在真实产品中这样用，因为并不安全。这里只是一个大概的步骤。

我们怎么穿件一个传入  `authorization` 消息头的 token 呢？让我们继续下一小节，编写 `login` mutation 的 resolver。

## 编写 mutation resolver

`Mutation` resolver 的编写和我们已经编写的那些是类似的。首先让我们编写一个 `login` resolver 来完成鉴权过程，在 `Query` resolver 中添加以下代码：

_src/resolvers.js_

```js
Mutation: {
  login: async (_, { email }, { dataSources }) => {
    const user = await dataSources.userAPI.findOrCreateUser({ email });
    if (user) return Buffer.from(email).toString('base64');
  }
},
```

`Login` resolver 接受一个邮箱地址作为入参，如果用户存在则返回一个 token。在接下来的章节，我们会雪灾怎么在客户端保存 token。

现在，让我们为 `Mutation` 添加两个 resolver：`bookTrips` 和 `cancelTrip`。

_src/resolvers.js_

```js
Mutation: {
  bookTrips: async (_, { launchIds }, { dataSources }) => {
    const results = await dataSources.userAPI.bookTrips({ launchIds });
    const launches = await dataSources.launchAPI.getLaunchesByIds({
      launchIds,
    });

    return {
      success: results && results.length === launchIds.length,
      message:
        results.length === launchIds.length
          ? 'trips booked successfully'
          : `the following launches couldn't be booked: ${launchIds.filter(
              id => !results.includes(id),
            )}`,
      launches,
    };
  },
  cancelTrip: async (_, { launchId }, { dataSources }) => {
    const result = await dataSources.userAPI.cancelTrip({ launchId });

    if (!result)
      return {
        success: false,
        message: 'failed to cancel trip',
      };

    const launch = await dataSources.launchAPI.getLaunchById({ launchId });
    return {
      success: true,
      message: 'trip cancelled',
      launches: [launch],
    };
  },
},
```

`BookTrips` 和 `cancelTrips` 都必须返回我们定义在 schema 中的 `TripUpdateResponse` 类型所包含的那些字段，包括 `success` 标示是否成功，`message` 标示状态信息，`launches` 标示我们预定或取消的火箭发射的数组。`BookTrips` mutation 要复杂一点，因为有的发射可以被成功预定，有些可能会失败。现在，我们简单地把部分成功这个状态放在 `message` 字段中。

## 在 playground 中运行 mutation

是时候展现真正地技术了，让我们在 playground 中试试刚刚完成的 mutation！回到浏览器的 playground 界面重新加载一下 shcema。

GraphQL muation 和 query 的结构一样，除了用 `mutation` 关键字。将下面的 muatation 拷贝到 playground 并运行：

```graphql
mutation LoginUser {
  login(email: "daisy@apollographql.com")
}
```

你应该收到像这样的回应：`ZGFpc3lAYXBvbGxvZ3JhcGhxbC5jb20=`。将这个字符串拷贝下来，待会会用到。

现在，让我们试试预定一个旅程。只有鉴权成功的用户才能预定旅程，但幸运的是，playground 有一个区域我们可以自定义消息头，所以我们可以把刚才得到的 token 填进去。将下面的 muation 拷贝到 playground：

```graphql
mutation BookTrips {
  bookTrips(launchIds: [67, 68, 69]) {
    success
    message
    launches {
      id
    }
  }
}
```

接下来，将 `authorization` 消息头拷贝到底部的 HTTP Headers 输入框中：

```json
{
  "authorization": "ZGFpc3lAYXBvbGxvZ3JhcGhxbC5jb20="
}
```

接着运行mutation。你应该能看到一条成功的消息，包括我们刚刚预定的班次号。在 playground 手动测试 mutation 是检查你的 API 行为的一种不错的方式，但在真实的产品中，我们应该使用自动化测试来保证我们重构代码时的安全性。在下一个章节，我们将学习在产品中使用 graph 而不仅仅是在 playground 中测试它们。