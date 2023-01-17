```json
{
  "date":"2022.01.17 13:20",
  "author":"XinceChan",
  "tags":["Golang"],
  "musicId":"3986376"
}
```

本章将从工程实践角度，讲授在企业项目实际开发过程中的所遇的难题，重点讲解Go语言的进阶之路，以及在其依赖管理过程中如何演进。包括：1、语言进阶，从并发编程的视角带大家了解Go高性能的本质；2、依赖管理，了解Go语言依赖管理的演进路线；3、测试，从单元测试实践出发，提升大家的质量意识；4、项目实战，通过项目需求、需求拆解、逻辑设计、代码实现带领大家感受下真正的项目开发。

>本节讲述 1、语言进阶，从并发编程的视角带大家了解Go高性能的本质

### 语言进阶 - 并发编程

- 并发：多线程程序在一个核的cpu上运行

- 并行：多线程程序在多个核的cpu上运行

- 协程：用户态，轻量级线程，栈MB级别。
- 线程：内核态，线程跑多个协程，栈KB级别。

#### GoRoutine

```go
func hello(i int) {
  println("hello goroutine : " + fmt.Sprint(i))
}

func HelloGoRoutine() {
  for i := 0; i < 5; i++ {
    go func(j int) {
      hello(j)
    }(i)
  }
  time.Sleep(time.Second)
}
```

**输出结果**

```go
hello goroutine : 3
hello goroutine : 2
hello goroutine : 1
hello goroutine : 0
hello goroutine : 4
```

#### Channel

```go
make(chan T, [buffer size])
```

- 无缓冲通道  `make(chan int)`
- 有缓冲通道  `make chan(int, 2)`

```go
// A 子协程发送0～9数字
// B 子协程计算输入数字的平方
// 主协程输出最后的平方数
func CalSquare() {
  src := make(chan int)
	dest := make(chan int, 3)
	// A
	go func() {
		defer close(src)
		for i := 0; i < 10; i++ {
			src <- i
		}
	}()
	// B
	go func() {
		defer close(dest)
		for i := range src {
			dest <- i * i
		}
	}()
	// 主协程
	for i := range dest {
		// 复杂操作
		println(i)
	}
}
```

#### 并发安全 - Lock

```go
var (
  x int64
  lock sync.Mutex
)

func addWithLock() {
  for i := 0; i < 2000; i++ {
    lock.Lock()
    x += 1
    lock.Unlock()
  }
}

func addWithoutLock() {
  for i := 0; i < 2000; i++ {
    x += 1
  }
}

func Add() {
  x = 0
  for i := 0; i < 5; i++ {
    go addWithoutLock()
  }
  time.Sleep(time.Second)
  println("WithoutLock:", x)
  x = 0
  for i := 0; i < 5; i++ {
    go addWithLock()
  }
  time.Sleep(time.Second)
  println("WithLock:", x)
}
```

**输出结果**

```go
WithoutLock: 8090
WithLock: 10000
```

#### WaitGroup

```go
Add(delta int)  // 计数器+delta
Done()  // 计数器-1
Wait()  // 阻塞直到计数器为0
```

```go	
func ManyGoWait() {
	var wg sync.WaitGroup
	wg.Add(5)
	for i := 0; i < 5; i++ {
		go func(j int) {
			defer wg.Done()
			hello(j)
		}(i)
	}
	wg.Wait()
}
```

