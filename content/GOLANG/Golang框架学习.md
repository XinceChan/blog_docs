```json
{
    "date":"2023.01.20 17:00",
    "author":"XinceChan",
    "tags":["Golang","Go框架"],
    "musicId":"2004562490"
}
```

## 三件套的使用

### Gorm的基础使用

- Gorm 使用名为 ID 的字段作为主键
- 使用结构体的蛇形复数作为表明
- 字段名的蛇形作为列名
- 使用 CreatedAt、 UpdatedAt 字段作为创建、更新时间

```go
// 定义gorm model
type Product struct {
    Code string
    Price uint
}
// 为model定义表名
func (p *Product) TableName() string {
    return "product"
}
func main() {
    db, err := gorm.Open(mysql.Open("user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"), &gorm.Config{}) // 连接数据库
    if err != nil {
        panic("failed to connect database")
    }
    // Create
    db.Create(&Product{Code: "D42", Price: 100})
    // Read
    var product Product
    db.First(&product, 1) // 根据整形主键查找
    db.First(&product, "code = ?", "D42") // 查找 code 字段值为 D42 的记录
    // Update - 将 product 的 price 更新为 200
    db.Model(&product).Update(Product{Price: 200, Code: "F42"}) // 仅更新非零值字段
    db.Model(&product).Updates(map[string]interface{}{"Price": 200, "Code": "F42"})
    // Delete - 删除 product
    db.Delete(&product, 1)
}
```

### Gorm支持的数据库

GORM 官方支持的数据库类型有： MySQL, PostgreSQL, SQlite, SQL Server

```go
import (
    "gorm.io/driver/mysql"
    "gorm.io/gorm"
)

func main() {
    // 参考 https://github.com/go-sql-driver/mysql#dsn-data-source-name 获取详情
    dsn := "user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
    db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
}
```

### Gorm创建数据

```go
package main

import (
	"fmt"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

type Product struct {
	ID    uint   `gorm:"primarykey"`
	Code  string `gorm:"column: code"`
	Price uint   `gorm:"column: user_id"`
}

func main() {
    db, err := gorm.Open(mysql.Open("user:pass@tcp(127.0.0.1:3306)/dbname?charset=utf8mb4&parseTime=True&loc=Local"), &gorm.Config{}) // 连接数据库
	if err != nil {
		panic("failed to connect database")
	}
	db.AutoMigrate(&Product{})
	// 创建一条
	p := &Product{Code: "D42"}
	res := db.Create(p)
	fmt.Println(res.Error) // 获取err
	fmt.Println(p.ID)      // 返回插入数据的主键
	// 创建多条
	products := []*Product{{Code: "D42"}, {Code: "D43"}}
	res = db.Create(products)
	fmt.Println(res.Error) // 获取err
	for _, p := range products {
		fmt.Println(p.ID)
	}
}
```

#### 如何使用默认值

您可以通过标签 `default` 为字段定义默认值，如：

>```go
>type User struct {
>  	ID         int64
>  	Name       string `gorm:"default:galeone"`
>  	Age        int64  `gorm:"default:18"`
>  	uuid.UUID  UUID   `gorm:"type:uuid;default:gen_random_uuid()"` // 数据库函数
>}
>```

**注意** `0`、`''`、`false` 之类零值，这些字段定义的默认值不会被保存到数据库，您需要使用指针类型或 Scanner/Valuer 来避免这个问题，例如：

> ```go
> type User struct {
>   	gorm.Model
>   	Name string
>   	Age  *int           `gorm:"default:18"`
>   	Active sql.NullBool `gorm:"default:true"`
> }
> ```

**注意** 对于在数据库中有默认值的字段，你必须为其 struct 设置 `default` 标签，否则 GORM 将在创建时使用该字段的零值，例如：

>```go
>type User struct {
>    ID   string `gorm:"default:uuid_generate_v3()"`
>    Name string
>    Age  uint8
>}
>```

#### 如何使用upsert

GORM 为不同数据库提供了兼容的 Upsert 支持

```go
import "gorm.io/gorm/clause"

// 不处理冲突
DB.Clauses(clause.OnConflict{DoNothing: true}).Create(&user)

// `id` 冲突时，将字段值更新为默认值
DB.Clauses(clause.OnConflict{
  Columns:   []clause.Column{{Name: "id"}},
  DoUpdates: clause.Assignments(map[string]interface{}{"role": "user"}),
}).Create(&users)
// MERGE INTO "users" USING *** WHEN NOT MATCHED THEN INSERT *** WHEN MATCHED THEN UPDATE SET ***; SQL Server
// INSERT INTO `users` *** ON DUPLICATE KEY UPDATE ***; MySQL
```

### Gorm查询数据

