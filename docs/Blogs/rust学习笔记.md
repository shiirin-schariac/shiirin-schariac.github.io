## 变量
加mut可变，注意引用时要加`&mut`

使用`:`标注类型，常量在声明时必须标注类型

e.g.`const THREE_HOURS_IN_SECONDS: u32 = 60 * 60 * 3;`

变量重定义会发生隐藏（shadowing），本质是在当前作用域下创造了一个同名的新变量。

严格来说rust不允许空变量存在，虽然在定义时可以暂时不给予变量值，但是调用该变量前必须赋予值否则会报错。

## 多个元素构成的复合类型
1. 元组

长度不可变、不同元素数据类型可以不同
```rust
fn main() {
      let tup = (500, 6.4, 1);

      let (x, y, z) = tup;
      println!("The value of y is: {y}");
}
```
可以使用`tup.0`访问不同下标的元素

2. 数组（=栈）

长度不可变、元素数据类型必须相同

`let a: [i32; 5] = [1, 2, 3, 4, 5];`

## 函数
声明时必须声明所有参数的数据类型。

语句没有返回值，表达式不加分号可直接在函数体中返回计算结果

指明返回类型的范例
```rust
  fn five() -> i32 {
      5
	}
```

## 控制流
if和else-if的返回值数据类型必须相同

