```json
{
  "date":"2022.01.17 13:20",
  "author":"XinceChan",
  "tags":["Golang"],
  "musicId":"1948620191"
}
```

本章将从工程实践角度，讲授在企业项目实际开发过程中的所遇的难题，重点讲解Go语言的进阶之路，以及在其依赖管理过程中如何演进。包括：1、语言进阶，从并发编程的视角带大家了解Go高性能的本质；2、依赖管理，了解Go语言依赖管理的演进路线；3、测试，从单元测试实践出发，提升大家的质量意识；4、项目实战，通过项目需求、需求拆解、逻辑设计、代码实现带领大家感受下真正的项目开发。

>本节讲述 3、测试，从单元测试实践出发，提升大家的质量意识

## 测试相关

- 回归测试
- 集成测试
- 单元测试

## 单元测试

- 所有测试文件以_test.go结尾
- func TestXxx(*testing.T)
- 初始化逻辑放到TestMain()中

```go
func TestMain(m *testing.M) {
  // 测试前:数据装载，配置初始化等前置工作
  
  code := m.Run()
  
  // 测试后:释放资源等收尾工作
  
  os.Exit(code)
}
```

#### example

```go
func HelloTom() string {
  return "Jerry"
}

func TestHelloTom(t *testing.T) {
  output := HelloTom()
  expectOutput := "Tom"
  if output != expectOutput {
    t.Errorf("Expected %s do not match actual %s", expectOutput, output)
  }
}
```

可以通过引入 github.com/stretchr/testify/assert 来简化测试，也有很多开源的assert包

```go
func HelloTom() string {
  return "Jerry"
}

func TestHelloTom(t *testing.T) {
  output := HelloTom()
  expectOutput := "Tom"
  assert.Equal(t, expectOutput, output)
}
```

### 单元测试 - 覆盖率

```go
func JudgePassLine(score int16) bool {
	if score >= 60 {
		return true
	}
	return false
}

func TestJudgePassLine(t *testing.T) {
	isPass := JudgePassLine(70)
	assert.Equal(t, true, isPass)
}
```

**命令行输出**

```go
go test judge_test.go judge.go --cover
ok      command-line-arguments  0.420s  coverage: 66.7% of statements
```

通过添加测试 `TestJudgePassLineFail(t *testing.T)`

```go
func TestJudgePassLineFail(t *testing.T) {
	isPass := JudgePassLine(50)
	assert.Equal(t, false, isPass)
}
```

**命令行输出**

```go
go test judge_test.go judge.go --cover
ok      command-line-arguments  0.326s  coverage: 100.0% of statements
```

### 单元测试 - 文件处理

```go
func ReadFirstLine() string {
	open, err := os.Open("log")
	defer open.Close()

	if err != nil {
		return ""
	}
	scanner := bufio.NewScanner(open)
	for scanner.Scan() {
		return scanner.Text()
	}
	return ""
}

func ProcessFirstLine() string {
	line := ReadFirstLine()
	destLine := strings.ReplaceAll(line, "11", "00")
	return destLine
}

func TestProcessFirstLine(t *testing.T) {
	firstLine := ProcessFirstLine()
	assert.Equal(t, "line00", firstLine)
}
```

### 单元测试 - Mock

monkey : https://github.com/bouk/monkey

快速Mock函数

- 为一个函数打桩
- 为一个方法打桩

```go
func TestProcessFirstLineWithMock(t *testing.T) {
	monkey.Patch(ReadFirstLine, func() string {
		return "line110"
	})
	defer monkey.Unpatch(ReadFirstLine)
	line := ProcessFirstLine()
	assert.Equal(t, "line000", line)
}
```

### 单元测试 - 基准测试

```go
func BenchmarkSelect(b *testing.B) {
  InitServerIndex()
  b.ResetTimer() // 重置时间
  for i := 0; i < b.N; i++ {
    Select()
  }
}

func BenchmarkSelectParallel(b *testing.B) {
  InitServerIndex()
  b.ResetTimer()
  b.RunParallel(func(pb *testing.PB) {
    for pb.Next() {
      Select()
    }
  })
}
```

可以尝试 https://github.com/bytedance/gopkg

fastrand 在应对高并发的情况下，可能效果更好。