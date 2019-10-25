---
title: '2. Hook up your data sources'
description: 将 REST 和数据库数据连接到 graph
---

预计阅读时间: _10 分钟_

我们已经建好 schema 了，现在应该把数据源和 GraphQL API 接上了。GraphQL API是很灵活的，因为你可以把它放在所有服务的最上层，包括业务逻辑、REST API、数据库和 gRPC 服务。

Apollo 提供了一组数据源 API 让你很容易地就可以把这些服务连接到 graph。**Apollo data source** 是一个将数据获取逻辑隐藏在内部的类，还提供缓存和备份。用 Apollo data source 来连接 graph API 和数据，你同时也就符合了组织代码的最佳实践。

在下一小节，我们将实现 REST API 和 SQL 数据库的数据源。如果你还不熟悉这两个技术，别担心，你不须要理解它们的细节就可以跟上。😀

## 连接 REST API

首先，让我们把 [Space-X v2 REST API](https://github.com/r-spacex/SpaceX-API) 连到我们的 graph。在那之前，我们先安装一下依赖：

```bash
npm install apollo-datasource-rest --save
```

这个包暴露了 `RESTDataSource` 类，这是一个从 REST API 获取数据的类。为 REST API 构造一个数据源，让我们继承这个类并定义 `this.baseURL`；

在我们的例子中，`baseURL` 是 `https://api.spacexdata.com/v2/`。让我们添加下面的代码来实现 `LaunchAPI` 数据源：

_src/datasources/launch.js_

```js
const { RESTDataSource } = require('apollo-datasource-rest');

class LaunchAPI extends RESTDataSource {
  constructor() {
    super();
    this.baseURL = 'https://api.spacexdata.com/v2/';
  }
}

module.exports = LaunchAPI;
```

`RESTDataSource` 同时会自动缓存所有从 REST 数据源来的回应而不需要额外的设置。这样做的好处是你可以复用你的 REST API 暴露的缓存逻辑。看[这篇博客](https://blog.apollographql.com/easy-and-performant-graphql-over-rest-e02796993b2b)了解更多。

### 编写数据获取方法

下一步， 向 `LaunchAPI` 中添加方法让我们的 graph API 获取数据的方法。根据我们之前定的 schema，我们需要一个方法获取所有的火箭发射，所以让我们向 `LaunchAPI` 类增加一个 `getAllLaunches` 方法：

_src/datasources/launch.js_

```js
async getAllLaunches() {
  const response = await this.get('launches');
  return Array.isArray(response)
    ? response.map(launch => this.launchReducer(launch))
    : [];
}
```

`RESTDataSource` 有对应 HTTP `GET` 和 `POST` 的帮助函数。在上面的例子中，`this.get('launches')` 向 `https://api.spacexdata.com/v2/launches` 做了一次 `GET` 请求并把结果存到了 `response` 变量中。然后 `getAllLaunches` 方法调用 `this.launchReducer` 将回应映射成火箭发射 `launch` 的数组，如果没有任何发射，则返回一个空数组。

现在，我们需要写 `launchReducer` 方法，它将把火箭发射数据转换成我们的 schema 期望的样子。我们推荐用这种方式来将你的 graph API 和业务逻辑解耦。让我们看看 schema 里面 `Launch` 现在长什么样子：

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
下面，让我们写一个 `launchReducer` 函数来做数据转换。把下面的代码拷贝到 `LaunchAPI` 类中：

_src/datasources/launch.js_

```js
launchReducer(launch) {
  return {
    id: launch.flight_number || 0,
    cursor: `${launch.launch_date_unix}`,
    site: launch.launch_site && launch.launch_site.site_name,
    mission: {
      name: launch.mission_name,
      missionPatchSmall: launch.links.mission_patch_small,
      missionPatchLarge: launch.links.mission_patch,
    },
    rocket: {
      id: launch.rocket.rocket_id,
      name: launch.rocket.rocket_name,
      type: launch.rocket.rocket_type,
    },
  };
}
```

有了上面这些代码，我们就很容易修改 `launchReducer` 方法，同时保证 `getAllLaunches` 方法简单明了。`launchReducer` 方法同时让 `LaunchAPI` 类的测试更简单，这个稍后会详谈。

接下来，让我们搞定通过 ID 来获取特定的火箭发射。向 `LaunchAPI` 类增加 `getLaunchById` 和 `getLaunchesByIds` 方法：

_src/datasources/launch.js_

```js
async getLaunchById({ launchId }) {
  const response = await this.get('launches', { flight_number: launchId });
  return this.launchReducer(response[0]);
}

getLaunchesByIds({ launchIds }) {
  return Promise.all(
    launchIds.map(launchId => this.getLaunchById({ launchId })),
  );
}
```

`getLaunchById` 接受一个班次号 `flight_number` 作为入参并返回一个特定的火箭发射 `launch`，`getLaunchesByIds` 接受一组班次号返回一组发射。

现在，我们已经成功连接了 REST API，让我们连接数据库吧！

## 连接数据库

我们刚刚使用的 REST API 是只读的，所以我们需要将 graph API 连接到数据库来保存和获取用户数据。本教程使用 SQLite 作为我们的 SQL 数据库。`package.json` 已经包含了所需的包了，所以它们在教程的刚开始 `npm install` 这步被安装好了。同时，这个小节包含一些 SQL 相关的代码，但这些代码并不是理解 Apollo 数据源所必备的，所以我们已经在 `src/datasources/user.js` 实现了一个 `UserAPI` 数据源。现在打开这个文件，我们将解释一些总体的概念。

### 编写自定义的数据源

Apollo并不是必须支持 SQL 数据源（当然如果你有兴趣贡献代码的话，我们很愿意指导你如何做），所以我们需要通过继承 Apollo 数据源基类为我们的数据库创建一个自定义的数据源。你可以用 `apollo-datasource` 包来创建你自己的。

下面是一些创建你自己的数据源的核心概念：

- `initialize` 方法： 如果你想传一些配置参数到你的类里，你需要实现这个方法。这里，我们使用这个方法来访问我们 graph API的上下文（context）。
- `this.context`： Graph API 上下文就是所有 GraphQL 请求的 resolver 共享的一个对象。我们会在下一小节有更详细的解释，现在你须要知道的是上下文很重要，它存储了了用户信息。
- 缓存：使用 REST 数据源的时候它有内置的缓存，然而一般的数据源并没有。但你可以用[这里](https://www.apollographql.com/docs/apollo-server/features/data-sources/#using-memcached-redis-as-a-cache-storage-backend)提到的方式来自己实现。

看我们看看我们在 `src/datasources/user.js` 中创建的一些从数据库获取和更新数据的方法，在下一章节碰到它们的时候可以做个参考：

- `findOrCreateUser({ email })`: 通过 `email` 来查找或创建一个用户
- `bookTrips({ launchIds })`: 接受一个包含火箭发射版次号的数组的对象，然后为登录用户预定这些班次
- `cancelTrip({ launchId })`: 接受一个包含火箭发射版次号的数组的对象，然后为登录用户取消这些班次
- `getLaunchIdsByUser()`: 返回登录用的所有以预定火箭发射
- `isBookedOnLaunch({ launchId })`: 返回登录用户是否预定了某个特定的火箭班次

## 将数据源添加到 Apollo 服务器

现在我们实现了两个数据源，`LaunchAPI` 连接到 REST API，`UserAPI` 连接到 SQL 数据库。现在我们须要把它们添加到 graph API。

添加数据源很简单，在 `ApolloServer` 的构造函数入参中添加一个 `dataSources` 字段，这个字段是一个函数，它返回所有用到的数据源的集合对象。让我们看看怎么写吧，打开 `src/index.js` 文件，然后添加下面的代码：

_src/index.js_

```js{3,5,6,8,12-15}
const { ApolloServer } = require('apollo-server');
const typeDefs = require('./schema');
const { createStore } = require('./utils');

const LaunchAPI = require('./datasources/launch');
const UserAPI = require('./datasources/user');

const store = createStore();

const server = new ApolloServer({
  typeDefs,
  dataSources: () => ({
    launchAPI: new LaunchAPI(),
    userAPI: new UserAPI({ store })
  })
});

server.listen().then(({ url }) => {
  console.log(`🚀 Server ready at ${url}`);
});
```

首先，我们引入 `createStore` 函数来设置数据库，同时我们也引入了数据源 `LaunchAPI` 和 `UserAPI`。然后，我们调用 `createStore` 来创建数据库。最后我们添加 `dataSources` 到 `ApolloServer`，这样 `LaunchAPI` 和 `UserAPI` 就连接到我们的 graph了。我们同时也把创建的数据库传入 `UserAPI` 数据源。

如果你在数据看中使用了 `this.context`，很重要的一点是在 `dataSources` 函数中创建的新的实例而不是共享一个实例，否则的话 `initialize` 方法在一些异步代码中被调用的时候，`this.context` 可能会被另外一个用户的上下文替换。

现在我们把自己的数据源连接到 Apollo 服务器上了，是时候进行下一章节来学习怎么样在 resolver 中调用我们的数据源了。