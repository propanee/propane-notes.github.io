# golang数据库编程

## 连接流程

- 要连接到SQL数据库，首先加载目标数据库的驱动，驱动里包含与该数据库交互的逻辑；
  - 重要的包：database/sql和对应数据库驱动的包，如视频中微软SqlServer的go-mssqldb；
  - Go语言中的`database/sql`包提供了保证SQL或类SQL数据库的泛用接口，并不提供具体的数据库驱动。使用`database/sql`包时必须注入（至少）一个数据库驱动。

- sql.Open
  - 数据库驱动的名称，数据源名称（数据库连接字符串）；
  - 得到只想sql.DB这个struct的指针；
  - 并不会连接数据库，甚至不会验证其参数，只是把后续连接到数据库所必需的struct设置好了；
  - 真正的连接是在被需要的时候才进行懒设置，即要连数据库时该连接才真正的被设置。
- sql.DB用于操作数据库，代表0个或多个底层连接的池，这些连接由sql包维护，sql包自动创建和释放这些连接；
  - 它对于多个goroutine并发的使用是安全的（线程安全）；
  - sql.DB不许要进行关闭（想关闭也是可以的）；
  - 它是用来处理数据库的而不是实际的连接；
  - 这个抽象包含了数据库连接的池，并会对此进行维护；
  - 使用sql.DB时，可定义它的全局变量进行使用，也可以将它传递函数/方法里。
- 获得驱动：正常做法是使用`sql.Register()`函数、数据库驱动的名称和一个实现了driver.Driver接口的struct，来注册数据库的驱动。如`sql.Register("sqlserver", &drv{})`；
  - 视频中没有写也可以加载成功，是因为SqlServer的驱动是在驱动的包go-mssqldb被引入时，它的init函数运行并进行了自我注册；
  - 引入go-mssqldb包时，将该包的名设置为下划线_，是因为不直接使用数据库驱动（只需要它起的“副作用”），只使用database/sql（无论使用哪种数据库，在代码层面都是一样的，sql语句可能会有一些差别，未来升级也无需改变代码）；
  - go没有官方的数据库驱动，所有的数据库驱动都是第三方的，但它们都遵循sql.driver包里定义的接口。
- func(*DB) PingContext
  - **db.PingContext()**函数是用来验证与数据库的连接是否仍然有效，如有必要则建立一个连接。
  - 这个函数需要一个Context(上下文)类型的参数，这种类型可以携带截止时间、取消信号和其它请求范围的值，并且可以横跨API边界和进程。
  - 创建context使用的是 **context.Background()**函数。该函数返回一个非nil的空Context。它不会被取消，它没有值，没有截止时间。通常用在main函数、初始化或测试中，作为传入请求的顶级Context。

> Ping 和PingContext都是检验验证与数据库的连接是否仍然有效，如有必要则建立一个连接。区别是Ping在内部使用context.Background，PingContext用于指明Context。
>
> Ping / PingContext verifies a connection to the database is still alive, establishing a connection if necessary.
> Ping uses context.Background internally; to specify the context, use PingContext.
>
> Ping and PingContext may be used to determine if communication with the database server is still possible. 
>
> When used in a command line application Ping may be used to establish that further queries are possible; that the provided DSN is valid. 
>
> When used in long running service Ping may be part of the health checking system. 

```go
var db *sql.DB

const (
	server   = "xxxx.database.windows.net"
	port     = 1433
	user     = "xxxxx"
	password = "xxxxx"
	database = "go-db"
)

func main() {
	connStr := fmt.Sprintf("server=%s;user_id=%s; passwOrd=%s; port=%d; database=%s;", server, user, password, port, database)
	db, err := sql.Open("sqlserver", connStr)
	if err != nil {
		log.Fatalln(err.Error())
	}
    
	ctx := context.Background()
	
	err = db.PingContext(ctx)
	if err != nil {
		log.Fatalln(err.Error())
	}
	fmt.Println("Connected ! ")
}
```

