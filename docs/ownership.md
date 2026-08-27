# Rust 所有权基础

> 所有权是 Rust 管理内存的核心规则：不依赖垃圾回收，也能在编译期保证大部分内存安全。

## 三条规则

1. Rust 中的每个值都有一个所有者。
2. 同一时间只能有一个所有者。
3. 所有者离开作用域时，值会被自动清理。

```rust
fn main() {
    let text = String::from("hello");
    println!("{text}");
} // text 离开作用域，String 自动释放
```

## 移动与复制

`String` 等拥有堆内存的类型赋值时会发生 **move**，原变量不再可用：

```rust
let first = String::from("hello");
let second = first;
// println!("{first}"); // 编译错误：value borrowed here after move
```

实现 `Copy` 的简单类型会按值复制，例如整数和布尔值：

```rust
let a = 10;
let b = a;
println!("{a} {b}");
```

## 借用

不转移所有权时，可以借用一个值。普通引用默认不可修改：

```rust
fn length(text: &String) -> usize {
    text.len()
}

let text = String::from("hello");
println!("{}", length(&text));
println!("{text}");
```

可变引用用 `&mut`，并且同一时间只能存在一个可变引用：

```rust
fn append_world(text: &mut String) {
    text.push_str(", world");
}

let mut text = String::from("hello");
append_world(&mut text);
```

借用规则可以简化为：

- 可以同时拥有多个不可变引用；或
- 只能拥有一个可变引用；
- 引用必须始终有效，不能引用已经离开作用域的值。

## 切片

切片是对连续数据的借用，不拥有数据：

```rust
fn first_word(text: &str) -> &str {
    text.split_whitespace().next().unwrap_or("")
}

let text = String::from("hello rust");
let word = first_word(&text);
assert_eq!(word, "hello");
```

使用 `&str` 而不是 `&String`，通常可以同时接受 `String` 和字符串字面量。

## 生命周期初见

生命周期描述引用必须保持有效的范围。很多时候编译器可以自动推断；当函数返回某个输入引用时，生命周期参数可以明确表达它们之间的关系：

```rust
fn longer<'a>(left: &'a str, right: &'a str) -> &'a str {
    if left.len() >= right.len() { left } else { right }
}
```

`'a` 不是延长数据寿命，而是约束返回引用不能超过相关输入引用的有效范围。

## 与 rustc 的关系

所有权、借用和生命周期检查发生在编译期。代码不满足规则时，`rustc` 会拒绝生成最终程序；可以结合错误码说明查看具体原因：

```bash
rustc --explain E0382
rustc --explain E0499
```

进一步学习可以阅读 Rust 官方书的[所有权章节](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)。
