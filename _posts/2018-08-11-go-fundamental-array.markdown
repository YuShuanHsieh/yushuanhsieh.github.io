---
layout: single
title:  "Go Fundamental - Array"
date:   2018-08-11 08:00:00 +0800
categories: Go
---
## 前言
最近在學 Go，而之前有一些 C 的基礎，因此在學到 Go 的 Array 時，發現它和 C 的 Array  差異性蠻大的，所以特別記錄下來。

## Array
Go
```go
students := [3]string{"student1", "student2", "student3"}
```
C
```c
char *students[3] = {"student1", "student2", "student3"}
```
### Go's arrays are values.
首先，最大的不同點是： Go 的 Array 是一個 Value，也就是 `students` 這個 Variable 就代表整個 Array 。反之， C 在宣告 Array 後， `students` 是代表指向 Array 第一個 element 的 pointer，而不是整個 Array。

```c
// C
char *students[3] = { "student1", "student2", "student3" };
char **p;
p = students; // `students` represents the first element of array.
printf (*p); // Output: student1
```
這也意味著， 在 Ｃ 中， Array type 是無法進行 assign.

> C - Assignment operator shall have a modifiable lvalue as its left operand. A modifiable lvalue is an lvalue that does not have array type.


```c
// C
char* copy[3];
copy = student; //Error: assignment to expression with array type.
```

而 Go 的 Array 是一個 Value，所以可以進行 assign.

```go
var copy [3]string
copy = students
fmt.Println(copy) //Output: ["student1", "student2", "student3"]
```

不過，由於 Go 的 Array 是一個 Value，所以**當作為 function 的傳入參數，或著 assign 給其他 Array時，會 copy Array 中的所有 elements**，也就是裡面的 element memory address 是完全不一樣的，因此即使在 function 內修改 element 的值，並不會影響到既有的 Array。

```go
func main() {
  students := [3]string{"student1", "student2", "student3"}
  changeName(students) //Output: [student4 student2 student3]
  fmt.Println(students) // Output: [student1 student2 student3]
}

func changeName(students [3]string) {
 students[0] = "student4"
 fmt.Println(students)
}
```

> Go's arrays are values. An array variable denotes the entire array; it is not a pointer to the first array element (as would be the case in C). This means that when you assign or pass around an array value you will make a copy of its contents. (To avoid the copy you could pass a pointer to the array, but then that's a pointer to an array, not an array.)

### There are major differences between the ways arrays work in Go and C. In Go,
最後引用官方 Effective Go 提到的不同點。
1. Arrays are values. Assigning one array to another copies all the elements.
2. In particular, if you pass an array to a function, it will receive a copy of the array, not a pointer to it.
3. The size of an array is part of its type. The types [10]int and [20]int are distinct.

## Reference
1. [Effective Go](https://golang.org/doc/effective_go.html#arrays)
2. [Go Slices: usage and internals](https://blog.golang.org/go-slices-usage-and-internals)
