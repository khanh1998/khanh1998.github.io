---
date: '2023-09-10T13:57:02+07:00'
draft: false
title: 'Why would sometimes 0.1 + 0.2 equals to 0.3, but sometimes it’s not in Go'
tags: ['go','cs']
---
You may have heard about *0.1 + 0.2 != 0.3* already, it’s a common issue in most programming languages. The reason behind that is the floating point (IEEE 754) as the computer can’t represent exactly a decimal in binary. By utilizing the floating point, the computer can hold a very large decimal, but the trade-off is it now can only represent the approximate of the true value.

However, in this post, I will not explain how floating point works but something else – [constant](https://go.dev/blog/constants). 

## Untyped constant

First, let’s take a look at my [Go sample code](https://go.dev/play/p/fdBVcZqFyB1), throughout this post, I will try to explain what is going on.
```go
// https://go.dev/play/p/fdBVcZqFyB1
package main
 
import "fmt"
 
func main() {
    // first example
    a := 0.1 + 0.2
    b := 0.3
    fmt.Println(a == b) // true
 
    // second example
    x := float64(0.1) + float64(0.2)
    y := float64(0.3)
    fmt.Println(x == y) // false
}
```
it’s weird right? sometimes 0.1 + 0.2 is equal to 0.3, but sometimes it is not.

The thing that caused the above issue which named *untyped constant*. In Go (and some other languages), an untyped contact can have *arbitrary precision*, simply put, *untyped constants* are much more **precise** than floating point.

There is a Pi constant in the `math` library:
```go
Pi  = 3.14159265358979323846264338327950288419716939937510582097494459
```
The precision of the constant `Pi` is much more than normal `float64`, if you assign `Pi` to a variable, it will lose some precision.
```go
package main
 
import (
    "fmt"
    "math"
)
 
func main() {
    pi := math.Pi
    fmt.Println(pi) // 3.141592653589793
}
```
With the *untyped constant* like the above `Pi`, the language allows you to do math calculations with higher precision, like the below example:
```go
package main
 
import (
    "fmt"
    "math"
)
 
const r = 589.897962323243 // untyped const
 
func main() {
    ra := r
    fmt.Println(math.Pi * r * r / 180)   // result: 6073.389853674304
    fmt.Println(math.Pi * ra * ra / 180) // result: 6073.389853674303
}
```
In the expression of `math.Pi * r * r / 180`, there are three *untyped constants*, `math.Pi`, `r`, and `180`. In the second expression, `math.Pi * ra * ra / 180`, we know that `math.Pi` and `180` are *untyped constants*, but `ra` is a `float64` variable and it causes the result less precision because the computer can’t represent exactly the decimal of `589.897962323243` in binary.

Now back to the first code snippet. `0.1`, `0.2`, and `0.3` are *untyped constants*. These *untyped constants* allow you to make high-precision calculations. If you think `0.1`, `0.2`, and `0.3` are floating point, you are totally wrong. Like the first example, `0.1 + 0.2 = 0.3`, you know that floating points can only represent an approximate true value, so no way `0.1 + 0.2` equals `0.3`. So floating point has nothing to do with the calculation of `0.1 + 0.2`. Thanks to the *untyped constants*, it is the reason you get the expected in the first example.

An untyped constant can:
- have very higher precision than floating point
- It never overflows – meaning you can store a very large number

But when you assign an *untyped constant* to a *variable* like `float64`, it will lose some precision, and the *untyped constant* value needs to fit in a `float64`, if the constant value is too big, there will be an error.

This leads us to the second example, `float64(0.1) + float64(0.2)`. We know that `0.1` and `0.2` are *untyped constants*. but when we assign them to `float64`, we lose some precision. It means `float64(0.1)` doesn’t equal `0.1`, likewise to `0.2` or `0.3`. so when we add two float representations of `0.1` and `0.2`, the result is not `0.3`, but a value that approximates `0.3`. For the `float64(0.3)`, the language can only represent an approximate value of `0.3` as well.
- `float(0.1)` -> approximate value of `0.1`
- `float(0.2)` -> approximate value of `0.2`
- `float(0.1) + float(0.2)` -> you are adding two approximate values -> approximate value of `0.3` **(1)**
- `float(0.3)` -> approximate value of `0.3` **(2)**
- **(1)** and **(3)** are two approximate values of three, but they can be different.

## fmt.Println
By default, to be able to print the constant, the constant needs to be assigned to a variable, and that causes a loss of some precision. Take a look at the below example:
```go
package main
 
import (
    "fmt"
)
 
func main() {
    fmt.Println(0.3)                           // 0.3
    fmt.Printf("%f\n", 0.30000000000000001)    // 0.300000
    fmt.Printf("%.16f\n", 0.30000000000000001) // 0.3000000000000000
    fmt.Printf("%.54f\n", 0.30000000000000001) // 0.299999999999999988897769753748434595763683319091796875
    fmt.Printf("%.54f\n", 0.3)                 // 0.299999999999999988897769753748434595763683319091796875
}
```
All of the constants in the above example are *untyped*, meaning it has very high precision, but the untyped constants like `0.3` need to be converted to `float64`, so that the `fmt.Println` can print it to the console, and the conversion from *untyped constant* to `float64` will cause some minor imprecision.

- `fmt.Println(0.3)` -> `0.3` in the console, that’s weird, right? `0.3` is an *untyped constant*, the `fmt.Println` needs to cast `0.3` to a `float64`, which will make the value lose some precision, That means it can’t print `0.3`, it has to be some approximate value right?
- `fmt.Printf(“%f\n”, 0.30000000000000001)` -> `0.300000`. So that seems like `fmt.Println` round the input `float64`. And default it rounds to **6 digits** after the decimal point.
- `fmt.Printf(“%.16f\n”, 0.30000000000000001)` -> `0.3000000000000000`. If you round up the float to 16 digits, you get `0.3`. That is true, Let’s try rounding up `0.2999999999999999`, and you will get `0.3`.
- `fmt.Printf(“%.54f\n”, 0.30000000000000001)` -> `0.299999999999999988897769753748434595763683319091796875`. With 54 digits after the decimal point, you now know the closest value to `0.3` can be represented in binary.

## References
https://en.wikipedia.org/wiki/IEEE_754
https://go.dev/blog/constants
https://go.dev/ref/spec#Constants
https://go.dev/ref/spec#Constant_expressions
https://stackoverflow.com/questions/38982278/how-does-go-perform-arithmetic-on-constants
https://stackoverflow.com/questions/57511935/what-is-the-purpose-of-arbitrary-precision-constants-in-go?rq=3
https://stackoverflow.com/questions/38806491/why-doesnt-left-bit-shifting-by-64-overflow-in-golang
https://stackoverflow.com/questions/58403028/floating-point-precision-golang
https://stackoverflow.com/questions/42153747/why-does-0-1-0-2-get-0-3-in-google-go