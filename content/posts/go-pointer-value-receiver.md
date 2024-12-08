---
date: '2024-05-21T14:24:03+07:00'
draft: false
title: 'The pointer receiver and value receiver in Go are just syntax sugar'
tags: ['go']
---
If you are a Gopher, you probably heard some advice that goes like this: when you edit a value then use a pointer receiver, when you read use a value receiver. That’s not always correct and you should be careful when using them.

## Issue

Today I encountered an issue which can be simplified by the bellow code snippet:
```go
package main
 
import (
    "fmt"
    "time"
)
 
type Counter struct {
    Count int
}
 
// Method with pointer receiver
func (c *Counter) Increment() {
    go func() {
        for i := 0; i < 10; i++ {
            c.Count++
            time.Sleep(1 * time.Second)
        }
    }()
}
 
// Method with value receiver
func (c Counter) Display() {
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println("counter value: ", c.Count)
            time.Sleep(1 * time.Second)
        }
    }()
}
 
func main() {
    c := Counter{Count: 0}
    c.Display()
    c.Increment()
 
    time.Sleep(15 * time.Second)
    fmt.Println("counter final value: ", c.Count)
}
```
I have a struct with two methods, `Increment` uses the pointer receiver to change the property of the struct, and `Display` uses the value receiver to read the value from the struct. Here is the result:
```
➜ go run .
counter value:  1
counter value:  1
counter value:  1
counter value:  1
counter value:  1
counter value:  1
counter value:  1
counter value:  1
counter value:  1
counter value:  1
counter final value:  11
```
Both methods concurrently edit and read the counter, but the method of Display can only show the counter’s initial value. Why does that happen?

It turns out that because of the value receiver of the method `Display`. When you call that method, it will create a **copy** of the `Counter` struct and operate on that copy. The `Increment` method changes the count on the original `Struct`, while the `Display` method reads from a **copy** of the Counter struct. Basically, two methods now are operating on two different struct instances.

To fix the issue, just use the pointer receiver in the `Display` method, and now both methods are operating on the same copy of the struct.

An another pattern is when you by mistake call an *pointer receiver method* inside an *value receiver method*, like this:
```go
package main

import (
	"fmt"
)

type Counter struct {
	mData map[int]int
	sData []int
}

// Method with pointer receiver
func (c *Counter) Put(i int) {
	c.mData[i] = i
	c.sData = append(c.sData, i)
	fmt.Println("Put mData=", c.mData)
	fmt.Println("Put sData=", c.sData)
}

// Method with value receiver
func (c Counter) Do() {
	// call api and get data
	i := 2
	c.Put(i)
}

func main() {
	c := Counter{mData: map[int]int{1: 1}, sData: []int{1}}
	c.Do()
	fmt.Println("main mData=", c.mData)
	fmt.Println("main sData=", c.sData)
}
```
The output:
```
Put mData= map[1:1 2:2]
Put sData= [1 2]
main mData= map[1:1 2:2]
main sData= [1]
```
This case is quite confusing when number 2 appears in the map, but disappear in the list 😂 The fix is similar to previous case, make the `Do` method become pointer receiver.
## Sugar syntax

In Go, *methods* are just sugar syntax. To make things easier to understand, I rewrite the two methods as below:
```go
// Function with pointer args
func Increment(c *Counter) {
    go func() {
        for i := 0; i < 10; i++ {
            c.Count++
            time.Sleep(1 * time.Second)
        }
    }()
}
 
// Function with value args
func Display(c Counter) {
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println("counter value: ", c.Count)
            time.Sleep(1 * time.Second)
        }
    }()
}
```
## Method expression
Go also has something called `Method Expressions`, I can rewrite the main function as below:
```go
func main() {
    c := Counter{Count: 1}
 
    display := Counter.Display
    display(c)
    inc := (*Counter).Increment
    inc(&c)
 
    time.Sleep(15 * time.Second)
    fmt.Println("counter final value: ", c.Count)
}
```
From the code, you can see, that I can convert methods to functions, and pass the struct to the functions as the addition argument.

# References
https://go.dev/ref/spec#Method_expressions
https://golang.org/ref/spec#Method_values
https://www.reddit.com/r/golang/comments/6p1woc/are_gos_methods_just_syntactic_sugar