let的右边可以使用if：`let number = if condition { 5 } else { 6 };

可以通过命名循环且break加关键字指定中断的循环

loop：无终止条件

for用于遍历集合的元素

## 所有权
关于堆和栈是如何分配内存的

> 栈中的所有数据都必须占用已知且固定的大小。在编译时大小未知或大小可能变化的数据，要改为存储在堆上。 堆是缺乏组织的:当向堆放入数据时，你要请求一定大小的空间。内存分配器(memory allocator)在堆的某处找到一块足够大的空位，把它标记为已使用，并返回一个表示该位置地址的指针(pointer)。这个过程称作在堆上分配内存。

栈：可以储存在栈中的数据类型都是已知大小的，并且当离开作用域时被移出栈，如果代码 的另一部分需要在不同的作用域中使用相同的值，可以快速简单地复制它们来创建一个新的独立实例。

所有权的目的：管理**堆**数据

**所有权的规则**

1. Rust 中的每一个值都有一个所有者(owner)。（assign）

2. 值在任一时刻有且只有一个所有者。（拷贝后原变量无效，见p79）

3. 当所有者(变量)离开作用域，这个值将被丢弃。

为了支持一个可变，可增长的数据，需要在堆上分配一块在编译时未知大小的内存来存放内容。这意味着:

1. 必须在**运行时**向内存分配器(memory allocator)请求内存。

2. 需要一个当我们处理完 String 时将内存返回给分配器的方法。

对于第二步，Rust 采取了一个不同的策略:内存在拥有它的变量离开作用域（该作用域结束/变量被传送至其他作用域->将指向堆的指针重新绑定给新的变量）后就被自动释放。

拷贝的过程为将指向堆的指针重新绑定给新的变量，原变量就此无效以保证不会发生内存的重复释放（也是为了保证值在任一时刻有且只有一个所有者）。且Rust 永远也不会自动创建数据的 “深拷贝”（深拷贝可以使用clone函数）。因此，任何自动的复制都可以被认为是对运行时性能影响较小的。

因为对于堆上的数据一般只做指针的重新绑定，所以任何可能导致堆上数据发生变化的行为都会使原变量失效（例如将字符串作为参数传入函数中）。**也就是说，指向堆的指针的位置一旦被变化（所有权发生转移），就会使原来绑定该指针的变量失效**

copy trait：如果一个类型实现了 Copy trait，那么一个旧的变量在将其赋值给其他变量后仍然可用（等价于创造了一个副本）。Rust 不允许自身或其任何部分实现了 Drop trait（即在作用域结束后自动释放内存）的类型使用 Copy trait。可使用copy trait的类型如下

- 所有整数类型，比如 u32 。

- 布尔类型，bool ，它的值是 true 和 false 。

- 所有浮点数类型，比如 f64 。
- 字符类型，char 。

- 元组，当且仅当其包含的类型也都实现 Copy 的时候。比如，(i32, i32) 实现了 

- Copy ，但(i32, String) 就没有。

## 引用
引用(reference)像一个指针，因为它是一个地址，我们可以由此访问储存于该地址的属于其他变量的数据。 与指针不同，引用确保指向某个特定类型的有效值，它们允许你使用值但不获取其所有权。

引用(默认)不允许修改引用的值。可以使用mut操作符使引用可修改被引用变量的值（并非对副本修改而是对本体修改）。

> 可变引用有一个很大的限制:如果你有一个对该变量的可变引用，你就不能再创建对该变量的引用，无论是可变引用还是不可变引用。（保证同一时间内有且只有一个可变引用）。
> 这个限制的好处是 Rust 可以在编译时就避免数据竞争。数据竞争(data race)类似于竞态条件，它可由这三个行为造成:
> 	
> 	• 两个或更多指针同时访问同一数据。 
> 	
> 	• 至少有一个指针被用来写入数据。  
 >	
> 	• 没有同步数据访问的机制。

注意一个引用的作用域从声明的地方开始一直持续到最后一次使用为止。

在具有指针的语言中，很容易通过释放内存时保留指向它的指针而错误地生成一个 悬垂指针 (dangling pointer)，所谓悬垂指针是其指向的内存可能已经被分配给其它持有者。相比之下，在 Rust 中编译器确保引用永远也不会变成悬垂状态:当你拥有一些数据的引用，编译器确保数据不会在其引用之前离开作用域。（在有引用的状态下被引用的数据不会被释放内存=被引用的数据被释放内存时引用也一起失效）

#### slice类型（本质为引用）
字符串slice：slice 的数据结构存储了 slice 的开始位置和长度：`let slice = &a[1..3];`

## 结构体（非Copy数据类型）
结构体定义了一个新的数据类型的命名空间，其中定义每一部分数据的名字和类型，我们称为字段(field)。Rust 并不允许只将某个字段标记为可变，除非将整个结构体实例标记为可变。

同其他 、任何表达式一样，我们可以在函数体的最后一个表达式中构造一个结构体的新实例，来隐式地 、返回这个实例。
```rust
fn build_user(email: String, username: String) -> User {
	User {
		active: true,  
		username: username,
		email: email,
	    sign_in_count: 1,
	}
}
```

结构体更新语法
```rust
let user2 = User {
	email: String::from("another@example.com"),
    ..user1
};// 使用结构体更新语法为一个 User 实例设置一个新的 email 值，不过其余值来自 user1 变量中实例的字段
```

结构更新语法就像带有 = 的赋值，因为它移动了数据，在这个例子中，总体上说我们在创建 user2 后不能就再使用 user1 （user1整个实例一起失效）了，因为 user1 的 username 字段中的 String 被移到 user2 中。可以使结构体存储被其他对象拥有的数据的引用，不过这么做的话需要用上生命周期 (lifetimes)。生命周期确保结构体引用的数据有效性跟结构体本身保持一致。如果你尝试在结构体中存储一个引用而不指定生命周期将是无效的。

元组结构体是结构体，但没有字段名称。可以通过和元组一样的方式调用数据。
类单元结构体(unit-like structs)因为它们类似于() ，即“元组类型”一节中提到的unit类型。类单元结构体常常在你想要在某个类型上实现 trait 但不需要在类型中存储数据的时候发挥作用。

#### 方法
```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle { // 为了使函数定义于 `Rectangle` 的上下文中，我们开始了一个 `impl` 块（`impl` 是 _implementation_ 的缩写），这个 `impl` 块中的所有内容都将与 `Rectangle` 类型相关联。
    fn area(&self) -> u32 {
        self.width * self.height
    }
}
```

方法(method)与函数类似:它们使用 fn 关键字和名称声明，可以拥有参数和返回值，同时包含在某处调用该方法时会执行的代码。不过方法与函数是不同的，因为它们在结构体的上下文中被定义，**并且它们第一个参数总是self**（一般是self的引用），它代表调用调用该方法的结构体实例。调用method的方法为`rect1.area()`。与字段同名的方法将被定义为只返回字段中的值（getter），而不做其他事情。

method可以可变地调用self。
> 在 C/C++ 语言中，有两个不同的运算符来调用方法:. 直接在对象上调用方法，而 - > 在一个对象的指针上调用方法，这时需要先解引用(dereference)指针。换句话 说，如果 object 是一个指针，那么 object->something() 就像 (\*object).something() 一样。rust不存在指针调用和实例调用的区别->不需要手动对指针解引用：
```rust
// p1是一个指向某实例的指针，以下代码是等效的
p1.distance(&p2);
(&p1).distance(&p2);
```

#### 关联函数
> 与方法的区别在于，方法是由实例调用（第一个参数必须为self），关联函数只是与该数据类型有关联的函数，并不作用于一个结构体的实例

关联函数可以用于定义构造函数，即返回类型为结构体。关联函数位于结构体的命名空间中:`::` 语法用于关联函数和模块创建的命名空间。

## 枚举-enums
枚举允许你通过列举可能的成员(variants)来定义一个类型。枚举定义了一个**数据类型**的命名空间，可以集合使用不同的数据类型实例化的成员->对不同的数据类型进行包装。结构体中也可以定义一个被定义的枚举的数据类型变量，变量的值为枚举中某个成员被
实例化后的实例。

```rust
enum IpAddr {
	V4(u8,u8,u8,u8),
	V6(String), // 也可以使用其他数据类型（包括自定义的结构体）初始化
	// 枚举中的成员初始化使用的数据类型不需要相同
}
```

枚举也可以使用`impl`来定义其上的method.

#### Option枚举
option枚举的成员一个有值，一个没值，从类型系统的角度来表达这个概念就意味着编译器需要检查是否处理了所有应该处理的情况。Rust 并没有空值，不过它确实拥有一个可以编码存在或不存在概念的枚举。这个枚举是Option\<T>，如下：
```rust
enum Option<T> {
    None,
	Some(T), 
}

