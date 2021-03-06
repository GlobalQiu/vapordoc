# Services

服务是 Vapor 的 *依赖注入* (也叫 *控制反转*) 框架, 服务框架允许您注册, 配置和初始化任何必要的东西.

## Container

你与服务的大部分交互都经由一个容器, 这个容器包括:

- [Services](#services): 已注册服务的集合.
- [Config](#config): 声明某些服务优先于其他服务.
- [Environment](#environment): 应用程序当前环境类型 (测试环境, 生产环境等).
- [Worker](async.md#event-loop): 当前容器关联的事件循环.

在 Vapor 中最常见的容器是:

- `Application`
- `Request`
- `Response`


你应当将 `Application ` 作为容器创建在启动应用的时候所需的服务, 将 `Request` 或者 `Response` 容器创建响应请求的服务 (在 route 闭包或 controllers 中).

### Make

创建服务很简单, 只要使用容器调用 `.make(_:)` 函数, 然后传递你需要的类型 (通常是 `Client` 这样的协议类型).

```swift
let client = try req.make(Client.self)
```

如果你确切地知道你所需要的, 你可以定义一个具体类型.

```swift
let leaf = try req.make(LeafRenderer.self)
print(leaf) /// Definitely a LeafRenderer

let view = try req.make(ViewRenderer.self)
print(view) /// ViewRenderer, might be a LeafRenderer
```

>tip
>
>如果可以的话, 尝试依赖具体类型的协议, 这会让你的代码更容易被测试 (你可以轻松地替换功能), 并且为你的代码解耦.

## Services

`Services` 结构体包含所有你&mdash;或者你添加过的协议提供者的&mdash;已注册的所有服务, 我们通常在 [`configure.swift`](structure.md#configureswift) 中注册和配置你的服务. 

### Instance

你可以使用 `.register(_:)` 注册已初始化的服务实例.

```swift
/// Create an in-memory SQLite database
let sqlite = SQLiteDatabase(storage: .memory)

/// Register to sevices.
services.register(sqlite)
```

在你服务注册之后, 它可以供 `Container` 创建.

```swift
let db = app.make(SQLiteDatabase.self)
print(db) // SQLiteDatabase (the one we registered earlier)
```

### Protocol

再注册服务同时, 你也可以对特定协议声明一致性, 你可能发现 Vapor 是怎样注册主路由的了.

```swift
/// Register routes to the router
let router = EngineRouter.default()
try routes(router)
services.register(router, as: Router.self)
```

我们使用 `as: Router.self` 注册 `router`, 它也可以由其他具体类型或协议创建.

```swift
let router = app.make(Router.self)
let engineRouter = app.make(EngineRouter.self)
print(router) // Router (actually EngineRouter)
print(engineRouter) // EngineRouter
print(router === engineRouter) // true
```

## Environment

环境是用来在具体情境下动态改变 Vapor 程序的行为方式. 比如说, 在你部署应用程序时为数据库使用不同的用户名和密码, 这时 `Environment` 类型就让管理变得容易.

从命令行工具运行 Vapor 程序时, 你可以传递一个可选的 `env` 标志来指定环境. 环境类型默认为 `.development`.

```sh
swift run Run --env prod
```

在上面的示例代码中, 我们在 `.production(生产)` 环境中运行 Vapor, 这个环境的定义是 `isRelease = true`.

你可以使用传递给 [`configure.swift`](structure.md#configureswift) 的环境来动态注册服务.

```swift
let sqlite: SQLiteDatabase
if env.isRelease {
    /// Create file-based SQLite db using $SQLITE_PATH from process env
    sqlite = try SQLiteDatabase(storage: .file(path: Environment.get("SQLITE_PATH")!))
} else {
    /// Create an in-memory SQLite database
    sqlite = try SQLiteDatabase(storage: .memory)
}
services.register(sqlite)
```

>info
>
>使用静态方法 `Environment.get(_:)` 从进程环境获取字符串值.

你也可以使用工厂方法`.register(_:)` 基于环境动态注册服务.

```swift
services.register { container -> BCryptConfig in
  let cost: Int
  
  switch container.environment {
  case .production: cost = 12
  default: cost = 4
  }
  
  return BCryptConfig(cost: cost)
}
```

## Config

如果给定协议有多种服务可用, 你需要使用 `Config` 结构体来声明你倾向的服务.

```sh
ServiceError.ambiguity: Please choose which KeyedCache you prefer, multiple are available: MemoryKeyedCache, FluentCache<SQLiteDatabase>.
```

这也在 `configure.swift`](structure.md#configureswift) 中实现, 只需要使用 `config.prefer(_:for:)` 方法.

```swift
/// Declare preference for MemoryKeyedCache anytime a container is asked to create a KeyedCache
config.prefer(MemoryKeyedCache.self, for: KeyedCache.self)

/// ...

/// Create a KeyedCache using the Request container
let cache = req.make(KeyedCache.self)
print(cache is MemoryKeyedCache) // true
```