```go
func main() {
	db, err := gorm.Open(mysql.Open("root:1838830210@tcp(127.0.0.1:3306)/gorm?charset=utf8mb4&parseTime=True&loc=Local"), &gorm.Config{}) // 连接数据库
	if err != nil {
		panic("failed to connect database")
	}
	db.AutoMigrate(&User{})
	// 获取第一条记录（主键升序），查询不到数据则返回 ErrRecordNotFound
	u := &User{}
	db.First(u) // SELECT * FROM users ORDER BY id LIMIT 1;
	// 查询多条数据
	users := make([]*User, 0)
	result := db.Where("age > 10").Find(&users) // SELECT * FROM users where age > 10;
	fmt.Println(result.RowsAffected)            // 返回找到的记录数，相当于 `len(users)`
	fmt.Println(result.Error)                   // returns error
	// IN SELECT * FROM users WHERE name IN ('jinzhu', 'jinzhu 2');
	db.Where("name IN ?", []string{"jinzhu", "jinzhu 2"}).Find(&users)
	// LIKE SELECT * FROM users WHERE name LIKE '%jin%';
	db.Where("name LIKE ?", "%jin%").Find(&users)
	// AND SELECT * FROM users WHERE name = 'jinzhu' AND age >= 22;
	db.Where("name = ? AND age >= ?", "jinzhu", "22").Find(&users)

	// SELECT * FROM users WHERE name = "jinzhu";
	db.Where(&User{Name: "jinzhu", Age: 0}).Find(&users)
	// SELECT * FROM users WHERE name = "jinzhu" AND age = 0;
	db.Where(map[string]interface{}{"Name": "jinzhu", "Age": 0}).Find(&users)
}
```

#### First 的使用踩坑

使用 `First` 时，需要注意查询不到数据会返回 `ErrRecordNotFound`。

使用 `Find` 查询多条数据，查询不到数据不会返回错误。

#### 使用结构体作为查询条件

当使用结构作为条件查询时，Gorm 只会查询非零值条件。这意味着如果您的字段值为 `0`、`''`、`false` 之类零值，该字段不会被用于构建查询条件，使用 `Map`来构建查询条件。

#### 选择特定的字段

> ```go
> 选择您想从数据库中检索的字段，默认情况下会选择全部字段
> 
> db.Select("name", "age").Find(&users)
> // SELECT name, age FROM users;
> 
> db.Select([]string{"name", "age"}).Find(&users)
> // SELECT name, age FROM users;
> 
> db.Table("users").Select("COALESCE(age,?)", 42).Rows()
> // SELECT COALESCE(age,'42') FROM users;
> ```

#### 智能选择字段

GORM 允许通过 [`Select`](https://gorm.cn/zh_CN/docs/query.html) 方法选择特定的字段，如果您在应用程序中经常使用此功能，你也可以定义一个较小的结构体，以实现调用 API 时自动选择特定的字段，例如：

> ```go
> type User struct {
>   	ID     uint
>   	Name   string
>   	Age    int
>   	Gender string
>   // 假设后面还有几百个字段...
> }
> 
> type APIUser struct {
>   	ID   uint
>   	Name string
> }
> 
> // 查询时会自动选择 `id`, `name` 字段
> db.Model(&User{}).Limit(10).Find(&APIUser{})
> // SELECT `id`, `name` FROM `users` LIMIT 10
> ```

### Gorm更新数据

```go
// 条件更新单个列
// UPDATE users SET name='hello', updated_at = '2013-11-17 21:34:10' WHERE age > 18;
db.Model(&User{ID: 111}).Where("age > ?", 18).Update("name", "hello")

// 更新多个列
// 根据 `struct` 更新属性，只会更新非零值的字段
// UPDATE users SET name='hello', age = 18, updated_at = '2013-11-17 21:34:10' WHERE id = 111;
db.Model(&User{ID: 111}).Updates(User{Name: "hello", Age: 18})

// 根据 `map` 更新属性
// UPDATE users SET name='hello', age = 18, actived=false, updated_at = '2013-11-17 21:34:10' WHERE id = 111;
db.Model(&User{ID: 111}).Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})

// 更新选定字段
// UPDATE users SET name='hello' WHERE id = 111;
db.Model(&User{ID: 111}).Select("name").Updates(map[string]interface{}{"name": "hello", "age": 18, "actived": false})

// SQL表达式更新
// UPDATE users SET price=price * 2 + 100, updated_at = '2013-11-17 21:34:10' WHERE id = 111;
db.Model(&User{ID: 111}).Update("age", gorm.Expr("age * ? + ?", 2, 100))
```

**注意** 使用 `struct` 更新时，只会更新非零值，如果需要更新零值可以使用 `Map` 更新或者使用 `Select` 选择字段。

### Gorm删除数据

#### 物理删除

```go
db.Delete(&User{}, 10) // DELETE FROM users WHERE id = 10;
db.Delete(&User{}, "10") // DELETE FROM users WHERE id = 10;
db.Delete(&User{}, []int{1, 2, 3}) // DELETE FROM users WHERE id IN (1, 2, 3);
db.Where("name LIKE ?", "%jinzhu%").Delete(User{}) // DELETE FROM users where name LIKE "%jinzhu%";
db.Delete(User{}, "email LIKE ?", "%jinzhu%") // DELETE FROM users where name LIKE "%jinzhu%";
```

#### 软删除

如果您的模型包含了一个 `gorm.deletedat` 字段（`gorm.Model` 已经包含了该字段)，它将自动获得软删除的能力！