fn main() {
    let some_number = Some(5);
	let some_char = Some('e');
	let absent_number: Option<i32> = None; // 定义absent_number为i32类型的option枚举中的不存在概念/无效值
}
```

\<T> 是一个泛型类型参数，意味着 Option 枚举的 Some 成员可以包含任意类型的数据，同时每一个用于 T 位置的具体类型使得 Option\<T> 整体作为不同的类型。因为 Option\<T> 和 T (这里 T 可以是任何类型)是不同的类型，编译器不允许像一个肯定有效的值那样使用Option\<T> 。在对 Option\<T> 进行运算之前必须将其转换为 T 。通常这能帮助我们捕获到空值最常见的问题之一:假设某值不为空但实际上为空的情况。

> 为了拥有一个可能为空的值，你必须要显式的将其放入对应类型的 Option\<T> 中。接着，当使用这个值时，必须明确地处理值为空的情况。只要一个值不是 Option\<T> 类型，你就可以安全地认定它的值不为空。

#### match控制流
match可以用于对一个枚举集合类型的变量进行其与枚举中的成员匹配，必须穷尽。可以使用通配符`other`来指代剩余的所有情况（必须放在最后）。也可以使用占位符`_`来表示不使用这个值，它可以匹配任意值而不绑定到该值。
```rust
fn value_in_cents(coin: Coin) -> u8 {
    match coin { //match 后跟一个表达式
	    Coin::Penny => {
		    println!("Lucky penny!");
		    1
	    }
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
	} 
}
```

```rust
fn main() {
	fn plus_one(x: Option<i32>) -> Option<i32> {
        match x {
	        None => None,
            Some(i) => Some(i + 1),
        }
	}
    let five = Some(5);
    let six = plus_one(five);
    let none = plus_one(None);
}
```

```rust
 fn main() {
    let dice_roll = 9;
    match dice_roll {
	    3 => add_fancy_hat(),
        7 => remove_fancy_hat(),
        _ => (),
    }
	fn add_fancy_hat() {}
	fn remove_fancy_hat() {} 
}
```

#### if-let简洁控制流
if let 语法获取通过等号分隔的一个模式和一个表达式。它的工作方式与match 相同，这里 的表达式对应 match 而模式则对应第一个分支。该语法不要求穷尽性检查。
```rust
let config_max = Some(3u8);
if let Some(max) = config_max {
    println!("The maximum is configured to be {}", max);
} // 模式是 Some(max) ，max 绑定为 Some 中的值。
```

## 模块系统
• **包**(Packages):Cargo 的一个功能，它允许你构建、测试和分享 crate。

• **Crates** :一个模块的树形结构，它形成了库或二进制项目。  

• **模块**(Modules)和 **use**:允许你控制作用域和路径的私有性。  

• **路径**(path):一个命名例如结构体、函数或模块等项的方式。

Rust提供将包分成多个 crate，将 crate 分成模块，通过指定绝对或相对路径从一个模块引用另一个模块中定义的项的方式，并通过这些方式组织模块系统。

> 一些关于模块组织的方法/规则：**从 crate 根节点开始**、**声明模块**、**声明子模块**、**模块中的代码路径**、**私有 vs 公用**、**`use` 关键字**
#### 包和crate
- crate 是Rust在编译时最小的代码单位，它有两种形式:
	
	1. 二进制项：二进制项可以被编译为可执行程序，比如一个命令行程序或者一个服务器。它们**必须有一个 main 函数**来定义当程序被执行的时候所需要做的事情。crate root 是一个源文件，构成你crate 的根模块。
	
	2. 库：库并没有 main 函数，它们也不会编译为可执行程序，它们提供一些诸如函数之类的东西，使其他项目也能使用这些东西。

- 包(package)是提供一系列功能的一个或者多个 crate。包中可以包含**至多一个**库 crate。包中可以包含任意多个二进制 crate，但是必须至少包含一个 crate(无论是库的还是二进制的)。

库或二进制项crate都有一个入口crate（crate根文件），对于一个库crate而言是*src/lib.rs*，在二进制项中一般位于*src/main.rs*，由这个入口crate开始，编译器将寻找需要编译的代码。

#### 定义模块来控制作用域与私有性
我们定义一个模块，是以 mod 关键字为起始，然后指定模块的名字，并且用大括号包围模块的主体。在模块内，我们还可以定义其他的模块。模块还可以保存一些定义的其他项，比如结构体、 枚举、常量、特性、或者函数。crate也是一个模块，一般是位于模块树最底端的隐式模块。

在 Rust 中，默认**所有项**（函数、方法、结构体、枚举、模块和常量）对父模块都是私有的。父模块中的项不能使用子模块中的私有项，但是子模块中的项可以使用它们父模块中的项。这是因为子模块封装并隐藏了它们的实现详情，但是子模块可以看到它们定义的上下文。

使模块公有并不使其内容也是公有的。模块上的 `pub` 关键字只允许其父模块引用它，而不允许访问内部代码。因为模块是一个容器，只是将模块变为公有能做的其实并不太多；同时需要更深入地选择将一个或多个项变为公有。

简单来说：子->父OK，父->子需要子为公开。将module设为pub只能保证父module能访问，但是子module的所有字段、函数等对于父module依旧是私有的。结构体同理。如果一个结构体具有私有字段，结构体需要提供一个公共的关联函数（与结构体同级可以访问私有字段）来构造该结构体的实例。

#### 引用模块项目的路径
路径有两种形式:

• 绝对路径(absolute path)是以 crate 根(root)开头的全路径;对于外部 crate 的代码， 是以 crate 名开头的绝对路径，对于当前 crate 的代码，则以字面值 crate 开头。

• 相对路径(relative path)从当前模块开始，以 self 、super 或定义在当前模块中的标识符开头。我们可以通过在路径的开头使用 `super` ，从父模块开始构建相对路径，而不是从当前模块或者 crate 根开始。这类似以 `..` 语法开始一个文件系统路径。

在作用域中增加 `use` 和路径类似于在文件系统中创建软连接（符号连接，symbolic link）。注意 `use` 只能创建 `use` 所在的特定作用域内的短路径。要想使用 `use` 将函数的父模块引入作用域，我们必须在调用函数时指定父模块，这样可以清晰地表明函数不是在本地定义的，同时使完整路径的重复度最小化。

使用 `use` 将两个同名类型引入同一作用域这个问题有另一个解决办法：在这个类型的路径后面，我们使用 `as` 指定一个新的本地名称或者别名。范例如下:
```rust
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result {
    // --snip--
}

