---
title: Go CRUD Project Structure
date: 2025-09-13T11:23:29+08:00
cover: /images/face2.jpg
images:
  - /images/face2.jpg
categories:
  - '学习笔记'
tags:
  - Go
  - Gin
# nolastmod: true
# math: true
draft: false
---

关于Go CRUD项目结构的一些思考

<!--more-->

> ⚠️ **【重要免责声明】**
>
> 1. 我在写 [go-gin-starter](https://github.com/precioussilence/go-gin-starter) 的过程中查阅了许多文档和资料，经历了许多思考和取舍，该笔记就是对这些思考和取舍结果的一个记录，一方面重温一下这个思路诞生的心路历程，另一方面回忆本身也是对学习效果的一种强化。
> 1. 该笔记所描述的分层思路带有强烈的个人喜好，萝卜白菜各有所爱，个中取舍未必符合您的胃口，若您不喜姑且一笑置之。
> 1. 受限于个人眼界，这里所列的知识点未必实时和精确，若您有缘看到该笔记，我只能说此内容颇具抛砖引玉之功，并无指点迷津之效。

Springboot项目经常把代码拆分成Controller、Service、Dao等模块，各个模块之间高内聚低耦合，职责清晰、边界明确，彼此紧密协作共同支撑起一个又一个天差地别，甚至千奇百怪的项目。受此启发，我们不禁要想了：Go+Gin项目又该如何分层呢？

## 1. Go的模块管理模式

Go是如何进行模块管理的呢？关于module的创建和管理可以参考以下官方文档：

- [Tutorial: Create a Go module](https://go.dev/doc/tutorial/create-module)
- [Managing module source](https://go.dev/doc/modules/managing-source)
- [Organizing a Go module](https://go.dev/doc/modules/layout)
- [Internal Directories](https://pkg.go.dev/cmd/go#hdr-Internal_Directories)

其中文档一主要描述了如何创建一个module；文档二主要描述了管理一个module仓库需要遵循的规则；文档三主要描述了一个Go项目应该如何组织代码；文档四主要介绍了internal这个包的特别之处。

综合以上四个文档，基本可以对Go的module创建、管理有一个清晰的认识，然后让我们正式走入本笔记的主题：CRUD项目结构！

## 2. Go CRUD项目结构

借鉴Springboot的分层思路和Go官方的推荐配置，按照我个人的理解，一个职责清晰、边界明确的项目结构可以是这样的：

```txt
go-gin-starter/
├── cmd/
│   └── main.go
├── config/
│   └── config.yaml
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── domain/
│   │   ├── user.go
│   │   └── role.go
│   ├── repository/
│   │   ├── repository.go
│   │   ├── user_repository.go
│   │   └── role_repository.go
│   ├── service/
│   │   ├── service.go
│   │   ├── user_service.go
│   │   └── role_service.go
│   ├── handler/
|   |   ├── handler.go
│   │   ├── user_handler.go
│   │   └── role_handler.go
│   └── router/
|       ├── router.go
|       ├── user_router.go
│       └── role_router.go
├── go.mod
└── go.sum
```

> 完整代码在这里 [go-gin-starter](https://github.com/precioussilence/go-gin-starter) ，若是这个代码能够对您有所助益，我将不胜荣幸之至，要是您再动动小手帮我star一下，那我不得快乐得飞起啊！哈哈哈~~~

### 2.1. 分层思路

接下来我们逐步剖析这个项目的结构。项目整体分为三个部分：cmd、config和internal，其中cmd是应用入口，config是全局配置，internal是应用主体。cmd和config比较直观，无需赘言；internal由六个部分组成，各部分功能如下：

- config是应用配置加载入口，一般是选择viper执行加载工作
- domain是业务实体类，比如用户表的Entity定义
- repository是CRUD代码抽象，也就是Spring开发经常提到的Dao，即数据访问层
- service是业务逻辑层，负责维护具体的业务逻辑
- handler是请求处理的入口，可类比Spring开发经常提到的Controller（更确切的说，handler+router相当于Spring的Controller），即控制层
- router是路由绑定配置，负责维护请求路径和处理器的映射

它们的依赖关系如下：

```txt
main.go
  ↓
router
  ↓
handler
  ↓
service
  ↓
repository
  ↓
domain
```

如此划分之后，大体上已能实现类似于Spring的Controller、Service和Dao的分层效果。然而Gin毕竟不是Spring，它本身不提供依赖注入功能，那么代码划分为多层之后，依赖又该如何传递和管理呢？

### 2.2. 依赖管理

[go-gin-starter](https://github.com/precioussilence/go-gin-starter) 主要借助三个组合来管理依赖：

- interface、struct和pointer receiver组合
- 非导出字段、工厂函数和公开方法组合
- 容器和工厂函数组合

让我们来逐个剖析这三个组合

#### 2.2.1. interface、struct和pointer receiver组合

这种 `interface + struct + pointer receiver` 的组合，是Go社区在探索如何构建可测试、可扩展、松散耦合的应用程序的过程中逐步形成的一种约定俗成的最佳实践，那么为什么这种组合脱颖而出成为了事实上的标准了呢？别急，我们先来看一段代码：

```go
package repository

import (
 "context"

 "github.com/precioussilence/go-gin-starter/internal/domain"
 "gorm.io/gorm"
)

type UserRepository interface {
 Create(ctx context.Context, user *domain.User) error
 FindByID(ctx context.Context, id uint) (*domain.User, error)
}

type userRepositoryImpl struct {
 db *gorm.DB
}

func (r *userRepositoryImpl) Create(ctx context.Context, user *domain.User) error {
 return gorm.G[domain.User](r.db).Create(ctx, user)
}

func (r *userRepositoryImpl) FindByID(ctx context.Context, id uint) (*domain.User, error) {
 user, err := gorm.G[domain.User](r.db).Where("id = ?", id).First(ctx)
 if err != nil {
  return nil, err
 }
 return &user, nil
}
```

在上述代码示例中，UserRepository就是interface，userRepositoryImpl是struct，*userRepositoryImpl是pointer receiver。

使用interface的意图很明显，我们通过interface对用户行为进行抽象，上层只依赖interface不关心具体实现，我们可以根据需要选择MySQL、PostgreSQL或是 Mock 实现，这就给扩展和测试带来了极大的便利。

在Go中，所有函数（function）参数都是值传递，也就是说，传进去的任何东西，都会被复制一份。而方法（method）是拥有接收器的特殊函数，其接收器会作为第一个参数隐式传入函数，**因此接收器总是会被拷贝一份传入方法中**，这就会带来两个问题：

- 若接收器是一个特别大的struct，那么拷贝会非常浪费资源和时间
- 若方法需要修改接收器的属性值，那么拷贝会导致永远无法修改原始struct

因此，在大多数场景中pointer receiver是必须的选择，Go标准库中几乎全部使用pointer receiver来实现接口，第三方库也遵循这一惯例。基于这条准则，`go-gin-starter`选择了使用*userRepositoryImpl作为interface的实现者。

至于userRepositoryImpl首字母小写则是因为它不需要被导出，这也是 Go 鼓励的“暴露行为，隐藏实现”准则。

#### 2.2.2. 非导出字段、工厂函数和公开方法组合

这种 `非导出字段 + 工厂函数 + 公开方法` 的组合，是 `Go` 语言中一种典型的、被广泛采用的设计模式，老规矩，先来看一段代码：

```go
package handler

import (
 "net/http"
 "strconv"

 "github.com/gin-gonic/gin"
 "github.com/precioussilence/go-gin-starter/internal/service"
)

type UserHandler struct {
 userService *service.UserService
}

func NewUserHandler(userService *service.UserService) *UserHandler {
 return &UserHandler{
  userService: userService,
 }
}

func (h *UserHandler) GetUserByID(c *gin.Context) {
 id, err := strconv.ParseUint(c.Param("id"), 10, 32)
 if err != nil {
  c.JSON(http.StatusBadRequest, gin.H{"error": "invalid id parameter"})
  return
 }
 user, err := h.userService.GetUserByID(c.Request.Context(), uint(id))
 if err != nil {
  c.JSON(http.StatusNotFound, gin.H{"error": "user not found"})
  return
 }
 c.JSON(http.StatusOK, user)
}
```

在这段示例代码中，userService是非导出字段，NewUserHandler是工厂函数，GetUserByID是公开方法。

`Go` 使用首字母大小写控制标识符的可见性，首字母小写为非导出字段，仅本包内可见。通过定义非导出字段，可以封装实现细节，防止外部包直接修改内部状态；

一个松散耦合的、方便测试的应用往往通过构造函数注入依赖，而非在内部硬编码创建，比如现代Springboot应用就是推荐构造器注入。但是呢 `GO` 没有构造器，所以这里我们用工厂函数模拟构造器。借助工厂函数我们把创建实例的逻辑集中到一处，这也给后续的修改、扩展和Mock带来了极大便利。

公开方法就更好理解了，只开放外部必须的方法，其他全部隐藏。

#### 2.2.3. 容器和工厂函数组合

现在层内已经封装良好，暴露适当了，那么问题来了：如何在层间传递依赖呢？没错！让我们再看一段代码：

```go
package service

import "github.com/precioussilence/go-gin-starter/internal/repository"

type Services struct {
 User *UserService
}

func NewServices(repositories *repository.Repositories) *Services {
 return &Services{
  User: NewUserService(repositories.User),
 }
}
```

在这段代码中，Services是容器，NewServices是工厂函数。工厂函数负责调用创建实例的工厂函数，并把实例添加到容器，上层通过容器获取所需要的实例。

如此一来，我们就可以把所有实例统一管理、统一注入，这种管理模式也给扩展和测试提供了便利。
