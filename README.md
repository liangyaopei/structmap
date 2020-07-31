[![Go Report Card](https://goreportcard.com/badge/github.com/liangyaopei/structmap)](https://goreportcard.com/report/github.com/liangyaopei/structmap)
[![GoDoc](https://godoc.org/github.com/liangyaopei/structmap?status.svg)](http://godoc.org/github.com/liangyaopei/structmap)

[中文版说明](README_zh.md)

This repo provides a function convert a struct in Golang to a map(unmarshal a struct to a map).It supports
1. use tag to define the name of a filed in struct when converted in map. If no tag is specified, it will the name of field as key.
2. a filed in struct can customize its to map method.
To note that the customized method should have `(string,interface{})` as output, with the `string` as key in map and `interface{}` as value in map.

Also, it skips unexported filed and nil pointer.


## Tags
For now, it supports 3 tags:
- '-' means ignoring this field
- 'omitempty' means ignoring this field when it is empty
- 'dive' means recursively traversing the struct, converting every filed in struct to be a key in map


## Example
For example,
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
	NoDive int
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
by using 
```go
res, err := struct_to_map.StructToMap(&user, tag, methodName)
```
can be converted to be
```go
map[string]interface{}{
	"name":    "user",
	"no_dive": StructNoDive{NoDive: 1},
    // dive struct field
	"url":     "https://github.com/liangyaopei",
	"star":    1,
    // customized method
	"time":    "2020-07-21 12:00:00",
}
```

## Performance
Compared with using `marshal`/`unmarshal` function in `json` package, this repo has better performance.
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
The result of using `StructToMap`
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
The result of using `json` package
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