fn function2() -> IoResult<()> {
    // --snip--
}
```

使用 `use` 关键字，将某个名称导入当前作用域后，这个名称在此作用域中就可以使用了，但它对此作用域之外还是私有的。如果想让其他人调用此代码时，也能够正常使用这个名称，就好像它本来就在当前作用域一样，那我们可以将 `pub` 和 `use` 合起来使用。这种技术被称为 “_重导出_（_re-exporting_）”：我们不仅将一个名称导入了当前作用域，还允许别人把它导入他们自己的作用域。例如：
```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

pub use crate::front_of_house::hosting;

pub fn eat_at_restaurant() {
    hosting::add_to_waitlist();
}
// 现在可以用路径`restaurant::hosting::add_to_waitlist()`调用函数add_to_waitlist。
```

- **使用外部包**
首先在*Cargo.toml*中添加依赖项，如`rand = "0.8.5"`。然后在项目包中使用`use`将要使用的module引入作用域。

- **嵌套路径来消除大量的use行**
```rust
use std::cmp::Ordering; 
use std::io;
// to be simplified
use std::{cmp::Ordering, io};
```

#### 将模块拆分为多个文件
将模块声明保留，其他对于模块的使用代码保留，将模块代码移动至与模块同名的文件中，编译器会自动加载。

## 集合
集合指向的数据是储存在堆上的。

- _vector_ 允许我们一个挨着一个地储存一系列数量可变的值

- **字符串**（_string_）是字符的集合。我们之前见过 `String` 类型，不过在本章我们将深入了解。

- **哈希 map**（_hash map_）允许我们将值与一个特定的键（key）相关联。这是一个叫做 _map_ 的更通用的数据结构的特定实现。

#### vector
[官方文档中vector部分](https://doc.rust-lang.org/std/vec/struct.Vec.html)

vector 只能储存相同类型的值，`Vec<T>` 是一个由标准库提供的类型，它可以存放任何类型。

vector可以使用索引或`get`方法。

- 使用索引访问vector时，使用 `&` 和 `[]` 会得到一个索引位置元素的引用:`let i : &i32 = &v[2];`。->当索引超出vector边界，程序Panic。

- 当使用索引作为参数调用 `get` 方法时，会得到一个可以用于 `match` 的 `Option<&T>`:`let i : Option<&i32> = v.get(2);`。->当索引超出vector边界，`get`方法返回`None` 值。

注意引用的规则（不可同时存在可变引用与不可变引用、注意引用的作用域..）->一旦对vector中的某一项进行了引用，vector将不能再被编辑（包括插入和删除）
可以使用枚举包装不同数据类型的成员，然后创造枚举类型的vector来储存不同数据类型的数据，例如：
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
vector离开作用域时失效。

#### String
[官方文档中的String部分](https://doc.rust-lang.org/std/string/struct.String.html)

很多 `Vec` 可用的操作在 `String` 中同样可用，事实上 `String` 被实现为一个带有一些额外保证、限制和功能的字节 vector 的封装。

`&String` 可以被 **强转**（_coerced_）成 `&str`。

Rust 的字符串不支持索引，取而代之可以使用 `[]`和一个 range 来创建**含特定字节数**的字符串 slice。也就是说，slice包含的字符串部分和组成单个字母所需的字节数有关，例如：
```rust
let hello = "Здравствуйте";
let s = &hello[0..4];
// 这里，`s` 会是一个 `&str`，它包含字符串的头四个字节。因为这些字母都是两个字节长的，所以这意味着 `s` 将会是 “Зд”。
```

遍历字符串需要表明遍历的是字符还是字节，两者区别如下：
```rust
// in chars
for c in "Зд".chars() {
    println!("{c}");
}
// print out:
// З
// д

