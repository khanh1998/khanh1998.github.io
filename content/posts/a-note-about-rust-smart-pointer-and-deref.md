---
date: '2024-12-08T13:33:55+07:00'
draft: false
title: 'A Note About Rust Smart Pointer and Deref'
tags: ['CS']
---
this article assumes you have some basic knowledge about [Rust smart pointers](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html).
## the dereference (*) operator
*dereference* operator `*` is used for getting the actual value from a reference:
```rust
let a: i32 = 5;    
let b: &i32 = &a;
assert_eq!(*b, 5);
```
in the above example, `b` is a reference to `a`, `*b` will return the value that `b` pointing to.
## the smart pointers
### `Box<T>`
here is an example of how you can init and modify the value of a Box<T>:
```rust
let mut b: Box<i32> = Box::new(6);
*b = 7;
assert_eq!(*b, 7);
```
in the above example, I init a box with value 6 inside, then I changed the value to 7, and check the value inside the box with assert to ensure the change is affected.

in the first example of this post, we apply the dereference to a reference which is `&i32`, but the data type of b now is `Box<i32>`, how can we apply to dereference to a `Box<i32>`?

the answer is the `Box<T>` is implemented the trait [Deref](https://doc.rust-lang.org/alloc/boxed/struct.Box.html#impl-Deref-for-Box%3CT%2C%20A%3E) and [DerefMut](https://doc.rust-lang.org/alloc/boxed/struct.Box.html#impl-DerefMut-for-Box%3CT%2C%20A%3E), so when you write `*b`, the compiler implicitly translates it to `*(b.deref())` if you want to get the value or `*(b.deref_mut())` if you want to change the value.

if you have a `Box<T>`, the `deref` method returns a `&T`, and `deref_mut` returns a `&mut T`. after calling the `deref` method, you now have a reference to the actual value, therefore you can use the dereference operator on these references.

the above snippet can be explicitly rewritten as below:
```rust
let mut b: Box<i32> = Box::new(6);

let c: &mut i32 = b.deref_mut();
*c = 7;
    
let d: &i32 = b.deref();
assert_eq!(*d, 7);
```
firstly, we create a new `box` to hold the value 6. then we call `deref_mut` to get a `&mut i32`, in order to change the value inside the box. finally, we get a read-only reference `&i32` to the value inside the box, by calling `deref`.
### `Rc<T>`
`Rc<T>` is quite similar to `Box<T>`, except that you can only read data in Rc, you cannot edit it. because Rc only implements the trait `Deref`, not implement `DerefMut`. Therefore, you can only call the `deref` method on Rc only, there is no such method `deref_mut`.

```rust
// implicitly
let rc1 = Rc::new(3);
assert_eq!(*rc1, 3);
// explicitly
let a = rc1.deref();
assert_eq!(*a, 3);
```
### `RefCell<T>`
```rust
// wrong code
let ref_cell: RefCell<i32> = RefCell::new(5);
*ref_cell = 7;
assert_eq!(*ref_cell, 7);
```
if you write code like the above example, you will get this error: type `RefCell<{integer}>` cannot be dereferenced. it’s because `RefCell<i32>` doesn’t implement either `Deref` or `DerefMut` traits. you can check this out on the docs of [RefCell](https://doc.rust-lang.org/core/cell/struct.RefCell.html).

the way `RefCell<T>` works is a bit different from `Box<T>`, to read or modify data inside `RefCell` you need to explicitly call method `borrow` or `borrow_mut` respectively.

`borrow` will return a `Ref<T>` which implements trait `Deref`, and `borrow_mut` will return a `RefMut<T>` which implements both `DerefMut` and `Deref`.

because `Ref<T>` and `RefMut<T>` implements `Deref`, so we can use *dereference* operator on them.

so you can write code like this:
```rust
// implicit
let ref_cell = RefCell::new(5);
*(ref_cell.borrow_mut()) = 7;
assert_eq!(*(ref_cell.borrow()), 7);

// a bit explicit
let ref_cell = RefCell::new(5);
*(ref_cell.borrow_mut().deref_mut()) = 7;
assert_eq!(*(ref_cell.borrow().deref()), 7);
```
or, if you want more explicitly, I have this code:
```rust
// wrong code
let ref_cell = RefCell::new(5);

let mut a: RefMut<i32> = ref_cell.borrow_mut();
let b: &mut i32 = a.deref_mut();
*b = 8;

let a: Ref<i32> = ref_cell.borrow();
let b: &i32 = a.deref();
assert_eq!(*b, 8);
```
if you run the above code, you will get this error: `already mutably borrowed: BorrowError`.

it’s because, in line number 2, you call `borrow_mut`, and in line number 3 you call `deref_mut` to get a mutability reference.

in line number 5, you call `borrow` in order to read the data inside `RefCell`, which is forbidden by Rust. you can’t have a read-only reference to the value that currently has a mutability reference to prevent the race condition.
so you can rewrite it as below:
```rust
let ref_cell = RefCell::new(5);
    {
        let mut a: RefMut<i32> = ref_cell.borrow_mut();
        let b: &mut i32 = a.deref_mut();
        *b = 8;
    } // borrow_mut will be drop here, at the end of block
    {
        // there are no mutability reference to the value, so we can borrow.
        let a: Ref<i32> = ref_cell.borrow();
        let b: &i32 = a.deref();
        assert_eq!(*b, 8);
    }
```
in the above code, the `borrow_mut` and `deref_mut` are put in the first block, so at the end of the first block, the mutability reference is dropped, there is no mutability reference to the value now. that is why in the second block, you can (read-only) borrow as much as you want from the `RefCell`.

## references
https://doc.rust-lang.org/book/ch15-00-smart-pointers.html