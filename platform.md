---
title: The Apollo GraphQL platform
description: 如何使用Apollo从零开始一个项目
---

# Apollo GraphQL平台

Apollo是GraphQL的一个实现，用以构建现代的、数据驱动的应用。它提倡敏捷、增量的方法，特别关注避免引入对任何现有API和服务的改动。Apollo同时特别强调工具和流程。

Apollo的最佳使用场景是作为你的服务和应用的中间层，它是开源组件、商业扩展和云服务的集合。主要部件包括：

![Graph层](../img/platform-diagram.png)

## 开源组件（平台核心部分）

* **Apollo Server**是一个JavaScript GraphQL服务器，它定义 _schema_ 和一组实现这个schema的 _resolvers_ 。一个典型的Apollo Server是可扩展的：可以在请求管道（request pipline）和服务器自己生命周期的任何一步hook一个插件，这使得自定义的行为可以用扩展包的形式来开发。Apollo Server支持AWS Lambda和其他serverless环境。

* **Apollo Client**是一个精巧的GraphQL客户端，管理数据和状态。它的优点包括：提供了一种声明式的编程范式，让开发者可以将query定义为UI组件的一部分；管理query结果和UI的绑定的所有犄角旮旯的细节，包括数据一致性，缓存等；通过 _扩展_ 客户端GraphQL schema引入额外的数据结构，Apollo Client能够非常优雅地管理状态。Apollo Client可以与React、React Native、Vue、Angular或其他视图层框架的集成。

* **iOS and Android**客户端源于社区，支持原生iOS和Android应用。

* **Apollo CLI**是一个简单的命令行客户端，用来访问Apollo的云服务。

## 云服务

* **Schema registry**是一个中心化的schema注册表。
* **Operation registry**是一个schema所有已知操作的注册表。
* **Trace warehouse**是一个数据管道和存储层，它抓取Apollo服务器（或者其它实现了Apollo日志API的服务器）执行每一个GraphQL操作的结构化信息，包括特定field的访问，resolver的调用树（包括时间戳，用户ID，schema版本信息等）。

## 网关

* **Apollo Gateway**是Apollo服务器的配置和提供GraphQL网关功能的插件。网关由一些独立部署但相互引用（reference）的微schema组成，从客户端看来就像是一个正常的GraphQL schema一样。在回应query请求时，网关从每个上游GraphQL服务上获取数据并合并成一个结果返回。

## 工作流程

在所有的组件智商，Apollo实现了一些非常有用的工作流程来管理GraphQL API。这些工作流程保证了平台的不同部分协同工作的舒畅，下面其中一些例子：

### Schema变动有效性检查

Apollo可以检查给定schema和操作的兼容性。这用到了trace warehouse、operation registry和client registry。例如，一个引用了不存在的field的操作或者没有输入必需的参数的操作会引起一个兼容性错误。兼容性检查是一种静态检查，得益于schema的类型定义，它不须要运行在一个服务器上。

### 安全列表

Apollo为客户端和请求提供了一个端到端的 _白名单_ 机制，一个最佳实践是将产品可以使用的GraphQL API预先限定好。这涉及到两部分：第一，Apollo CLI从客户端代码中提取出所有可能的请求组成所有请求的一个子集，保存在operation registry中。第二，Apollo服务器的插件同步这些信息到服务器上，有了这些信息，服务器就可以拒绝所有不在这个子集中的请求。