// in bytes
for b in "Зд".bytes() {
    println!("{b}");
}
// print out:
// 208
// 151
// 208
// 180
```

#### HashMap
[官方文档中哈希表部分](https://doc.rust-lang.org/std/collections/struct.HashMap.html)

`HashMap<K, V>` 类型储存了一个键类型 `K` 对应一个值类型 `V` 的映射。它通过一个 **哈希函数**（_hashing function_）来实现映射，决定如何将键和值放入内存中。
使用哈希表时，必须首先 `use` 标准库中集合部分的 `HashMap`。哈希 map 将数据储存在堆上，所有的键必须是同类型，值也必须是同类型。

通过键值访问值时，`get` 方法返回 `Option<&V>`，如果某个键在哈希 map 中没有对应的值，`get` 会返回 `None`。对于实现了Copy trait的类型，一般使用`copied` 方法来获取一个 `Option<V>` 而不是 `Option<&V>`。还可以使用`unwrap_or()` 在哈希表中没有该键所对应的项时将其设置为零。

对于像 `i32` 这样的实现了 `Copy` trait 的类型，其值可以拷贝进哈希 map。对于像 `String` 这样拥有所有权的值，其值将被移动而哈希 map 会成为这些值的所有者。如果将值的引用插入哈希 map，这些值本身将不会被移动进哈希 map。但是这些引用指向的值必须至少在哈希 map 有效时也是有效的。

通过`Entry` 的 `or_insert` 方法可以实现只在键没有对应值时插入键值对，它在键对应的值存在时就**返回这个值的可变引用**，如果不存在则将参数作为新值插入并**返回新值的可变引用**。例如可以通过如下的代码统计单词出现次数：
```rust
let mut map = HashMap::new(); 
for word in text.split_whitespace() { 
	let count = map.entry(word).or_insert(0);// 若该单词没有出现过，返回一个键为该单词、值为0的键值对的可变引用
	*count += 1; // 对引用进行计算前需要解引用
}
```

## 错误处理
#### 不可恢复的错误-使用panic!处理
代码发生panic有两种可能：

1. 执行会造成代码 panic 的操作（比如访问超过数组结尾的内容）

2. 显式调用 `panic!` 宏

> 当出现 panic 时，程序默认会开始 **展开**（_unwinding_），这意味着 Rust 会回溯栈并清理它遇到的每一个函数的数据，不过这个回溯并清理的过程有很多工作。另一种选择是直接 **终止**（_abort_），这会不清理数据就退出程序。
> 那么程序所使用的内存需要由操作系统来清理。如果你需要项目的最终二进制文件越小越好，panic 时通过在 _Cargo.toml_ 的 `[profile]` 部分增加 `panic = 'abort'`，可以由展开切换为终止。

#### 可恢复的错误-使用result处理
```rust
enum Result<T, E> {
    Ok(T), // `T` 代表成功时返回的 `Ok` 成员中的数据的类型
    Err(E), // `E` 代表失败时返回的 `Err` 成员中的错误的类型
}
// `Result` 枚举和其成员被导入到了 prelude 中，不需要指定`Result::`
```
使用match匹配result类型：
```rust
use std::fs::File;
use std::io::ErrorKind;

