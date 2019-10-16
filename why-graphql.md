---
title: Why GraphQL?
description: 了解为什么采用GraphQL以及它如何帮助你更快地交付新功能
---

# 为什么使用GraphQL

在现代的应用程序中，管理数据越来越成为一个挑战。开发者有时需要从不同的数据源聚合数据，有时又需要将数据分发到不同的平台，同时还要想办法把这些数据显示到界面上。对于前端开发者来说，还要决定如何管理状态（state）。与此同时，开发者还需要考虑诸如缓存（caching）和乐观UI（Optimistic UI）等等复杂的需求。

人生苦短，我用GraphQL。下面，让我们来看看GraphQL声明式的获取数据方式是如何简单且高效的吧，同时你还能学习到Apollo平台完整的工具链。

## 开发体验

使用Apollo平台能帮你更快的发布新功能。我们的首要目标是简单化跨越不同端的数据管理。一些通常认为比较困难的功能，比如说全栈缓存，数据正则化和乐观UI在有了Apollo之后变得如此简单。让我们来看看如何做到吧！

## 浏览API

得益于GraphQL的强类型系统，开发者可以在开发过程中实时浏览一个GraphQL的schema的信息，包括它支持哪些query请求。同时，自动文档化，自动补齐也变得可能。

#### GraphQL Playground

