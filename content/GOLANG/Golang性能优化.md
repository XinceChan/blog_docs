```json
{
  "date":"2023.01.17 23:30",
  "author":"XinceChan",
  "tags":["Golang","性能优化"],
  "musicId":"64106"
}
```

本节主要简要介绍了高质量编程的定义和原则，分享了代码格式、注释、命名规范、控制流程、错误和异常处理五方面的常见编码规范。目标主要达成以下四点：如何编写更简洁清晰的代码；常用Go语言程序优化手段；熟悉Go程序性能分析工具；了解工程中性能优化的原则和流程。

## 性能优化

> 简介

- 性能优化的前提是满足正确可靠、简洁清晰等质量因素
- 性能优化是综合评估，有时候时间效率和空间效率可能对立
- 针对Go语言特性，介绍Go相关的性能优化建议

### 性能优化建议 - Slice

> slice 预分配内存

- 尽可能在使用make()初始化切片时提供容量信息

```go
func NoPreAlloc(size int) {
  data := make([]int, 0)
  for k := 0; k < size; k++ {
    data = append(data, k)
  }
}

func PreAlloc(size int) {
  data := make([]int, size)
  for k := 0; k < size; k++ {
    data = append(data, k)
  }
}
```

```go
BenchmarkNoPreAlloc
BenchmarkNoPreAlloc-8           11345640                97.54 ns/op          248 B/op          5 allocs/op
BenchmarkPreAlloc
BenchmarkPreAlloc-8             24897922                47.88 ns/op          240 B/op          2 allocs/op
```

- 切片本质是一个数组片段的描述
  - 包含数组指针
  - 片段的长度
  - 片段的容量（不改变内存分配情况下的最大长度）
- 切片操作并不复制切片指向的元素
- 创造一个新的切片会复制原来切片的底层数组

> 另一个陷阱：大内存未释放

- 在已有切片基础上创建切片，不会创建新的底层数组
- 场景
  - 原切片较大，代码在原切片基础上新建小切片
  - 原底层数组在内存中有引用，得不到释放
- 可使用copy替代re-slice

```go
func GetLastBySlice(origin []int) []int {
  return origin[len(origin)-2:]
}

func GetLastByCopy(origin []int) []int {
  result := make([]int, 2)
  copy(result, origin[len(origin)-2:])
  return result
}
```

### 性能优化建议 - Map

> map 预分配内存

```go
func NoPreAlloc(size int) {
  data := make(map[int]int)
  for i := 0; i < size; i++ {
    data[i] = 1
  }
}

func PreAlloc(size int) {
  data := make(map[int]int, size)
  for i := 0; i < size; i++ {
    data[i] = 1
  }
}
```

```go
BenchmarkNoPreAlloc
BenchmarkNoPreAlloc-8            4565164               232.5 ns/op           291 B/op          1 allocs/op
BenchmarkPreAlloc
BenchmarkPreAlloc-8              6930066               172.4 ns/op           291 B/op          1 allocs/op
```

- 不断向map中添加元素的操作会触发map的扩容
- 提前分配好空间可以减少内存拷贝和Rehash的消耗
- 建议根据实际需求提前预估好需要的空间

### 性能优化建议 - 字符串处理

> 使用 strings.Builder

常用的字符串拼接方式

```go
func Plus(n int, str string) string {
  s := ""
  for i := 0; i < n; i++ {
    s += str
  }
  return s
}

func ByteBuffer(n int, str string) string {
  buf := new(bytes.Buffer)
  for i := 0; i < n; i++ {
    buf.WriteString(str)
  }
  return buf.String()
}

func StrBuilder(n int, str string) string {
  var builder strings.Builder
  for i := 0; i < n; i++ {
    builder.WriteString(str)
  }
  return builder.String()
}
```

- 使用 + 拼接性能最差，strings.Builder, bytes.Buffer 相近，strings.Builder 更快
- 分析
  - 字符串在Go语言中是不可变类型，占用内存大小是固定的
  - 使用 + 每次都会重新分配内存
  - strings.Builder, bytes.Buffer 底层都是 []byte 数组
  - 内存扩容策略，不需要每次拼接重新分配内存
- bytes.Buffer 转化为字符串时重新申请了一块空间
- strings.Builder 直接将底层的 []byte 转化成了字符串类型并返回

```go
func PreStrBuilder(n int, str string) string {
  var builder strings.Builder
  builder.Grow(n * len(str))
  for i := 0; i < n; i++ {
    builder.WriteString(str)
  }
  return builder.String()
}

func PreByteBuffer(n int, str string) string {
  buf := new(bytes.Buffer)
  buf.Grow(n * len(str))
  for i := 0; i < n; i++ {
    buf.WriteString(str)
  }
  return buf.String()
}
```

```go
BenchmarkPreStrBuilder
BenchmarkPreStrBuilder-8        24024008                49.63 ns/op           24 B/op          1 allocs/op
BenchmarkPreByteBuffer
BenchmarkPreByteBuffer-8        17963545                66.67 ns/op           88 B/op          2 allocs/op
```

### 性能优化建议 - 空结构体

> 使用空结构体节省内存

- 空结构体struct{}实例不占据任何的内存空间
- 可作为各种场景下的占位符使用
  - 节省资源
  - 空结构体本身具备很强的语义，即这里不需要任何值，仅作为占位符

```go
func EmptyStructMap(n int) {
  m := make(map[int]struct{})
  for i := 0; i < n; i++ {
    m[i] = struct{}{}
  }
}

func BoolMap(n int) {
  m := make(map[int]bool)
  for i := 0; i < n; i++ {
    m[i] = false
  }
}
```

- 实现Set，可以考虑用 map 来代替

- 对于这个场景，只需要用到 map 的键，而不需要值
- 即使是将 map 的值设置为 bool 类型，也会多占据一个 1 个字节空间

一个开源实现: https://github.com/deckarep/golang-set/blob/main/threadunsafe.go

### 性能优化建议 - atomic

> 如何使用atomic包

```go
type atomicCounter struct {
  i int32
}

func AtomicAddOne(c *atomicCounter) {
  atomic.AddInt32(&c.i, 1)
}
```

```go
type mutexCounter struct {
  i int32
  m sync.Mutex
}

func MutexAddOne(c *mutexCounter) {
  c.m.Lock()
  c.i++
  c.m.Unlock()
}
```

```
BenchmarkAtomicAddOne
BenchmarkAtomicAddOne-8         169633371                6.971 ns/op           0 B/op          0 allocs/op
BenchmarkMutexAddOne
BenchmarkMutexAddOne-8          86425231                13.57 ns/op            0 B/op          0 allocs/op
```

- 锁的实现是通过操作系统来实现，属于系统调用
- atmoic 操作是通过硬件实现，效率比锁高
- sync.Mutex 应该用来保护一段逻辑，不仅仅用于保护一个变量
- 对于非数值操作，可以使用atomic.Value，能承载一个interface{}
