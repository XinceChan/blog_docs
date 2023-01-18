```json
{
  "date":"2023.01.18 15:00",
  "author":"XinceChan",
  "tags":["Golang","性能优化"],
  "musicId":"64106"
}
```

本节主要简要介绍了高质量编程的定义和原则，分享了代码格式、注释、命名规范、控制流程、错误和异常处理五方面的常见编码规范。目标主要达成以下四点：如何编写更简洁清晰的代码；常用Go语言程序优化手段；熟悉Go程序性能分析工具；了解工程中性能优化的原则和流程。

## 性能调优工具

> 性能调优原则

- 要依靠数据而不是猜测
- 要定位最大瓶颈而不是细枝末节
- 不要过早优化
- 不要过度优化

### 性能分析工具 pprof

- 希望知道应用在什么地方耗费了多少CPU、Memory
- pprof是用于可视化和分析性能分析数据的工具

![img](../../assets/images/pprof.png)

http://localhost:6060/debug/pprof/

```go
go tool pprof "http://localhost:6060/debug/pprof/profile?seconds=10"
```

- flat：当前函数本身的执行耗时
- flat%：flat占CPU总时间的比例
- sum%：上面每一行的flat%总和
- cum：指当前函数本身加上其调用函数的总耗时
- cum%：cum占CPU总时间的比例

```go
(pprof) top
Showing nodes accounting for 3320ms, 100% of 3320ms total
      flat  flat%   sum%        cum   cum%
    3190ms 96.08% 96.08%     3320ms   100%  github.com/wolfogre/go-pprof-practice/animal/felidae/tiger.(*Tiger).Eat
     130ms  3.92%   100%      130ms  3.92%  runtime.asyncPreempt
         0     0%   100%     3320ms   100%  github.com/wolfogre/go-pprof-practice/animal/felidae/tiger.(*Tiger).Live
         0     0%   100%     3320ms   100%  main.main
         0     0%   100%     3320ms   100%  runtime.main
```

#### 排查 CPU 问题

- 命令行分析

  - ```go
    go tool pprof "http://localhost:6060/debug/pprof/profile?seconds=10"
    ```

- top 命令

- list 命令

- 熟悉 web 页面分析

- 调用关系图，火焰图

- ```go
  go tool pprof -http=:8080 "http://localhost:6060/debug/pprof/cpu"
  ```

#### 排查堆内存问题

- ```go
  go tool pprof -http=:8080 "http://localhost:6060/debug/pprof/heap]"
  ```

#### 排查协程问题

- ```go
  go tool pprof -http=:8080 "http://localhost:6060/debug/pprof/goroutine"
  ```

#### 排查锁问题

- ```go
  go tool pprof -http=:8080 "http://localhost:6060/debug/pprof/mutex"
  ```

#### 排查阻塞问题

- ```go
  go tool pprof -http=:8080 "http://localhost:6060/debug/pprof/block"
  ```

## 性能优化案例

### 业务服务优化

- 服务：能单独部署，承载一定功能的程序
- 依赖：Service A的功能实现依赖Service B的响应结果，称为Service A依赖Service B
- 调用链路：能支持一个接口请求的相关服务集合及其相互之间的依赖关系
- 基础库：公共的工具包、中间件

> 流程

- 建立服务性能评估手段
- 分析性能数据，定位性能瓶颈
- 重点优化项改造
- 优化效果验证

> 建立服务性能评估手段

- 服务性能评估手段
  - 单独 benchmark 无法满足复杂逻辑分析
  - 不同负载情况下性能表现差异
- 请求流量构造
  - 不同请求参数覆盖逻辑不同
  - 线上真实流量情况
- 压测范围
  - 单机器压测
  - 集群压测
- 性能数据采集
  - 单机性能数据
  - 集群性能数据

> 分析性能数据，定位性能瓶颈

- pprof火焰图

> 重点优化项分析

> 进一步优化，服务整体链路分析

- 规范上游服务调用接口，明确场景需求
- 分析链路，通过业务流程优化提升服务性能

### 基础库优化

> AB实验SDK的优化

- 分析基础库核心逻辑和性能瓶颈
  - 设计完善改造方案
  - 数据按需获取
  - 数据序列化协议优化
- 内部压测验证
- 推广业务服务落地验证

### Go语言优化

> 编译器&运行时优化

- 优化内存分配策略
- 优化代码编译流程，生成更高效的程序
- 内部压测验证
- 推广业务服务落地验证