```go
// 组成sql语句的模板
Prepare
PrepareContext
```



# mysql

[Go语言操作MySQL-李文周](https://www.liwenzhou.com/posts/Go/mysql/)

数据库 [mysql](https://dev.mysql.com/doc/)，**MySQL**在过去由于性能高、成本低、可靠性好，已经成为最流行的开源数据库，因此被广泛地应用在Internet上的中小型网站中。随着MySQL的不断成熟，它也逐渐用于更多大规模网站和应用，比如维基百科Google和Facebook等网站。非常流行的开源软件组合LAMP中的“M”指的就是MySQL。

```shell
# 启动mysql服务
sudo service mysql start 
# 打开mysql客户端 输入密码
mysql -u root -p 
# 查看所有数据库
SHOW DATABASES;
# 创建数据库 Query OK, 1
CREATE DATABASE sql_test; 
# 使用数据库 Database changed
use sqltest
# 创建数据表
CREATE TABLE `user` (
    `id` BIGINT(20) NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(20) DEFAULT '',
    `age` INT(11) DEFAULT '0',
    PRIMARY KEY(`id`)
)ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4;
# 查看所有数据表
SHOW TABLES;
# 查看数据表的结构
DESCRIBE user;
# 查询数据表的所有数据
SELECT * FROM user;
# 查询数据表的特定数据
SELECT column1, column2 FROM table_name WHERE condition;
```

```sql
# mysql sql语句 占位符？
insert into user(name, age) values (?,?) # 增
delete from user where id = ? # 删
select id, `name`, age from `user` where id > ? # 查
update user set age=? where id = ? # 改
```

## 连接

mysql数据库的连接流程与上文一致。

```go
import (
	"database/sql"

	_ "github.com/go-sql-driver/mysql"
)
// 定义一个全局对象db
var db *sql.DB

// 定义一个初始化数据库的函数
func initDB() (err error) {
	// DSN:Data Source Name
	dsn := "user:password@tcp(127.0.0.1:3306)/sql_test?charset=utf8mb4&parseTime=True"
	// 不会校验账号密码是否正确
	// 注意！！！这里不要使用:=，我们是给全局变量赋值，然后在main函数中使用全局变量db
	db, err = sql.Open("mysql", dsn)
	if err != nil {
		return err
	}
	// 尝试与数据库建立连接（校验dsn是否正确）
	err = db.Ping()
	if err != nil {
		return err
	}
	return nil
}

func main() {
	err := initDB() // 调用输出化数据库的函数
	if err != nil {
		fmt.Printf("init db failed,err:%v\n", err)
		return
	}
}
```

其它函数

```go
//SetMaxOpenConns 设置与数据库建立连接的最大数目。 如果n大于0且小于最大闲置连接数，会将最大闲置连接数减小到匹配最大开启连接数的限制。 如果n<=0，不会限制最大开启连接数，默认为0（无限制）。
func (db *DB) SetMaxOpenConns(n int)
//SetMaxIdleConns 设置连接池中的最大闲置连接数。 如果n大于最大开启连接数，则新的最大闲置连接数会减小到匹配最大开启连接数的限制。 如果n<=0，不会保留闲置连接。
func (db *DB) SetMaxIdleConns(n int)
```

## CRUD

### 查询

- `db.QueryRow()`
  - 单行查询`db.QueryRow()`执行一次查询，并期望返回最多一行结果（即Row）。
  - QueryRow总是返回非nil的值，直到返回值的Scan方法被调用时，才会返回被延迟的错误。（如：未找到结果）

```go
func (db *DB) QueryRow(query string, args ...interface{}) *Row

// 查询单条数据示例
func queryRowDemo() {
	sqlStr := "select id, name, age from user where id=?"
	var u user
	// 非常重要：确保QueryRow之后调用Scan方法，否则持有的数据库链接不会被释放
	err := db.QueryRow(sqlStr, 1).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("scan failed, err:%v\n", err)
		return
	}
	fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
}
```

`type Row struct{}`：

```go
// Close 查询之后关闭rows释放持有的数据库链接
func (rs *Rows) Close() error
// ColumnTypes 返回结果数据中所有的列的信息
func (rs *Rows) ColumnTypes() ([]*ColumnType, error)
// Columns 返回所有的列名
func (rs *Rows) Columns() ([]string, error)
// Err 返回查询过程中的遇到错误
func (rs *Rows) Err() error
// Next 遍历结果集，每次读取一行，返回True表示还有数据，否则已经读到最后一行
func (rs *Rows) Next() bool
// NextResultSet 有多个结果集，准备好下一个结果集来读取，有下一个结果集返回True
func (rs *Rows) NextResultSet() bool
// Scan 将当前行的结果数据拷贝出来，然后将它们放到后面跟着的这些变量里
func (rs *Rows) Scan(dest ...any) error
```

- `db.Query()`
  - 多行查询`db.Query()`执行一次查询，返回多行结果（即Rows）。
  - 一般用于执行select命令。参数args表示query中的占位参数。

```go
func (db *DB) Query(query string, args ...interface{}) (*Rows, error)
                    
// 查询多条数据示例
func queryMultiRowDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	rows, err := db.Query(sqlStr, 0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	// 非常重要：关闭rows释放持有的数据库链接
	defer rows.Close()

	// 循环读取结果集中的数据
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
}
```

`type Rows struct{}`：

```go
// Close 查询之后关闭rows释放持有的数据库链接
func (rs *Rows) Close() error
// ColumnTypes 返回结果数据中所有的列的信息
func (rs *Rows) ColumnTypes() ([]*ColumnType, error)
// Columns 返回所有的列名
func (rs *Rows) Columns() ([]string, error)
// Err 返回查询过程中的遇到错误
func (rs *Rows) Err() error
// Next 遍历结果集，每次读取一行，返回True表示还有数据，否则已经读到最后一行
func (rs *Rows) Next() bool
// NextResultSet 有多个结果集，准备好下一个结果集来读取，有下一个结果集返回True
func (rs *Rows) NextResultSet() bool
// Scan 将当前行的结果数据拷贝出来，然后将它们放到后面跟着的这些变量里
func (rs *Rows) Scan(dest ...any) error
```

- 其他

```go
// 使用上下文，如指定截止时间，取消操作，或一些上下文的值
QueryContext
QueryRowContext
```

### 插入

- `Exec`

  - 插入、更新和删除操作都使用`Exec`方法。

  - Exec执行一次命令（包括查询、删除、更新、插入等），返回的Result是对已执行的SQL命令的总结。参数args表示query中的占位参数。


> // LastInsertId returns the integer generated by the database in response to a command. Typically this will be from an "auto increment" column when inserting a new row. Not all databases support this feature, and the syntax of such statements varies.

```go
func (db *DB) Exec(query string, args ...interface{}) (Result, error)

// 插入数据
func insertRowDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	ret, err := db.Exec(sqlStr, "王五", 38)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	theID, err := ret.LastInsertId() // 新插入数据的id
	if err != nil {
		fmt.Printf("get lastinsert ID failed, err:%v\n", err)
		return
	}
	fmt.Printf("insert success, the id is %d.\n", theID)
}
```

### 更新数据

>  // RowsAffected returns the number of rows affected by an update, insert, or delete. Not every database or database driver may support this.

```go
// 更新数据
func updateRowDemo() {
	sqlStr := "update user set age=? where id = ?"
	ret, err := db.Exec(sqlStr, 39, 3)
	if err != nil {
		fmt.Printf("update failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("update success, affected rows:%d\n", n)
}
```

### 删除数据

```go
// 删除数据
func deleteRowDemo() {
	sqlStr := "delete from user where id = ?"
	ret, err := db.Exec(sqlStr, 3)
	if err != nil {
		fmt.Printf("delete failed, err:%v\n", err)
		return
	}
	n, err := ret.RowsAffected() // 操作影响的行数
	if err != nil {
		fmt.Printf("get RowsAffected failed, err:%v\n", err)
		return
	}
	fmt.Printf("delete success, affected rows:%d\n", n)
}
```

## MySQL预处理

### 什么是预处理？

普通SQL语句执行过程：

1. 客户端对SQL语句进行占位符替换得到完整的SQL语句。
2. 客户端发送完整SQL语句到MySQL服务端
3. MySQL服务端执行完整的SQL语句并将结果返回给客户端。

预处理执行过程：

1. 把SQL语句分成两部分，命令部分与数据部分。
2. 先把命令部分发送给MySQL服务端，MySQL服务端进行SQL预处理。
3. 然后把数据部分发送给MySQL服务端，MySQL服务端对SQL语句进行占位符替换。
4. MySQL服务端执行完整的SQL语句并将结果返回给客户端。

### 为什么要预处理？

1. 优化MySQL服务器重复执行SQL的方法，可以提升服务器性能，提前让服务器编译，一次编译多次执行，节省后续编译的成本。
2. 避免SQL注入问题。

### Go实现MySQL预处理

`database/sql`中使用下面的`Prepare`方法来实现预处理操作。

```go
// Prepare creates a prepared statement for later queries or executions. Multiple queries or executions may be run concurrently from the returned statement. The caller must call the statement's Close method when the statement is no longer needed.
// Prepare uses context.Background internally; to specify the context, use PrepareContext.
func (db *DB) Prepare(query string) (*Stmt, error)
```

`Prepare`方法会先将sql语句发送给MySQL服务端，返回一个准备好的状态用于之后的查询和命令。返回值可以同时执行多个查询和命令。

```go
// 预处理查询示例
func prepareQueryDemo() {
	sqlStr := "select id, name, age from user where id > ?"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	rows, err := stmt.Query(0)
	if err != nil {
		fmt.Printf("query failed, err:%v\n", err)
		return
	}
	defer rows.Close()
	// 循环读取结果集中的数据
	for rows.Next() {
		var u user
		err := rows.Scan(&u.id, &u.name, &u.age)
		if err != nil {
			fmt.Printf("scan failed, err:%v\n", err)
			return
		}
		fmt.Printf("id:%d name:%s age:%d\n", u.id, u.name, u.age)
	}
}
```

插入、更新和删除操作的预处理十分类似，这里以插入操作的预处理为例：

```go
// 预处理插入示例
func prepareInsertDemo() {
	sqlStr := "insert into user(name, age) values (?,?)"
	stmt, err := db.Prepare(sqlStr)
	if err != nil {
		fmt.Printf("prepare failed, err:%v\n", err)
		return
	}
	defer stmt.Close()
	_, err = stmt.Exec("小王子", 18)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	_, err = stmt.Exec("沙河娜扎", 18)
	if err != nil {
		fmt.Printf("insert failed, err:%v\n", err)
		return
	}
	fmt.Println("insert success.")
}
```

### SQL注入问题

**我们任何时候都不应该自己拼接SQL语句！**

这里我们演示一个自行拼接SQL语句的示例，编写一个根据name字段查询user表的函数如下：

```go
// sql注入示例
func sqlInjectDemo(name string) {
	sqlStr := fmt.Sprintf("select id, name, age from user where name='%s'", name)
	fmt.Printf("SQL:%s\n", sqlStr)
	var u user
	err := db.QueryRow(sqlStr).Scan(&u.id, &u.name, &u.age)
	if err != nil {
		fmt.Printf("exec failed, err:%v\n", err)
		return
	}
	fmt.Printf("user:%#v\n", u)
}
```

此时以下输入字符串都可以引发SQL注入问题：

```go
sqlInjectDemo("xxx' or 1=1#")
sqlInjectDemo("xxx' union select * from user #")
sqlInjectDemo("xxx' and (select count(*) from user) <10 #")
```

**补充：**不同的数据库中，SQL语句使用的占位符语法不尽相同。

|   数据库   |  占位符语法  |
| :--------: | :----------: |
|   MySQL    |     `?`      |
| PostgreSQL | `$1`, `$2`等 |
|   SQLite   |  `?` 和`$1`  |
|   Oracle   |   `:name`    |

## Go实现MySQL事务

### 什么是事务？

事务：一个最小的不可再分的工作单元；通常一个事务对应一个完整的业务(例如银行账户转账业务，该业务就是一个最小的工作单元)，同时这个完整的业务需要执行多次的DML(insert、update、delete)语句共同联合完成。A转账给B，这里面就需要执行两次update操作。

在MySQL中只有使用了`Innodb`数据库引擎的数据库或表才支持事务。事务处理可以用来维护数据库的完整性，保证成批的SQL语句要么全部执行，要么全部不执行。

### 事务的ACID

通常事务必须满足4个条件（ACID）：原子性（Atomicity，或称不可分割性）、一致性（Consistency）、隔离性（Isolation，又称独立性）、持久性（Durability）。

|  条件  |                             解释                             |
| :----: | :----------------------------------------------------------: |
| 原子性 | 一个事务（transaction）中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行过程中发生错误，会被回滚（Rollback）到事务开始前的状态，就像这个事务从来没有执行过一样。 |
| 一致性 | 在事务开始之前和事务结束以后，数据库的完整性没有被破坏。这表示写入的资料必须完全符合所有的预设规则，这包含资料的精确度、串联性以及后续数据库可以自发性地完成预定的工作。 |
| 隔离性 | 数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。事务隔离分为不同级别，包括读未提交（Read uncommitted）、读提交（read committed）、可重复读（repeatable read）和串行化（Serializable）。 |
| 持久性 | 事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。 |

### 事务相关方法

Go语言中使用以下三个方法实现MySQL中的事务操作。 开始事务

```go
func (db *DB) Begin() (*Tx, error)
func (db *DB) BeginTx() (*Tx, error)
// Tx的方法和上面DB的方法差不多，但是效用只作用于当前事务的范围
```

提交事务

```go
func (tx *Tx) Commit() error
```

回滚事务

```go
func (tx *Tx) Rollback() error
```

### 事务示例

下面的代码演示了一个简单的事务操作，该事物操作能够确保两次更新操作要么同时成功要么同时失败，不会存在中间状态。

```go
// 事务操作示例
func transactionDemo() {
	tx, err := db.Begin() // 开启事务
	if err != nil {
		if tx != nil {
			tx.Rollback() // 回滚
		}
		fmt.Printf("begin trans failed, err:%v\n", err)
		return
	}
	sqlStr1 := "Update user set age=30 where id=?"
	ret1, err := tx.Exec(sqlStr1, 2)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec sql1 failed, err:%v\n", err)
		return
	}
	affRow1, err := ret1.RowsAffected()
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec ret1.RowsAffected() failed, err:%v\n", err)
		return
	}

	sqlStr2 := "Update user set age=40 where id=?"
	ret2, err := tx.Exec(sqlStr2, 3)
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec sql2 failed, err:%v\n", err)
		return
	}
	affRow2, err := ret2.RowsAffected()
	if err != nil {
		tx.Rollback() // 回滚
		fmt.Printf("exec ret1.RowsAffected() failed, err:%v\n", err)
		return
	}

	fmt.Println(affRow1, affRow2)
	if affRow1 == 1 && affRow2 == 1 {
		fmt.Println("事务提交啦...")
		tx.Commit() // 提交事务
	} else {
		tx.Rollback()
		fmt.Println("事务回滚啦...")
	}

	fmt.Println("exec trans success!")
}
```