[English Version](README.md)

这个仓库提供将Golang的结构体转化为`map`的函数。它支持:
1. 使用`tag`去定义结构体中的域(field)的名字。如果没有指定，就使用结构体的域的名字。
2. 域(field)可以自定义自己的转化成map的方法。这个自定义的方法要有`(string,interface{})`作为输出，其中`string`作为map的键(key)，`interface`作为map的值。

并且，会跳过没有导出的域，空指针，和没有标签的域。

## 标签
目前，支持3种标签
- '-':忽略当前这个域
- 'omitempty' : 当这个域的值为空，忽略这个域
- 'dive' : 递归地遍历这个结构体，将所有字段作为键

## 例子
举个例子，
```go
type User struct {
	Name      string       `map:"name,omitempty"`        // string
	Github    GithubPage   `map:"github,dive,omitempty"` // struct dive
	NoDive    StructNoDive `map:"no_dive,omitempty"`     // no dive struct
	MyProfile Profile      `map:"my_profile,omitempty"`  // struct implements its own method
}

type GithubPage struct {
	URL  string `map:"url"`
	Star int    `map:"star"`
}

type StructNoDive struct {
	NoDive int `map:"no_dive_int"`
}

type Profile struct {
	Experience string    `map:"experience"`
	Date       time.Time `map:"time"`
}

// its own toMap method
func (p Profile) StructToMap() (key string, value interface{}) {
	return "time", p.Date.Format(timeLayout)
}
```
使用
```go
res, err := structmap.StructToMap(&user, tag, methodName)
```
转化为
```go
map[string]interface{}{
	"name":    "user",
	"no_dive":  map[string]int{"no_dive_int": 1},
    // dive struct field
	"url":     "https://github.com/liangyaopei",
	"star":    1,
    // customized method
	"time":    "2020-07-21 12:00:00",
}
```

## 性能
与使用`json`包的`marshal`/`unmarshal`比较，这个函数有更好的性能
```go
func BenchmarkStructToMapByJson(b *testing.B) {
	user := newBenchmarkUser()
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		data, _ := json.Marshal(&user)
		m := make(map[string]interface{})
		json.Unmarshal(data, &m)
	}
}

func BenchmarkStructToMapByToMap(b *testing.B) {
	user := newBenchmarkUser()
	tag := "json"
	methodName := ""
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		struct_to_map.StructToMap(&user, tag, methodName)
	}
}
```
使用 `StructToMap`的结果
```sh
$ go test ./... -bench=BenchmarkStructToMapByToMap  -benchmem -run=^$ -count=10

BenchmarkStructToMapByToMap-4            1322222               960 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1225311               965 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1201177               947 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1308039               895 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1201592               936 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1351603               882 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1361508               878 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1355869               876 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1353205              1151 ns/op             488 B/op         14 allocs/op
BenchmarkStructToMapByToMap-4            1278057              1487 ns/op             488 B/op         14 allocs/op
``` 
使用 `json` 包的结果
```sh
$ go test ./... -bench=BenchmarkStructToMapByJson  -benchmem -run=^$ -count=10 
BenchmarkStructToMapByJson-4      383409              3171 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      314690              3297 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      363764              3009 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      365799              3122 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      361111              3057 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      335830              3176 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      347206              3069 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      380973              3067 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      373834              3000 ns/op             992 B/op         30 allocs/op
BenchmarkStructToMapByJson-4      373597              3206 ns/op             992 B/op         30 allocs/op
```