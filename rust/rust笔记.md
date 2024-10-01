# rust笔记

## 前言:

1. 有垃圾回收的语言,垃圾回收意在回收堆上的内存,栈上的内存超过作用域之后会自动回收

2. 栈中所有数据都必须占用已知且固定的大小,堆上数据访问速度快是因为每次访问数据是不用搜索,总是访问栈顶,而对内存分配时需要通过指针去访问(堆中找到一块内存,标记为已使用,并返回一个表示该位置的指针

## 1.所有权概念:

1. Rust中每一个值都有一个被称为所有者的变量

2. 值在任意时刻有且仅有一个所有者

3. 当所有者离开作用域,这个值将被丢弃

## 2.移动,克隆与拷贝

手动内存释放存在三个问题

1. **早释放**会导致出现无效变量
2. **晚释放/不释放**会导致内存泄漏
3. **二次释放**会导致内存污染,可能会导致潜在的安全漏洞

rust解决上述问题的方法

1. rust本身在变量超过作用域之后自动调用了一个特殊函数--drop来释放内存

2. 当堆内存数据s被赋值给s1时,s将不再有效,从而避免**二次释放**(堆内存赋值实际上是将指针进行赋值,并不是完整的在堆上创建一份新的数据),这个在rust中称为**移动**

**克隆** 如果我们确实需要深度复制堆上的数据,二部仅仅是栈上的数据,我们就需要使用到克隆,如下代码可以正确运行

```rust
let s1 = String::from("hello");
let s2 = s1.clone();
println!("s1 = {}, s2 = {}", s1, s2);
```

使用clone时需要注意 该操作对资源耗费非常大

只在栈上的数据: **拷贝**,如下代码

```rust
let x = 5;
let y = x;
println!("x = {}, y = {}", x, y);//将x赋值给y后y依旧可以正常访问
```



**扩展**

Rust 有一个叫做 `Copy` trait 的特殊注解，可以用在类似整型这样的存储在栈上的类型上（第十章详细讲解 trait）。如果一个类型拥有 `Copy` trait，一个旧的变量在将其赋值给其他变量后仍然可用。Rust 不允许自身或其任何部分实现了 `Drop` trait 的类型使用 `Copy` trait。如果我们对其值离开作用域时需要特殊处理的类型使用 `Copy` 注解，将会出现一个编译时错误。要学习如何为你的类型增加 `Copy` 注解，请阅读附录 C 中的 [“可派生的 trait”](https://www.bookstack.cn/read/trpl-zh-cn-1.41/appendix-03-derivable-traits.html)。

**所有权与函数**

将值传递给函数在语义上与变量赋值类似,向函数传递参数时,传递的变量可能会移动或者复制

1. 对于**堆内存数据**作为参数传递给函数之后,该变量在函数调用后将不可再访问,变量会在函数结束时被回收,想要该问题有两种办法:
   1. 将变量从函数中返回出来并接收(通过返回值交还所有权)
   2. 每次都传进来再返回出去很麻烦,所以引入了**引用**的概念,该方法已一个对象的引用作为参数而不是获取值的所有权

2. 对于**栈内存数据**作为参数传递给函数之后,该变量在函数调用后可以继续访问

## 引用与借用

**引用**: 以一个对象的引用作为参数而不是获取值的所有权(允许使用但不获取所有权),在参数定义时使用&,如下所示

```rust
fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);
    println!("The length of '{}' is {}.", s1, len);
}
fn calculate_length(s: &String) -> usize {
    s.len()
}
```

**借用**: 我们将获取引用作为函数参数称为 **借用**（*borrowing*）

**注**:如果尝试直接修改借用的值会直接报错,正如变量默认是不可改变的,引用也一样,(默认)不允许修改引用的值

**可变引用**

使用mut标记,如下:

```rust
fn main() {
    let mut s = String::from("hello");
    change(&mut s);
}
fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```



**注**: 可变引用有一个很大的限制,在同一个作用域中的同一个变量只能有一个可变引用,通过这种方式解决**数据竞争**问题,如下代码将报错:

```rust
let mut s = String::from("hello");
let r1 = &mut s;
let r2 = &mut s;
println!("{}, {}", r1, r2);
```

如下代码正确执行:

```rust
let mut s = String::from("hello");
{
    let r1 = &mut s;
} // r1 在这里离开了作用域，所以我们完全可以创建一个新的引用
let r2 = &mut s;
```

**注**:引用的作用域从声明的地方开始一直持续到最后一次使用为止,所以有了如下代码:

```rust
//以下代码是错误的
let mut s = String::from("hello");
let r1 = &s; // 没问题
let r2 = &s; // 没问题
let r3 = &mut s; // 大问题
println!("{}, {}, and {}", r1, r2, r3);
```

```rust
//以下代码正确运行
let mut s = String::from("hello");
let r1 = &s; // 没问题
let r2 = &s; // 没问题
println!("{} and {}", r1, r2);
// 此位置之后 r1 和 r2 不再使用
let r3 = &mut s; // 没问题
println!("{}", r3);
```

**总结**:

1. 不允许同时存在多个可变引用
2. 允许可变引用和不可变引用同时存在,前提是不可变引用已经超出了作用域



## Slice

**概念** slice允许引用集合中一段连续的元素片段,而不是引用整个集合,字符串字面值是切片



## 结构体Struct

**注**: 

1. 在实例化结果体后需要改变结构体字段值,需要使用**mut**关键字标记,rust不允许只将某个字段标记为可变

2. 结构体拥有所有权,即使所有字段类型结尾u8
3. 如果我们在一个结构体定义的前面使用了 `pub` ，这个结构体会变成公有的，但是这个结构体的字段仍然是私有的。

```rust
# struct User {
#     username: String,
#     email: String,
#     sign_in_count: u64,
#     active: bool,
# }
#
let mut user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};
user1.email = String::from("anotheremail@example.com");
```

**结构体绑定方法**:rust中允许同一个结构体定义多个`Impl`块

```rust
# #[derive(Debug)]
# struct Rectangle {
#     width: u32,
#     height: u32,
# }
#
impl Rectangle {
    fn area(&self) -> u32 {
        self.width * self.height
    }
}
impl Rectangle {
    fn can_hold(&self, other: &Rectangle) -> bool {
        self.width > other.width && self.height > other.height
    }
}
```

**关联函数**: 允许在impl块中定义不以self作为参数的函数,这被称为**关联函数**,使用::调用,例如String::from

```rust
# #[derive(Debug)]
# struct Rectangle {
#     width: u32,
#     height: u32,
# }
#
impl Rectangle {
    fn square(size: u32) -> Rectangle {
        Rectangle { width: size, height: size }
    }
}
```

## 枚举与模式匹配

**枚举**:枚举允许你通过列举可能的 **成员**（*variants*） 来定义一个类型

**注**:如果我们将枚举设为公有，则它的所有成员都将变为公有。

```rust
# enum Message {
#     Quit,
#     Move { x: i32, y: i32 },
#     Write(String),
#     ChangeColor(i32, i32, i32),
# }
#
impl Message {
    fn call(&self) {
        // 在这里定义方法体
    }
}
let m = Message::Write(String::from("hello"));
m.call();
/*
Quit 没有关联任何数据。
Move 包含一个匿名结构体。
Write 包含单独一个 String。
ChangeColor 包含三个 i32
*/
```

**Option**: `Option`是标准库定义的一个枚举,它编码了一个非常普遍的场景,即一个值要么有值要么没值

**扩展**: Rust 并没有很多其他语言中有的空值功能。**空值**（*Null* ）是一个值，它代表没有值。在有空值的语言中，变量总是这两种状态之一：空值和非空值。空值的问题在于当你尝试像一个非空值那样使用一个空值，会出现某种形式的错误。为此，Rust 并没有空值，不过它确实拥有一个可以编码存在或不存在概念的枚举。这个枚举是 `Option`,只要一个值不是 `Option` 类型，你就 **可以** 安全的认定它的值不为空。这是 Rust 的一个经过深思熟虑的设计决策，来限制空值的泛滥以增加 Rust 代码的安全性。

```rust
//Option在标准库中的定义
enum Option<T> {
    Some(T),
    None,
}
```

**match**: `match`表达式是用于处理枚举的控制流结构,它会根据枚举的成员运行不同的代码,这些代码可以使用匹配到的值中的数据

```rust
# #[derive(Debug)]
# enum UsState {
#    Alabama,
#    Alaska,
# }
#
# enum Coin {
#    Penny,
#    Nickel,
#    Dime,
#    Quarter(UsState),
# }
#
fn value_in_cents(coin: Coin) -> u8 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter(state) => {
            println!("State quarter from {:?}!", state);
            25
        },
    }
}
```

Rust中的匹配必须是**穷尽的**:必须例举出`enum`中的所有情况,如果指向列举一部分,需要显示的添加`_`(**通配符**)

```rust
let some_u8_value = 0u8;
match some_u8_value {
    1 => println!("one"),
    3 => println!("three"),
    5 => println!("five"),
    7 => println!("seven"),
    _ => (),
}
```

**`if let` 简单控制流**: 如果指向匹配一种结果时,上述语法太过于复杂,所以提供了`if let`来处理只匹配一个模式的值而忽略其他模式的情况(`if let` 是 `match` 的一个语法糖)

```rust
# #[derive(Debug)]
# enum UsState {
#    Alabama,
#    Alaska,
# }
#
# enum Coin {
#    Penny,
#    Nickel,
#    Dime,
#    Quarter(UsState),
# }
# let coin = Coin::Penny;
let mut count = 0;
if let Coin::Quarter(state) = coin {
    println!("State quarter from {:?}!", state);
} else {
    count += 1; //else 块中的代码与 match 表达式中的 _ 分支块中的代码相同
}
```



## 包、crate和模块

**crate**: `crate`是一个二进制项或者库

**包**: `包`是提供一系列功能的一个或者多个`crate`,一个`包`会包含有**一个**`Cargo.toml`阐述如何去构建这些`crate`

`规则`: `包`中所包含的内容有几条规则来确立,一个`包`中**最多只能包含一个**`库crate`,`包`中可以包含**任意多个**`二进制crate`,`包`中至少包含**一个**`crate`,无论是库的还是二进制的

Cargo 遵循的一个约定：*src/main.rs* 就是一个与包同名的二进制 crate 的 crate 根。同样的，Cargo 知道如果包目录中包含 *src/lib.rs*，则包带有与其同名的库 crate，且 *src/lib.rs* 是 crate 根

如果一个包同时含有 *src/main.rs* 和 *src/lib.rs*，则它有两个 crate：一个库和一个二进制项，且名字都与包相同。通过将文件放在 *src/bin* 目录下，一个包可以拥有多个二进制 crate：每个 *src/bin* 下的文件都会被编译成一个独立的二进制 crate。

**模块**: *模块* 让我们可以将一个 crate 中的代码进行分组，以提高可读性与重用性。模块还可以控制项的 *私有性*，即项是可以被外部代码使用的（*public*），还是作为一个内部实现的内容，不能被外部代码使用（*private*）。



## 常见集合

**注**: 集合指向的数据是储存在堆上的

### vector

**创建**

```rust
//创建空的vector
let v: Vec<i32> = Vec::new();
//创建包含初始值的vector
let v = vec![1, 2, 3];
```

**添加元素**

```rust
let mut v = Vec::new();
v.push(5);
v.push(6);
v.push(7);
v.push(8);
```

**读取**

```rust
let v = vec![1, 2, 3, 4, 5];
let third: &i32 = &v[2];//当引用一个不存在的元素时会直接Panic,类似数组下边越界
println!("The third element is {}", third);
match v.get(2) {//访问不存在的下标不会Panic,而是会返回None
    Some(third) => println!("The third element is {}", third),
    None => println!("There is no third element."),
}
```

**Vector引用**: 在vector的结尾增加新元素时,在没有足够空间将所有元素依次相邻存放的情况下,可能会要求分配新内存并将老的元素拷贝到新的空间中

```rust
//如下代码报错
let mut v = vec![1, 2, 3, 4, 5];
let first = &v[0];//push操作一旦重新分配内存,这里将指向被释放的内存,rust不允许这样做
v.push(6);
println!("The first element is: {}", first);

//错误信息
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:6:5
  |
4 |     let first = &v[0];
  |                  - immutable borrow occurs here
5 |
6 |     v.push(6);
  |     ^^^^^^^^^ mutable borrow occurs here
7 |
8 |     println!("The first element is: {}", first);
  |                                          ----- immutable borrow later used here
```



**遍历**:

```rust
let mut v = vec![100, 32, 57];
for i in &mut v {
    *i += 50; //在进行 += 运算符之前不许使用解引用 * 获取 中的值
}
```



**在vector中存储多种类型** (借用枚举实现,因为枚举中元素都是同种枚举类型

```rust
enum SpreadsheetCell {
    Int(i32),
    Float(f64),
    Text(String),
}
let row = vec![
    SpreadsheetCell::Int(3),
    SpreadsheetCell::Text(String::from("blue")),
    SpreadsheetCell::Float(10.12),
];
```



### String

#### 字符串相加

**使用+相加**

```rust
let s1 = String::from("Hello, ");
let s2 = String::from("world!");
let s3 = s1 + &s2; // 注意 s1 被移动了，不能继续使用
println!("s1 is {}",s2);
```

**注**: +只能将String与&str相加,并不能将两个String相加,并且String必须在前,&str必须在后,+操作使用起来像下边这样

```rust
fn add(self, s: &str) -> String {
```



**使用formart宏**

```rust
let s1 = String::from("tic");
let s2 = String::from("tac");
let s3 = String::from("toe");
let s = format!("{}-{}-{}", s1, s2, s3);
```



#### 字符串索引

**注**: rust中字符串**不支持**通过下标索引

```rust
//以下代码无法运行
let s1 = String::from("hello");
let h = s1[0];
```

**字符串slice**:使用时尤其要注意不同字符串占用的字节数不同

```rust
let hello = "Здравствуйте";
let s = &hello[0..4];//这里每个字符占用两字节(中文占用3字节),如果换成&hello[0..1]会直接报错如下信息

thread 'main' panicked at 'byte index 1 is not a char boundary; it is inside 'З' (bytes 0..2) of `Здравствуйте`', src/libcore/str/mod.rs:2188:4
```

### 哈希map

```rust
use std::collections::HashMap;//需要手动导入
let mut scores = HashMap::new();
scores.insert(String::from("Blue"), 10);//此操作会直接覆盖旧值
scores.entry(String::from("Yellow")).or_insert(50);//存在则不操作,不存在则插入
scores.entry(String::from("Blue")).or_insert(50);
println!("{:?}", scores);
```

**注**: `Entry` 的 `or_insert` 方法在键对应的值存在时就返回这个值的可变引用，如果不存在则将参数作为新值插入并返回新值的可变引用



## 错误处理

### panic!与不可恢复错处

1. 当出现`panic`时,程序默认会开始`展开`,这意味着Rust会回溯栈并清理遇到的每一个函数的数据

2. 另一种选择是直接**终止**,这会不清理数据就退出程序,程序所消耗的内存需要由操作系统来清理

   **注**: 如果想要项目的最终二进制文件越小越好,可以再`Cargo.toml`文件中`[profile]`文件中加入 `panic = 'abort'`,这可以由展开切换为终止,如果想要在release模式中 panic 时直接终止：

   ```rust
   [profile.release]
   panic = 'abort'
   ```

#### RUST_BACKTRACE

有时我们调用`panic!`时,错误信息中指向的代码并不是我们所编写的代码

```rust
fn main() {
    let v = vec![1, 2, 3];
    v[99];
}

报错信息如下:
$ cargo run
   Compiling panic v0.1.0 (file:///projects/panic)
    Finished dev [unoptimized + debuginfo] target(s) in 0.27s
     Running `target/debug/panic`
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', libcore/slice/mod.rs:2448:10
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

这时我们可以在执行`cargo run`之前将`RUST_BACKTRACE`设置成任意一个不为0的值来获取 执行到目前位置所有被调用的函数列表,为了获取带有这些信息的`backtrace`,必须启动`debug`标识,当不使用`--release`参数运行`cargo build`或`cargo run`时`debug`标识会默认启用

```rust
$ RUST_BACKTRACE=1 cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/panic`
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 99', libcore/slice/mod.rs:2448:10
stack backtrace:
   0: std::sys::unix::backtrace::tracing::imp::unwind_backtrace
             at libstd/sys/unix/backtrace/tracing/gcc_s.rs:49
   1: std::sys_common::backtrace::print
             at libstd/sys_common/backtrace.rs:71
             at libstd/sys_common/backtrace.rs:59
   2: std::panicking::default_hook::{{closure}}
             at libstd/panicking.rs:211
   3: std::panicking::default_hook
             at libstd/panicking.rs:227
   4: <std::panicking::begin_panic::PanicPayload<A> as core::panic::BoxMeUp>::get
             at libstd/panicking.rs:476
   5: std::panicking::continue_panic_fmt
             at libstd/panicking.rs:390
   6: std::panicking::try::do_call
             at libstd/panicking.rs:325
   7: core::ptr::drop_in_place
             at libcore/panicking.rs:77
   8: core::ptr::drop_in_place
             at libcore/panicking.rs:59
   9: <usize as core::slice::SliceIndex<[T]>>::index
             at libcore/slice/mod.rs:2448
  10: core::slice::<impl core::ops::index::Index<I> for [T]>::index
             at libcore/slice/mod.rs:2316
  11: <alloc::vec::Vec<T> as core::ops::index::Index<I>>::index
             at liballoc/vec.rs:1653
  12: panic::main
             at src/main.rs:4
  13: std::rt::lang_start::{{closure}}
             at libstd/rt.rs:74
  14: std::panicking::try::do_call
             at libstd/rt.rs:59
             at libstd/panicking.rs:310
  15: macho_symbol_search
             at libpanic_unwind/lib.rs:102
  16: std::alloc::default_alloc_error_hook
             at libstd/panicking.rs:289
             at libstd/panic.rs:392
             at libstd/rt.rs:58
  17: std::rt::lang_start
             at libstd/rt.rs:74
  18: panic::main
```

### Result与可恢复的错误

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let f = File::open(".hello.txt").unwrap_or_else(|error| {
        if error.kind() == ErrorKind::NotFound{
            File::create("hello.txt").unwrap_or_else(|error| {
                panic!("create file err is {}",error);
            })
        }else {
            panic!("open file err is {}",error);
        }
    });
}
```



## 泛型,trait与生命周期

**函数/方法中引用生命周期计算**: 编译器采用三条规则来判断引用何时不需要明确的注释,如果编译器在检查完这三条规则之后任然存在没有计算出生命周期的引用,编译器将会报错

1. 每一个是引用的参数都有它自己的生命周期
2. 如果只有一个`输入生命周期`参数,那么它将被赋予**所有**`输出生命周期`
3. 如果有**多个**`输入生命周期`参数并且其中一个参数是`&selt`或`&mut selt`,说明是个对象的方法,那么**所有**`输出生命周期`被赋予`self`生命周期



**静态生命周期**: `'static`其生命周期能够存活于**整个程序期间**,所有的**字符串字面值**都拥有 `'static`生命周期,我们也可以显式的标记出来:

```rust
let s: &'static str = "I have a static lifetime.";
```



## 迭代器与闭包

### 闭包

**概念**: Rust的闭包是可以**保存进变量**或**作为参数传递给其他函数**的**匿名函数**,并可以访问**被定义的作用域的变量**



### 迭代器

1. `iter`方法生成一个**不可变引用**的迭代器
2. `into_iter`**获取所有权**
3. `iter_mut`获取**可变引用**

**消费适配器**: 一个`消费适配器`的例子是`sum`方法,调用该方法之后不允许再使用`v1_iter`,因为调用`sum`之后它会**获取迭代器的所有权**

```rust
#[test]
fn iterator_sum() {
    let v1 = vec![1, 2, 3];
    let v1_iter = v1.iter();
    let total: i32 = v1_iter.sum();
    assert_eq!(total, 6);
}
```





## 智能指针

### `Box<T>`指向堆上的数据

`Box<T>`允许将一个值放在**堆**上而不是放在**栈**上,**栈**上保存**指向堆的指针**

有了`Box<T>`就允许我们创建递归类型,假设我们定义了一个枚举,枚举中某个元素包含自身,普通情况下是无法编译的,因为编译器无法计算递归占用的空间大小,这个时候我们便可以用`Box<T>`**包裹自身**,因为指针的大小是确定的,代码如下:

```rust
//下面代码无法编译
enum List {
    Cons(i32, List),
    Nil,
}
use crate::List::{Cons, Nil};
fn main() {
    let list = Cons(1, Cons(2, Cons(3, Nil)));
}
//报错如下:
error[E0072]: recursive type `List` has infinite size
 --> src/main.rs:1:1
  |
1 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
2 |     Cons(i32, List),
  |               ----- recursive without indirection
  |
  = help: insert indirection (e.g., a `Box`, `Rc`, or `&`) at some point to
  make `List` representable
  
  

//如下代码正常编译
enum List{
    Coin(i32,Box<List>),
    Nil,
}
use crate::List::{Coin,Nil};
fn main() {
    let list = Coin(1,Box::new(Coin(2,Box::new(Nil))));
}
```

存入`Box<T>`的数据之所以可以直接使用`*`解引用拿到存入的值是因为`Box<T>`实现了 `deref`

#### 解引用强制多态

**概念**: 将实现了`Deref`的类型的解引用转换为先执行`deref`方法,再执行解引用

假设我们定义了一个结构体,我们需要取结构体中的一个值进行解引用,一旦我们为这个结构体实现了`Deref` trait,那么对结构体进行解引用时会在解引用之前先调用`deref`方法,看起来像`*(y.deref())`



#### `Drop` trait

**概念**: 对于智能指针第二个重要的`trait`是`Drop`,它允许我们在值要离开作用域时执行一些代码逻辑,并且`drop`的执行顺序**与定义时相反**

**注**: `Rust`中**不允许**我们手动调用`Drop trait`的`drop`方法,即使是我们实现了自己的`Drop trait`,`Drop trait`中的`drop`方法只能在值离开作用域时自动调用(至于为什么这么做,其实也能想明白,如果我们手动调用了`Drop trait`的`drop方法`,那么Rust再自动执行一次,相当于内存**二次回收**了),如果我们想要在**超出作用域之前**回收内存,需要使用**标准库提供的`std::mem::drop`**



### `Rc<T>`引用计数智能指针

`Rc<T>`用于当我们希望在堆上分配一些内存供程序的多个部分读取,而且无法在编译时就确定程序的哪一部分最后结束使用(如果知道哪部分最后一个使用的话,就可以另最后使用者为所有者,正常的所有权规则就可以在编译时生效),`Rc<T>`允许一个值有多个所有者,引用计数则确保只要任何所有者依然存在其值也保持有效

**eg**: 假设我们在实现`图`结构时,多个边可能会拥有同一个节点,该节点知道没有边指向它时才应该被释放



### `RefCell<T>`和内部可变性模式

**内部可变性** 是Rust中的一个设计模式,它允许你即使在**有不可变引用**时也**可以改变数据**,`RefCell<T>`用于确信代码遵守借用规则,而编译器不能理解和确定的时候,其内部使用了`unsafe`代码来模糊Rust的可变性和借用规则



### Rc<T>和RefCell<T>结合来拥有多个可变数据所有者

`Rc<T>`允许对相同数据有多个所有者,不过只能提供数据的不可变访问,如果用`Rc<T>`来存储`RerCell<T>`就可以得到**多个所有者并且可以修改的值**



### 引用循环与内存泄漏

**循环引用例子**: 定义两个包含`Rc<T>`智能指针的变量,然后通过`Rc::clone`将a赋值给b,再将b赋值给a,变量定义时引用计数`Rc::strong_count()`为1,互相赋值后,a与b的`Rc::strong_count`均变成了2,在变量超出作用域之后执行`Drop trait`的`drop方法`,对于引用计数只能指针`Rc<T>` `drop方法`会将引用计数进行 `-1`操作,2-1 = 1 ,两个变量的引用数量都为1,不为0,所以内存就不会被回收



**弱引用**: `Rc::clone`会增加`Rc<T>`的`strong_count`,`strong_count`为0时内存才会被清理,可以通过**`Rc::downgrade`**来创建**弱引用**,调用`Rc::downgrade`会得到`Weak<T>`类型的智能指针,同时使`weak_count` +1,区别在于`weak_count`无须引用计数为0就能使`Rc`<T>`被清理,强引用`的目的在于共享`Rc<T>`的所有权,而`弱引用`不属于所有权,不会造成`引用循环`,因为`弱引用`的循环会在相关`强引用`技术为0时被打断





## 无畏并发

1. rust**不支持继承**,而是通过`trait`可以带有默认实现的方式实现的类似继承的功能,只需要非默认实现的方法,就可以使用带有默认实现的方法
2. 



## 高级特性

### 不安全的Rust

**概述**: 有以下五类可以再不安全Rust中进行而不能用于安全Rust的操作

1. 解引用裸指针(裸指针类型: `*const T`和`*mut T`)
2. 调用不安全的函数或方法
3. 通过`extern`进行FFI(外部函数调用),因为其他语言不会执行Rust的规则,并且Rust无法检查他们
4. 访问或修改可变静态变量
5. 实现不安全trait
6. 访问 `union`的字段

**注意**: `unsafe`并不会关闭借用检查器或禁用任何其他Rust安全检查,如果在不安全代码中使用引用,他仍会被检查

#### 裸指针

**特性: **

	1. 允许忽略借用规则,可以**同时拥有** **可变** 与 **不可变引用**  或 **多个**指向相同位置的 **可变引用**
 	2. 不保证指向有效的内存
 	3. 允许为空
 	4. 不能实现任何自动清理功能

**注意**: **可以**在**安全代码**中**创建**裸指针,**不可以**在不安全代码块之外**解引用**裸指针

#### 不安全函数或方法

**概念**: 不安全函数和方法与常规函数和方法十分类似,**开头有一个额外的`unsafe`关键字**

**不安全函数**: 不安全函数也是有效的`unsafe`块,所以在不安全函数中进行另一个不安全操作时**不用添加额外的`unsafe`标识**



#### 访问或修改可变静态变量

**静态变量与常量区别**: 

1. **常量**在编译阶段编译器会尽可能将其内联到代码中,所以在不同地方对同一个常量的引用**不能保证引用到相同的内存地址**
2. **静态变量**不会被内联,在在整个程序中,全局变量**只有一个**,也就是说**所有的引用都会指向一个相同的地址**,并且`静态变量`**可以定义为可变**,**访问和修改**  **可变**`静态变量`都是**不安全的**, **读取**  **不可变**`静态变量`是**安全的**,之所以Rust认为`静态变量`不安全是因为**难以保证不存在数据竞争**(例如多线程时就会出现数据竞争,这个时候我们需要使用`Arc<T>`智能指针)



#### 实现不安全`trait`

**不安全`trait`**: 在定义`trait`时使用`unsafe`标记



#### 访问`union`联合体中的字段



### 高级`trait`





### 宏

**宏与函数区别**: 

1. `宏`能接受不同数量的参数,而`函数`则不行
2. `宏`可以再编译器翻译代码之前展开,例如宏可以再一个给定类型上实现`trait`,`函数`则不行,因为函数是在运行时调用,同是`trait`需要在编译时实现

#### 声明宏

**简介**: 类似于`match`表达式,使用`macro_rules!`声明,例如`vec!`宏,假设我们定义了如下代码:

```rust
let v: Vec<u32> = vec![1, 2, 3];
```

`vec!`实际生成的代码是这样的

```rust
//实际上进行了三次匹配生成了如下代码
let mut temp_vec = Vec::new();
temp_vec.push(1);
temp_vec.push(2);
temp_vec.push(3);
temp_vec
```

`vec!`简略表示如下

```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```



#### 用于从属性生成代码的过程宏

**过程宏**: 过程宏接收Rust代码作为输入,在这些代码上进行操作,然后产生另外一些代码作为输出,而非像`声明宏`那样匹配对应模式,通过一部分代码替换占位符生成另一部分代码

`过程宏` 以**`TokenStream作为输入`**,并以**`TokenStream`作为输出**,需要依赖**`syn`**和**`quote`**库,`syn`用于将Rust语法解析成一个可以操作的数据结构(构建语法树),`quote`将`syn`解析的数据结构转换成Rust代码

**过程宏分类**:

1. 自定义派生(derive)
2. 类属性
3. 类函数

**自定义派生(derive)**

通过`derive`来统一实现一个`trait`

通常情况下我们想要使用`trait`中的某个功能,需要使用对象实现这个`trait`,也可以通过`derive`过程宏统一实现`trait`,使用时通过`#[derive(dirive名称)]`标记, **注意**: `derive`只能用于**结构体**和**枚举**

**类属性宏**

常见的用处是做web请求解析时,将请求方式,请求路径和处理程序绑定

**类函数宏**

使用方式和调用函数类似,常见的有像`sql!`,这个宏会解析SQL中的语句并检查其正确性