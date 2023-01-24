```json
{
    "date":"2023.01.24 23:20",
    "author":"XinceChan",
    "tags":["Golang","雪花ID"],
    "musidId":"1380239189"
}
```

在以前的项目中，最常见的两种主键类型是自增Id和UUID，在比较这两种ID之前首先要搞明白一个问题，就是为什么主键有序比无序查询效率要快，因为自增Id和UUID之间最大的不同点就在于有序性。

我们都知道，当我们定义了主键时，数据库会选择表的主键作为聚集索引(B+Tree)，mysql 在底层是以数据页为单位来存储数据的。

也就是说如果主键为`自增 id`的话，mysql 在写满一个数据页的时候，直接申请另一个新数据页接着写就可以了。**如果一个数据页存满了，mysql 就会去申请一个新的数据页来存储数据**。如果主键是`UUID`，为了确保索引有序，mysql 就需要将每次插入的数据都放到合适的位置上。**这就造成了页分裂，这个大量移动数据的过程是会严重影响插入效率的**。

一句话总结就是，InnoDB表的数据写入顺序能和B+树索引的叶子节点顺序一致的话，这时候存取效率是最高的。

但是为什么很多情况又不用`自增id`作为主键呢？

- 容易导致主键重复。比如导入旧数据时，线上又有新的数据新增，这时就有可能在导入时发生主键重复的异常。为了避免导入数据时出现主键重复的情况，要选择在应用停业后导入旧数据，导入完成后再启动应用。显然这样会造成不必要的麻烦。而UUID作为主键就不用担心这种情况。
- 不利于数据库的扩展。当采用自增id时，分库分表也会有主键重复的问题。UUID则不用担心这种问题。

那么问题就来了，`自增id`会担心主键重复，`UUID`不能保证有序性，有没有一种ID既是有序的，又是唯一的呢？

当然有，就是`雪花ID`。

引用自 知乎 [什么是雪花ID？](https://zhuanlan.zhihu.com/p/374667160) https://zhuanlan.zhihu.com/p/374667160

### 三类主键

- 自增ID： 1、2、3、4、5 .....
- uuid: UUID 是 通用唯一识别码（Universally Unique Identifier）的缩写，它是在一定的范围内（从特定的名字空间到全球）唯一的机器生成的标识符。通用唯一标识符的意思，可以以业务实际`user id`为主键，比如QQ号、手机号等
- 雪花ID: `snowflake`是 `Twitter` 开源的分布式ID生成算法，结果是64bit的Long类型的ID，有着全局唯一和有序递增的特点。

## 什么是雪花ID

```json
+--------------------------------------------------------------------------+
| 1 Bit Unused | 41 Bit Timestamp |  10 Bit NodeID  |   12 Bit Sequence ID |
+--------------------------------------------------------------------------+
```

### ID Format

By default, the ID format follows the original Twitter snowflake format.

- ID存储在 `int64` 中，用到 63 位。
- 最高位是符号位，因为生成的 ID 总是正数，始终为0，不可用。
- `41位` 用于存储一个精确到毫秒级的时间戳。时间位还有一个很重要的作用是可以根据时间进行排序。
- `10位` 的机器标识，支持存储 `0-1023` 共 `1024` 个节点
- `12位` 的计数序列号，序列号即一系列的自增ID，可以支持同一节点同一毫秒生成多个ID序号，12位的计数序列号支持每个节点每毫秒产生4096个ID序号。

### Go Snowflake

#### **Installing**

```go
go get github.com/bwmarrin/snowflake
```

#### Usage

- 使用 `snowflake.NewNode(node int64)` 构建一个新的雪花节点，里面传入一个唯一的节点号，默认设置允许 `0-1023`。

> If you have set a custom NodeBits value, you will need to calculate what your node number range will be.

- 使用 `node.Generate()` 生成一个雪花ID。

#### **Example**

```go
package main

import (
	"fmt"

	"github.com/bwmarrin/snowflake"
)

func main() {

	// Create a new Node with a Node number of 1
	node, err := snowflake.NewNode(1)
	if err != nil {
		fmt.Println(err)
		return
	}

	// Generate a snowflake ID.
	id := node.Generate()

	// Print out the ID in a few different ways.
	fmt.Printf("Int64  ID: %d\n", id)
	fmt.Printf("String ID: %s\n", id)
	fmt.Printf("Base2  ID: %s\n", id.Base2())
	fmt.Printf("Base64 ID: %s\n", id.Base64())

	// Print out the ID's timestamp
	fmt.Printf("ID Time  : %d\n", id.Time())

	// Print out the ID's node number
	fmt.Printf("ID Node  : %d\n", id.Node())

	// Print out the ID's sequence number
	fmt.Printf("ID Step  : %d\n", id.Step())

  	// Generate and print, all in one.
  	fmt.Printf("ID       : %d\n", node.Generate().Int64())
}
```

**output**:

```
Int64  ID: 1617911886731284480
String ID: 1617911886731284480
Base2  ID: 1011001110011111110100100100111110110100000000001000000000000
Base64 ID: MTYxNzkxMTg4NjczMTI4NDQ4MA==
ID Time  : 1674575227803
ID Node  : 1
ID Step  : 0
ID       : 1617911886735478784
```

### 性能

在默认设置下，雪花生成器每秒可以最多生成 `4096` 个唯一 ID。由于雪花生成器是单线程的，主要的限制是你系统上单个处理器的最大速度。

```go
go test -run=^$ -bench=.
```

```
goos: darwin
goarch: arm64
pkg: snowflake-example/snowflake
BenchmarkParseBase32-8           	133637931	         8.610 ns/op	       0 B/op	       0 allocs/op
BenchmarkBase32-8                	42938995	        27.53 ns/op	      24 B/op	       1 allocs/op
BenchmarkParseBase58-8           	157382682	         7.700 ns/op	       0 B/op	       0 allocs/op
BenchmarkBase58-8                	83316454	        14.41 ns/op	       0 B/op	       0 allocs/op
BenchmarkGenerate-8              	 4924300	       244.0 ns/op	       0 B/op	       0 allocs/op
BenchmarkGenerateMaxSequence-8   	30233998	        38.89 ns/op	       0 B/op	       0 allocs/op
BenchmarkUnmarshal-8             	30578994	        38.70 ns/op	      24 B/op	       1 allocs/op
BenchmarkMarshal-8               	40085125	        29.66 ns/op	      24 B/op	       1 allocs/op
PASS
ok  	snowflake-example/snowflake	11.619s
```