fn main() {
    let greeting_file_result = File::open("hello.txt");

    let greeting_file = match greeting_file_result {
        Ok(file) => file,
        Err(error) => match error.kind() {
            ErrorKind::NotFound => match File::create("hello.txt") { // 一旦错误类型为找不到文件，尝试创造文件
                Ok(fc) => fc,
                Err(e) => panic!("Problem creating the file: {:?}", e),
            },
            other_error => {
                panic!("Problem opening the file: {:?}", other_error);
            }
        },
    };
}
```

`unwrap` 方法：如果 `Result` 值是成员 `Ok`，`unwrap` 会返回 `Ok` 中的值。如果 `Result` 是成员 `Err`，`unwrap` 会为我们调用 `panic!`。 `expect` 方法同理，但可以向 `expect` 传入参数作为调用 `panic!`时使用的错误信息。一个例子：
```rust
use std::fs::File;

fn main() {
    let greeting_file = File::open("hello.txt")
        .expect("hello.txt should be included in this project");
}
```

传播错误：在error发生时将一个 `Err(e)` 错误返回给调用它的代码，也就是说调用它的函数返回类型为 `Result<T, E>` ->调用这个函数的代码最终会得到一个包含对应信息的 `Ok` 值，或者一个包含 `Error` 的 `Err` 值。
传播错误可以使用简写 `?` 运算符：
```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut username_file = File::open("hello.txt")?;// 问号省略的部分即为`return Err(e)`
    let mut username = String::new();
    username_file.read_to_string(&mut username)?;
    Ok(username)
}
```

`match` 表达式与 `?` 运算符所做的有一点不同：`?` 运算符所使用的错误值被传递给了 `from` 函数，它定义于标准库的 `From` trait 中，其用来将错误从一种类型转换为另一种类型。当 `?`运算符调用 `from` 函数时，收到的错误类型被转换为由当前函数返回类型所指定的错误类型。`?` 运算符只能被用于返回值（返回 `Result<T, E>` ）与 `?` 作用的值相兼容的函数。

`?` 也可用于 `Option<T>` 值。如同对 `Result` 使用 `?` 一样，只能在返回 `Option`的函数中对 `Option` 使用 `?`。在 `Option<T>` 上调用 `?` 运算符的行为与 `Result<T, E>` 类似：如果值是 `None`，此时 `None` 会从函数中提前返回。如果值是 `Some`，`Some` 中的值作为表达式的返回值同时函数继续。

> 函数通常都遵循 **契约**（_contracts_）：它们的行为只有在输入满足特定条件时才能得到保证。如果函数有一个特定类型的参数，可以在知晓编译器已经确保其拥有一个有效值的前提下进行你的代码逻辑。例如，如果你使用了一个并不是 `Option` 的类型，则程序期望它是 **有值** 的并且不是 **空值**。你的代码无需处理 `Some` 和 `None` 这两种情况，它只会有一种情况就是绝对会有一个值。尝试向函数传递空值的代码甚至根本不能编译，所以你的函数在运行时没有必要判空。

## 泛型、Trait 和生命周期
#### 泛型
泛型是具体类型或其他属性的抽象替代，允许我们使用一个可以代表多种类型的占位符来替换特定类型。泛型的用法： `fn largest<T>(list: &[T]) -> &T {` 。可以这样理解这个定义：函数 `largest` 有泛型类型 `T`。它有个参数 `list`，其类型是元素为 `T` 的 slice。`largest` 函数会返回一个与 `T` 相同类型的引用。

可以使用 `<T: std::cmp::PartialOrd>` 来确保泛型能比较，同理可以确保泛型时实现了某种trait的类型。

同样也可以用 `<>` 语法来定义结构体，它包含一个或多个泛型参数类型字段：
```rust
struct Point<T, U> { 
	x: T, 
	y: U, 
}
```
定义结构体 `Point<T>`，在其上实现名为 `x` 的方法：
```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```
定义方法时也可以为泛型指定限制（constraint）。例如，可以选择为 `Point<f32>` 实例实现方法，而不是为泛型 `Point` 实例:
```rust
impl Point<f32> {
    fn distance_from_origin(&self) -> f32 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}
// 这段代码意味着 `Point<f32>` 类型会有一个方法 `distance_from_origin`，而其他 `T` 不是 `f32` 类型的 `Point<T>` 实例则没有定义此方法。
```

#### trait：定义共同行为
[trait的文档部分](https://doc.rust-lang.org/reference/items/traits.html)

> 本质为定义了一个特性，不同的数据类型可以为实现这个特性提供方法，来达到让不同的数据类型拥有一个特性

_trait_ 定义了某个特定类型拥有可能与其他类型共享的功能。可以通过 trait 以一种抽象的方式定义共同行为。可以使用 _trait bounds_ 指定泛型是任何拥有特定行为的类型。
_trait_ 定义是一种将方法签名组合起来的方法，目的是定义一个实现某些目的所必需的行为的集合。例如：
```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```
这里使用 `trait` 关键字来声明一个 trait，后面是 trait 的名字，在这个例子中是 `Summary`。我们也声明 `trait` 为 `pub` 以便依赖这个 crate 的 crate 也可以使用这个 trait。在大括号中声明描述实现这个 trait 的类型所需要的行为的方法签名，在这个例子中是 `fn summarize(&self) -> String`。

可以在定义方法签名后，在trait的声明处提供一个默认的实现方法，这样当为某个特定类型实现 trait 时，可以选择保留或重载每个方法的默认行为。例如：
```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
```
同时，默认实现允许调用相同 trait 中的其他方法，哪怕这些方法没有默认实现。注意无法从相同方法的重载实现中调用默认方法。

当声明了trait之后，需要在类型上实现trait：
```rust
pub struct NewsArticle {
    pub headline: String,
    pub location: String,
    pub author: String,
    pub content: String,
}

impl Summary for NewsArticle {
    fn summarize(&self) -> String {
        format!("{}, by {} ({})", self.headline, self.author, self.location)
    }
}
```
在使用时，需要将trait引入作用域。只有在 trait 或类型至少有一个属于当前 crate 时，我们才能对类型实现该 trait。例如，可以为 `aggregator` crate 的自定义类型 `Tweet` 实现如标准库中的 `Display` trait，这是因为 `Tweet` 类型位于 `aggregator` crate 本地的作用域中。类似地，也可以在 `aggregator` crate 中为 `Vec<T>` 实现 `Summary`，这是因为 `Summary` trait 位于 `aggregator` crate 本地作用域中。

不能为外部类型实现外部 trait。例如，不能在 `aggregator` crate 中为 `Vec<T>` 实现 `Display`trait。这是因为 `Display` 和 `Vec<T>` 都定义于标准库中，它们并不位于 `aggregator` crate 本地作用域中。

如果将实现trait的某种类型作为参数传入函数，这个函数就可以调用trait内定义的一些方法，例如：
```rust
pub fn notify(item: &impl Summary) { // 参数可以为任何实现了Summary trait的类型
    println!("Breaking news! {}", item.summarize());
}
```
如果需要参数同时满足实现两个不同的trait，有如下两种写法：
```rust
pub fn notify(item: &(impl Summary + Display)) {
pub fn notify<T: Summary + Display>(item: &T) { // 使用trait bound
```

返回实现trait的类型可以如此书写函数签名：
```rust
fn returns_summarizable() -> impl Summary {
// 其中Summary为一个trait类型
```

通过使用 `impl Summary` 作为返回值类型，我们指定了 `returns_summarizable` 函数返回某个实现了 `Summary` trait 的类型，但是不确定其具体的类型。并且，函数体中可能返回的类型必须是一致的，假如返回两个不同但是同样实现了目标trait的类型也不能通过编译。

可以对任何实现了特定 trait 的类型有条件地实现 trait。对任何满足特定 trait bound 的类型实现 trait 被称为 _blanket implementations_，它们被广泛的用于 Rust 标准库中。例如，标准库为任何实现了 `Display` trait 的类型实现了 `ToString` trait。这个 `impl` 块看起来像这样：
```rust
impl<T: Display> ToString for T {
    // --snip--
}
```
因为标准库有了这些 blanket implementation，我们可以对任何实现了 `Display` trait 的类型调用由 `ToString` 定义的 `to_string` 方法。同理，我们可以对任何实现了traitA的类型实现traitB来补充方法。

#### 生命周期
生命周期是针对**引用**的，为了保证实际使用时的引用是有效的，需要我们给出泛型生命周期的参数。一个关于生命周期的示例：
```rust
fn main() {
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
// 这里将 `r` 的生命周期标记为 `'a` 并将 `x` 的生命周期标记为 `'b`。
```
在编译时，Rust 比较这两个生命周期的大小，并发现 `r` 拥有生命周期 `'a`，不过它引用了一个拥有生命周期 `'b` 的对象。程序被拒绝编译，因为生命周期 `'b` 比生命周期 `'a` 要小：被引用的对象比它的引用者存在的时间更短。

生命周期注解**并不改变任何引用的生命周期的长短**。相反它们描述了**多个引用生命周期相互的关系**（也就是两个引用是否保持同步的生命周期等），而不影响其生命周期。与当函数签名中指定了泛型类型参数后就可以接受任何类型一样，当指定了泛型生命周期后函数也能接受任何生命周期的引用。生命周期参数注解位于引用的 `&` 之后，并有一个空格来将引用类型与生命周期注解分隔开。
```rust
&i32        // 引用
&'a i32     // 带有显式生命周期的引用
&'a mut i32 // 带有显式生命周期的可变引用
```
例如如果函数有一个生命周期 `'a` 的 `i32` 的引用的参数 `x`。还有另一个同样是生命周期 `'a` 的 `i32` 的引用的参数 `y`。这两个生命周期注解意味着引用 `x` 和 `y` 必须与这泛型生命周期存在得一样久。例如：
```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
```
这个函数签名说明**对于某些生命周期 `'a`**，函数会获取两个参数，它们都是与生命周期 `'a` 存在得一样久的字符串 slice。函数会返回一个同样也与生命周期 `'a` 存在得一样久的字符串 slice。它的实际含义是 `longest` 函数返回的引用的生命周期与函数参数所引用的值的生命周期的**较小者**一致。当具体的引用被传递给 `longest` 时，被 `'a` 所替代的具体生命周期是 `x` 的作用域与 `y` 的作用域**相重叠的那一部分**。换一种说法就是泛型生命周期 `'a` 的具体生命周期等同于 `x` 和 `y` 的生命周期中较小的那一个。因为我们用相同的生命周期参数 `'a` 标注了返回的引用值，所以返回的引用值就能保证在 `x` 和 `y` 中较短的那个生命周期结束之前保持有效。

`longest` 函数并不需要知道 `x` 和 `y`具体会存在多久，而只需要知道有**某个可以被 `'a` 替代的作用域**将会满足这个签名。当在函数中使用生命周期注解时，这些注解出现在函数签名中，而不存在于函数体中的任何代码中。生命周期注解成为了函数约定的一部分，非常像签名中的类型。

> 总结：函数签名注明了一个**泛型**生命周期，在实际传入参数后会根据传入参数情况实例化成一个具体的生命周期，根据不同变量的生命周期注解不同，被具体化的生命周期也随之变化

存放引用的结构体
```rust
struct ImportantExcerpt<'a> { 
	part: &'a str, 
}
// `ImportantExcerpt` 的实例不能比其 `part` 字段中的引用存在的更久。
```

静态生命周期
这里有一种特殊的生命周期值得讨论：`'static`，其生命周期**能够**存活于整个程序期间。所有的字符串字面值都拥有 `'static` 生命周期，我们可以选择像下面这样标注出来：`let s: &'static str = "I have a static lifetime.";`

### After Chapter11
TODO
感觉上面的暂时够用了，剩下的等我看到不懂的代码再补

#### Arc - atomic reference counted
`Arc` 是一种允许多个所有者共享数据并在不同线程之间安全地访问它所指向的数据的智能指针。

`Arc` 和多个不可变引用之间有几个重要的区别：

1. **线程安全性**：
    
	- `Arc` 是原子引用计数类型，可以安全地在**多个线程**之间共享。它通过使用原子操作来确保引用计数的线程安全性，因此可以安全地在并发环境中使用。
    
	- 多个不可变引用只能在单个线程中使用，因为它们没有内置的同步机制来处理多线程并发访问的问题。如果尝试在多个线程中共享不可变引用，就可能会导致数据竞争和不安全的行为。

2. **内存管理**：
    
	- `Arc` 允许多个所有者共享数据，并在所有所有者都释放它时销毁数据。这意味着即使在程序的不同部分有多个引用，数据也只会在最后一个引用被销毁时释放。
    
	- 多个不可变引用是针对单个所有者的。如果希望在不同部分使用相同的数据，必须通过复制或移动数据来传递所有权，这可能会导致内存拷贝和额外的内存开销。

3. **可变性**：
    
	- `Arc` 只允许不可变引用。因为它的目标是允许多个所有者共享数据，并在多个所有者之间进行安全的并发访问，所以不支持可变引用。如果需要在多个线程之间共享可变数据，可以考虑使用 `Mutex`、`RwLock` 等类型。
    
	- 多个不可变引用也只能提供不可变性，无法直接支持可变引用。如果希望在单个线程中进行可变访问，则需要使用可变引用（`&mut`）。