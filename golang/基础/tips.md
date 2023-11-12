- 返回中声明变量

```go
// 可以直接在返回类型中声明变量，这样函数中就不用短声明:=
// 并且可以直接return
func getOne(id int) (u userDB, err error) {
   u = userDB{}
   err = db.QueryRow("select id, name, age from sqltest.user").Scan(&u.id, &u.name, &u.age)
   return
}
```

- 变量遮蔽

```go
var db *sql.DB
func main() {
	......
	var err error
	db, err = sql.Open("mysql", dsn)
    // 用db, err := sql.Open("mysql", dsn)则发生了变量遮蔽，main中的db变成了局部变量
	......
}
```

- if时短声明变量

```go
if err := tx.Rollback(); err != nil {        
 log.Fatalf("unable to back: %v", err)
```

- 
