```json
{
  "date": "2023.01.15 18:17",
  "tags": ["Golang", "微服务"],
  "author": "XinceChan",
  "musicId": "1959547786"
}
```

维基上微服务定义为：一种软件开发技术- 面向服务的体系结构（SOA）架构样式的一种变体，它提倡将单一应用程序划分成一组小的服务，服务之间互相协调、互相配合，为用户提供最终价值。每个服务运行在其独立的进程中，服务与服务间采用轻量级的通信机制互相沟通（通常是基于HTTP的RESTful API）。每个服务都围绕着具体业务进行构建，并且能够独立地部署到生产环境、类生产环境等。另外，应尽量避免统一的、集中式的服务管理机制，对具体的一个服务而言，应根据上下文，选择合适的语言、工具对其进行构建。

一直想了解一下微服务究竟是什么东西，该如何去实现。刚才在YouTube上看到关注的youtuber有相关介绍golang微服务的视频，之前看了一点简单的做了了解。这次跟着视频做一个建议的项目看看能不能实现这个内容。

> Monolith to Microservice - A Golang Projects Series!
>
> 这个章节将分为几个章节，用来展示这个项目的核心思路，部分代码等。

### MONOLITH TO MICROSERVICE

#### 系统架构

![archi](../../assets/images/monolith-archi.png)

#### 微服务架构

![image-20230115212557982](../../assets/images/archi.png)

#### 对应端口

![image-20230115213513436](../../assets/images/ports.png)

#### 文件架构

```json
├── cmd
│   ├── microservices
│   │   ├── orders
│   │   │   └── main.go
│   │   ├── payments
│   │   │   └── main.go
│   │   └── shop
│   │       └── main.go
│   └── monolith
│       └── main.go
├── docker
├── pkg
│   ├── common
│   │   ├── cmd
│   │   │   ├── router.go
│   │   │   ├── signals.go
│   │   │   └── wait.go
│   │   ├── http
│   │   │   └── error.go
│   │   └── price
│   │       ├── price.go
│   │       └── price_test.go
│   ├── orders
│   │   ├── application
│   │   │   └── orders.go
│   │   ├── domain
│   │   │   └── orders
│   │   │       ├── address.go
│   │   │       ├── address_test.go
│   │   │       ├── order.go
│   │   │       ├── order_test.go
│   │   │       ├── product.go
│   │   │       └── respository.go
│   │   ├── infrastructure
│   │   └── interfaces
│   ├── payments
│   │   ├── application
│   │   ├── infrastructure
│   │   └── interfaces
│   └── shop
│       ├── application
│       ├── domain
│       ├── fixtures.go
│       ├── infrastructure
│       └── interfaces
├── tests
│   └── acceptance_test.go
├── Dockerfile
├── Makefile
├── docker-compose.yml
└── go.work
```

