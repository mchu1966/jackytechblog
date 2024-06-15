+++ 
draft = false 
date = 2024-06-16T03:00:54+08:00 
title = 'String handling in Golang'
description = "" 
slug = "" 
authors = ["Jacky Cheng"] 
tags = ["Golang", "string"] 
categories = ["Note"] 
externalLink = "" 
series = [] 
+++

String Handling 在編程中非常常見。本文記錄了使用 Golang 處理 String 的常見方法。

# Golang 的 String 是什麼

> string is the set of all strings of 8-bit bytes, conventionally but not necessarily representing UTF-8-encoded text. A string may be empty, but not nil. Values of string type are immutable.

本質上，[string](https://pkg.go.dev/builtin#string) 是一串 byte (byte slice, 1 byte=8 bits)，而 Golang 的 [byte](https://pkg.go.dev/builtin#byte) 等於 uint8。

byte 按照 [UTF-8](https://en.wikipedia.org/wiki/ASCII) encode 後產生對應的 character (字元)。由於 1 byte 只有 8 bits，因此單使用 byte 並不能代表所有 UTF-8 encoded 字元，例如中文字、emoji 等，所以 Golang 另外有`rune` type 處理這些特殊字元。而 Golang 的 rune 等於 int32, 可想而知 run

從 [runtime/string.go](https://github.com/golang/go/blob/master/src/runtime/string.go#L232-L241) 可見 Golang runtime 對 string 的定義是由 byte pointer 跟一個 int 組成。

```go
type stringStruct struct {
	str unsafe.Pointer // underlying bytes
	len int            // number of bytes
}

// Variant with *byte pointer type for DWARF debugging.
type stringStructDWARF struct {
	str *byte
	len int
}
```

# String Handling

## `nil`

Golang 有自己一套對 null (空值) 的處理。對於 String type 而言，是沒有 nil 的，只有預設值`""`(empty string)。

```go
str := "hello"
fmt.Println(str == nil)
```

這段 code 會報錯:

```shell
invalid operation: str == nil (mismatched types string and untyped nil)
```

雖然可以通過 `str==""` 檢查 null string，但是對於某些情況而言，empty string 跟 null string 有其各自的意義，並不對等。因此會使用`*string` (pointer of string) 處理需要 null string 的情況。

```go
var strp *string
fmt.Println(strp == nil) // true
// fmt.Println(*strp)
// runtime error: invalid memory address or nil pointer dereference
var str = ""
strp = &str
fmt.Println(strp == nil) // false
fmt.Println(*strp) //
```

## `len()`

因為 string 是一串 bytes，即 byte slice。可以通過 built-in function `len()` 找出 string 的長度。

```go
var str = "Hello\n"
fmt.Println(len(str)) // 6
```

## String Literals

跨行的 string 有兩種表達方式，分別是使用`""`跟` `` `：

```go
var str = "Hello1\n2World3\n4!"
```

等於

```go
var str = `Hello1
2World3
4!`
```

## String concatenation

### operator `+`

也稱作 concatenation operator

```go
s := "Hello" + "World!"
fmt.Println(s) // HelloWorld!
```

### `fmt.Sprint`, `fmt.Sprintln`, `fmt.Sprintf`

[fmt.Sprintf](https://pkg.go.dev/fmt#Sprintf)

```go
s := fmt.Sprint("HelloWorld!")
fmt.Println(s) // HelloWorld!
s = fmt.Sprintln("Bye", "World", "~")
fmt.Println(s) // ByeWorld~
s = fmt.Sprintf("%s", "NiceWorld!")
fmt.Println(s) // NiceWorld!
```

`Sprint`亦可以將不同類型的 variable 轉換成 String：

```go
sli := []int{1,2,3}
str := fmt.Sprint(sli)
fmt.Println(str) // [1 2 3]
```

### `strings.Join()`

[strings.Join](https://pkg.go.dev/strings#Join) 背後是使用 strings.builder [實現](https://github.com/golang/go/blob/master/src/strings/strings.go#L452-L458)

```go
ss := []string{"Hello", "World", "~"}
s := strings.Join(ss, "")
fmt.Println(s) // HelloWorld~
```

### `bytes.Buffer`

[bytes.Buffer](https://pkg.go.dev/bytes#Buffer)

```go
var b bytes.Buffer
b.WriteString("Hello")
b.WriteString("World!")
fmt.Println(b.String()) // HelloWorld!
```

### 注意事項

每個方法的效能比較: [Go String Concat Performance](https://thefortunedays.com/articles/golang-string-concatenation-performance/)，作者給出的其中一個結論是

> Avoid memory allocation as much as we can

因為[string 是不可變的](https://go.dev/ref/spec#String_types)，當使用 `+` 或 `fmt` 方法時，會造成 memory allocation，[尤其對於較長的 string](https://github.com/golang/go/blob/master/src/cmd/compile/internal/walk/expr.go#L497-L511)。

相對而言，Buffer 的結構是

```go
type Builder struct {
	addr *Builder
	buf  []byte
}
```

Buffer 的 `WriteXxx` method 使用 [`append`](https://go.dev/ref/spec#Appending_and_copying_slices) 對 `b.buf` 進行操作，減少 memory allocation，提升效能。不過由於對同一個 memory location (slice) 進行操作，[重複使用同一個 Buffer](https://blog.wu-boy.com/2022/06/reuse-the-bytes-buffer-in-go/) 需要注意覆寫的問題。

## String convertion

使用[strconv](https://pkg.go.dev/strconv) package

```go
str := "1234"
v, _ := strconv.Atoi(str)
fmt.Printf("%T\n", v) // int
// Or, use "reflect", fmt.Println(reflect.TypeOf(v))
s := strconv.Itoa(v)
fmt.Printf("%T\n", s) // string
u, _ := strconv.ParseUint(str, 10, 32)
fmt.Printf("%T\n", u) // uint64
```

### Convert int slice into string

```go
a := []int{1,2,3,4}
str := strings.Trim(strings.Replace(fmt.Sprint(a), " ", ",", -1), "[]")
fmt.Print(str) // 1,2,3,4
```

## 特殊情況

### Remove the last character from a string

```go
ss := []string{"Hello", "World", "Peter", "Tom"}
var s string
for _, v := range ss{
  s = s + v + ", "
}
// s == "Hello, World, Peter, Tom, "
s = s[:i] + strings.Replace(s[i:], ", ", "", 1)
fmt.Println(s) // Hello, World, Peter, Tom
```

如果你知道要刪除的 substring 是什麼，可以使用`bytes.Buffer`:

```go
b.WriteString("Hello")
b.WriteString("World!")
b.Truncate(b.Len() - len("rld!"))
fmt.Println(b.String()) // HelloWo
```

### Create a random string

```go
letterRunes := []rune("3456789ABCEFGHJKLMNPQRSTXY")
func RandStringRunes(n int) string {
	b := make([]rune, n)
	for i := range b {
		b[i] = letterRunes[rand.Intn(len(letterRunes))]
	}
	return string(b)
}
```

### Handle full space character

使用`rune`可以處理更多的 UTF-8 字元，包含中文字、emoji 等全形字。

使用 byte slice 表示中文字時會發現，Golang 會使用多於一個 byte 去表示中文字元，例如:

```go
fmt.Println([]byte("Hello, 世界"))
// [72 101 108 108 111 44 32 228 184 150 231 149 140]
// 228 184 150 世
// 231 149 140 界
```

舉一個例子：`validateComment` 這個 function 會用 `*` 取代 comment 中的特定字眼，其中 comment 可以包含全形字。

使用`strings.Builder.WriteRune`就可以直接寫入全形字元。

（這個例子同時展示了，如何還原`ToLower`後的 String。）

```go
func validateComment(comment string) string {
    lowerComment := sensitive.Filter.Replace(strings.ToLower(comment), '*')
    var sb strings.Builder
    var runeCount int
    for _, runeValue := range lowerComment {
        if runeValue != '*' {
            _, err := sb.WriteRune([]rune(comment)[runeCount])
            if err != nil {
                log.Error(err)
            }
        } else {
            _, err := sb.WriteRune(runeValue)
            if err != nil {
                log.Error(err)
            }
        }
        runeCount++
    }
    defer sb.Reset()
    return sb.String()
}
// sensitive package: https://github.com/importcjj/sensitive
```

# Reference

- https://pkg.go.dev/builtin
- https://go101.org/article/string.html
- https://github.com/golang/go/blob/master/src/runtime/string.go