Prisma开发的[GraphQL Playground](https://github.com/prismagraphql/graphql-playground)是一个很棒的IDE，它可以自动生成文档，也可以用来执行schema的query，你可以在上面调试你的GraphQL API。你可以认为它有点像Postman之类测试API的工具，但更强大。

![GraphQL Playground](../assets/graphql-playground.png)

Apollo Server 2以上的版本默认开启了GraphQL Playground，所以你无须做任何事就可以马上试用。

#### Apollo调试工具

Apollo调试工具是一个Chrome插件，它让你可以查看Apollo客户端的缓存，跟踪query和查看mutation。你也可以访问Apollo调试工具中的GraphiQL，这让你很方便地在使用了Apollo客户端的前端代码里面调试query。

![Apollo DevTools](../assets/dev-tools.png)

### 简化前端代码

如果你曾经使用REST和一些状态管理的库（比如Redux），你应该习惯于编写action creator、reducer，正则化数据，集成中间件来完成一个简单的网络请求。但使用Apollo客户端，你将不在须要担心这些东东了！Apollo客户端帮你设置好了你需要的一切，你可以直接开始编写业务代码而不是先写个几千行的样板代码了。

```javascript
import ApolloClient from 'apollo-boost';

const client = new ApolloClient({
  uri: 'https://dog-graphql-api.glitch.me/graphql'
});
```

那些切换到Apollo客户端的团队纷纷表示删了数千行的样板代码（[看这里](https://blog.apollographql.com/reducing-our-redux-code-with-react-apollo-5091b9de9c2a)），同时极大地降低了代码复杂度。Apollo客户端可以同时搞定本地和远端数据，你可以用Apollo缓存作为单一可信数据源来存储应用的全局状态。

### 现代化的工具

使用Apollo平台来开发GraphQL API让你可以使用现代化的工具，更快发现bug，可视化API和更有信息开发一些挑战性的功能（例如缓存）。

[Apollo Graph Manager](https://engine.apollographql.com/login)是GraphQL生态中唯一一个提供监视和分析API功能的工具。(译者：这是一个收费的商业产品，有兴趣的可以看看，译者没有使用过，所以相关章节暂无翻译)

![Apollo Graph Manager](../assets/engine.png)

## 声明式地获取数据

GraphQL的一个主要的优势就在于其声明式的数据获取。使用GraphQL，你不在须要去若干个终端获取数据，再手动地聚合这些数据（我们使用REST时的常规操作）。相反的，你声明你想要什么样的数据GraphQL就会给你你想要的。

使用REST时，你可能会分别地调用下面这些接口来获取每一个列表里面的每一项，过滤出你要的数据，然后把剩下的数据聚合起来，以满足你的视图组件的需求。

```bash
GET /api/dogs/breeds
GET /api/dogs/images
GET /api/dogs/activities
```

这样的处理方式不单更耗时、更加容易出错，同时在跨平台的产品中也不大容易逻辑复用。对比下GraphQL的声明式数据获取：

```graphql
const GET_DOGS = gql`
  query {
    dogs {
      id
      breed
      image {
        url
      }
      activities {
        name
      }
    }
  }
`;
```

在这里，我们声明我们从服务器获取的数据应该长什么样。GraphQL会帮我们搞定数据的过滤和聚合，返回的就是你想要的数据结构。

那我们在应用中怎么使用声明式的数据获取呢？在React应用中，所有获取数据、跟踪加载状态和错误状态和更新UI都包装在一个简单的`useQuery` hook中了。这让你将数据获取和视图组件的结合变得非常容易。看看下面这个例子：

```javascript
function Feed() {
  const { loading, error, data } = useQuery(GET_DOGS);

  if (error) return <Error />;
  if (loading || !data) return <Fetching />;

  return <DogList dogs={data.dogs} />;
}
```

Apollo客户端搞定一个请求从开始到结束的方方面面，包括追踪加载状态和错误状态。不需要设置任何中间件和写任何样板代码，也不要担心数据转换和缓存。你要做的就是声明你的组件想要什么样的数据，剩下的苦活累活就交给Apollo客户端了。给力💪！

你会发现，当你切换到Apollo客户端后，你会删除一大坨不需要的数据管理的代码。至于能删多少，取决于你自己的应用，不过有一些团队说删了几千行（编者：这个故事说了好几遍还行）。想知道怎么样用更少的代码实现乐观UI、刷新数据、分页等等高级特性，就去看看Apollo客户端的文档吧。

## 性能提升

在许多情况下，仅仅在现存的REST API上包装一层GraphQL API就可以提高性能，特别是当设备的网络条件不好时。但你这么做之前，应该先评估这么做的影响，通常来说，GraphQL会减少请求的数量和体积。

### 体积更小

因为从服务器返回的回应只包含你所需要的字段，GraphQL相对REST显著地减小了交互的体积。让我们再看看前面小狗的例子：

```graphql
const GET_DOGS = gql`
  query {
    dogs {
      id
      breed
      image {
        url
      }
      activities {
        name
      }
    }
  }
`;
```

从服务器返回的回应一个包含了`id`、 `breed`、 `image`和`activities`字段的dog对象的列表。即时我们的resolver调用的底层的REST接口返回的对象包括100个字段也没关系，那些不需要的字段在返回客户端之前就被过滤掉了。

### 避免多次请求

因为一个GraphQL请求只有一个交互，切换到GraphQL能帮你避免多次交互的开销。当我们使用REST时，每一个资源代表一次交互，这样就可以很方便都添加新的资源。这样的话，当你请求一系列的资源时，不得不为每一个资源做一次交互，这导致加载变慢，尤其是在移动端。

```bash
GET /api/dogs/breeds
GET /api/dogs/images
GET /api/dogs/activities
```

当使用GraphQL时，每个query代表一次单独的交互。如果你想更进一步地减少交互，你可是实现[query batching](https://www.apollographql.com/docs/react/advanced/network-layer#query-batching)以在一个求情中批量处理多个query。

### 产品化

Fackbook在2015年第一次公布GraphQL，在此之前的2012年GraphQL就已经是他们移动端应用的关键组件了。

对Apollo来说，我们发现GraphQL对我们遇到的很多现有技术所难以解决的问题都是一个很棒的解决方案，现在更被用来增强底层架构。在过去的几年，我们和开源社区、客户和合作伙伴一起工作，持续给开源的Apollo带来改进，我们很自豪地说我们提供的方案对无论是刚起步的项目还是大规模部署的项目都同样适用。

除了我们自己的经验，我们还收到了Apollo GraphQL使用者的很多反馈、贡献和支持。下面是其中一些（都是英文的）：

* [**纽约时报**](https://open.nytimes.com/the-new-york-times-now-on-apollo-b9a78a5038c)
* [**爱彼迎**](https://medium.com/airbnb-engineering/reconciling-graphql-and-thrift-at-airbnb-a97e8d290712)
* [**Express**](https://dev-blog.apollodata.com/changing-the-architecture-of-express-com-23c950d43323)
* [**美足联**](https://dev-blog.apollodata.com/reducing-our-redux-code-with-react-apollo-5091b9de9c2a)
* [**Expo**](https://dev-blog.apollodata.com/using-graphql-apollo-at-expo-4c1f21f0f115)
* [**KLM**](https://youtu.be/T2njjXHdKqw)