拥有软删除能力的模型调用 `Delete` 时，记录不会被从数据库中真正删除。但 GORM 会将 `DeletedAt` 置为当前时间， 并且你不能再通过正常的查询方法找到该记录。

> ```go
> // user 的 ID 是 `111`
> db.Delete(&user)
> // UPDATE users SET deleted_at="2013-10-29 10:23" WHERE id = 111;
> 
> // 批量删除
> db.Where("age = ?", 20).Delete(&User{})
> // UPDATE users SET deleted_at="2013-10-29 10:23" WHERE age = 20;
> 
> // 在查询时会忽略被软删除的记录
> db.Where("age = 20").Find(&user)
> // SELECT * FROM users WHERE age = 20 AND deleted_at IS NULL;
> ```

如果您不想引入 `gorm.Model`，您也可以这样启用软删除特性：

> ```go
> type User struct {
>   ID      int
>   Deleted gorm.DeletedAt // 启用软删除特性
>   Name    string
> }
> ```

#### Unscoped() 查询被软删除的记录

正常查询会忽略被软删除的记录

> ```go
> db.Unscoped().Where("age = 20").Find(&users)
> // SELECT * FROM users WHERE age = 20;
> ```

#### 永久删除

您也可以使用 `Unscoped` 永久删除匹配的记录

> ```go
> db.Unscoped().Delete(&order)
> // DELETE FROM orders WHERE id=10;
> ```

### Gorm事务

Gorm 提供了Begin、Commit、Rollback方法用于使用事务

```go
tx := db.Begin() // 开始事务
// 在事务中执行一些 db 操作（从这里开始，您应该使用 'tx' 而不是 'db'）
if err := tx.Create(&User{Name: "name"}).Error; err != nil {
    tx.Rollback()
    // 遇到错误时事务回滚
    return
}
if err := tx.Create(&User{Name: "name1"}).Error; err != nil {
    tx.Rollback()
    return
}
// 提交事务
tx.Commit()
```

#### 存在问题

在正常开发中，可能存在漏写 Commit、Rollback。

Gorm 提供了 Transaction 方法用于自动提交事务，避免用户漏写 Commit、Rollback。

```go
if err := db.Transaction(func(tx *gorm.DB) error {
    if err = tx.Create(&User{Name: "name"}).Error; err != nil {
        return err
    }
    if err = tx.Create(&User{Name: "name1"}).Error; err != nil {
        tx.Rollback()
        return err
    }
    return nil
}); err != nil {
    return
}
```

### Gorm Hook

Hook 是在创建、查询、更新、删除等操作之前、之后调用的函数。

如果您已经为模型定义了指定的方法，它会在创建、更新、查询、删除时自动被调用。如果任何回调返回错误，GORM 将停止后续的操作并回滚事务。

钩子方法的函数签名应该是 `func(*gorm.DB) error`

#### 创建对象

```json
// 开始事务
BeforeSave
BeforeCreate
// 关联前的 save
// 插入记录至 db
// 关联后的 save
AfterCreate
AfterSave
// 提交或回滚事务
```

```go
func (u *User) BeforeCreate(tx *gorm.DB) (err error) {
    if u.Age < 0 {
        return errors.New("can't save invalid data")
    }
    return
}
func (u *User) AfterCreate(tx *gorm.DB) (err error) {
    return tx.Create(&Email{ID: u.ID, Email: u.Name + "@***.com"}).Error
}
```

> **注意** 在 GORM 中保存、删除操作会默认运行在事务上， 因此在事务完成之前该事务中所作的更改是不可见的，如果您的钩子返回了任何错误，则修改将被回滚。

其他相关操作： [Gorm Hook](https://gorm.cn/zh_CN/docs/hooks.html)

### Gorm 性能提高

```go
db, err := gorm.Open(mysql.Open("root:1838830210@tcp(127.0.0.1:3306)/gorm?charset=utf8mb4&parseTime=True&loc=Local"), 
                     &gorm.Config{
                         SkipDefaultTransaction: true,
                         PrepareStmt: true,
                     })
```

对于写操作（创建、更新、删除），为了确保数据的完整性，GORM 会将它们封装在事务内运行。但这会降低性能，你可以使用 `SkipDefaultTransaction` 关闭默认事务。

使用 `PrepareStmt` 缓存预编译语句可以提高后续调用的速度。

相关文档： [Gorm Performance](https://gorm.cn/zh_CN/docs/performance.html)

