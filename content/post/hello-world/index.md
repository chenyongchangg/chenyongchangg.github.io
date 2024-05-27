---
title: Rust Closure
description: Rust Closure
date: 2024-05-23 00:00:00+0000
categories:
    - rust
tags:
    - closure
---
Rust闭包理解
前言#
这篇文章的目的是让读者最快最直观的了解什么是闭包，Rust中的三种闭包之间有什么区别。为了达到这个目的——即降低复杂性，本篇文章的用词可能不够严谨，见谅。

看本篇文章之前，请确保你对Rust的值、借用、生命周期、trait、泛型等概念有了充分的理解，推荐这篇文章：Rust生命周期的理解。

最近在编码中用到了闭包，其实闭包在其它语言中没什么好讲的，本来就是为了便于开发而存在的一个特性，但Rust里因为有所有权和借用限制，所以其中的闭包很特别，很值得研究。

什么是闭包#
简单来说，闭包是一个可以执行的东西，这个东西可以捕获外部资源，就像这样：

function counter() {
let i = 0; // 外部资源
return function() {  // 闭包体，这里是可执行的匿名函数
return i++; // 捕获外部变量i
}
}
各种语言都有其自己的闭包实现，JS中可以用函数、lambda实现闭包，Java中可以用匿名内部类、lambda表达式实现闭包.....而且对于被捕获的外部资源的处理方式，各种语言也不尽相同。

关于闭包实现细节的讨论，可以看一下这一篇文章：Lambda与final变量，其讨论了Groovy中的闭包以及用Java来模拟这种闭包的方式。

在其它语言中实现闭包该考虑什么#
下面是一段伪代码，你可以将它考虑成JS，它很自然的创建创建了一个闭包并返回给外界，你清楚的知道，你在外界每调用一次闭包，i都会自增并返回。

function counter() {
let i = 0;
return function() {
return i++;
}
}

let c = counter();
assert(c(), 0);
assert(c(), 1);
assert(c(), 2);
但是i存在哪里了？当counter返回时，该函数的栈已经被销毁，i这个本地变量也不复存在，换句话说，这个i一定被某种机制保留了下来，在Groovy和Java中，编译器可能会让一个内部类来持有闭包访问的全部外部资源，并在每次创建闭包时创建该类的一个实例。换句话说，它们转而将i分配到了堆上，而非栈上，此时，函数返回，栈被清空，但i还在堆中。

上面这一段，无论你用GC语言还是需要手动管理内存的语言来理解都行，但是千万不要用Rust理解，因为Rust变量无论在堆还是栈中，超出作用域都会被销毁。

再考虑一个问题：

function foo() {
let i = 0;

    call_some_async(() => {
        // do something...
    });

    // do something...
}
在这段代码中，foo中进行了一个异步调用，即call_some_async，call_some_async中的闭包可以访问i，并对其作出修改，但foo中下面的代码也可以对其进行修改，闭包操作的i是foo中的一个副本？还是同一份数据？

上面的两个问题，没有特定的答案，每一种语言都会选择自己的实现方式，这其中有来自很多方面的权衡。你可能学过很多语言，你也在很多语言中用过闭包，但你可能从来都没考虑过这些问题，因为大部分语言设计的初衷就是将这些问题屏蔽，让它们尽量少的扰乱开发人员，而借用和所有权机制让Rust必须把这些问题摆到明面上，开发人员必须清楚的知道你在写什么，或者，至少你通过瞎蒙让编译器明白了你在写什么。总之，在Rust中，程序有责任确定闭包如何访问外部资源。

Rust中的闭包#
语法基础#
Rust中的闭包语法类似lambda，但Rust书中并没有将其称为lambda，下面是Rust的一个闭包：

|x| x
被两条竖线包裹的是闭包的参数，闭包的最后一行是其返回值，当闭包只有一行时，可以省略花括号。所以，上面的闭包的完整格式是：

|x| {
x
}
使用闭包#
看一下下面这个使用闭包的代码，变量sum是一个闭包，由于没有类型信息，我们必须将类型信息补全，它接收两个i8类型变量，将它们相加后返回。

fn main() {
let sum = |x: i8, y: i8| -> i8 {
x + y
};
println!("{}", sum(2, 3));
}
这是很简单的一个闭包，它甚至没有捕获外部变量，所以上面我们见到的一系列头疼的问题，我们都没有考虑。

那要是捕获外部变量呢？#
默认借用#
假设，我们这个闭包现在返回字符串，它要捕获外部变量prefix，作为前缀添加到结果之前：

fn main() {
let prefix = String::from("x + y is");

    let sum = |x: i8, y: i8| -> String {
        format!("{} => {}", prefix, x + y)
    };

    println!("{}", sum(2, 3));
    println!("{}", prefix); // 尝试再次访问prefix
}
我们知道，Rust中的变量是有所有权的，所以按理来说，prefix的所有权被转移到了闭包中，稍后，我们不能再访问prefix，这段代码应该不能编译，但实际上它却成功编译了，并打印了：

x + y is => 5
x + y is
所以足以见得，默认情况下，闭包只会借用它所访问的外部变量。所以，根据Rust的借用规则，闭包sum的生命周期不可能超过其借用的值的生命周期。

注意：默认情况下这种说法并不准确，事实上，Rust有三种闭包类型，目前展示的只是其中一种以不可变借用来借用外部变量的闭包。不过你可以暂时这样理解。

我想改变外部变量#
如果你想改变外部变量呢？像这样？

fn main() {
let mut prefix = String::from("x + y is"); // 注意，这里改成了mut

    let sum = |x: i8, y: i8| -> String {
        prefix.push_str("123");
        format!("{} => {}", prefix, x + y)
    };

    println!("{}", sum(2, 3));
}
很遗憾，即使prefix是mut，这段代码也无法通过编译。我先说怎么改吧，你只需要把sum改成mut的就行，就像这样：

fn main() {
let mut prefix = String::from("x + y is");

    let mut sum = |x: i8, y: i8| -> String {
        prefix.push_str("123");
        format!("{} => {}", prefix, x + y)
    };

    println!("{}", sum(2, 3));
}
看起来好像，让人迷迷糊糊的，为什么要修改sum的类型呢？

闭包的状态#
闭包是有状态的。怎么说？

考虑一下，你大可以将sum传递给别的函数，或者是某种库函数，对于它们来说，sum是一个黑盒子，它们只知道给sum传入两个i8，然后它就会返回一个字符串。它们不知道sum中持有了main函数中的本地变量prefix，而闭包所持有的所有外部变量，就是它的状态。

举一个更生动的例子，就是之前我们的counter的例子：

fn main() {
let mut i = -1i32;

    let mut counter = || -> i32 {
        i += 1;
        i
    };

    assert_eq!(0, counter());
    assert_eq!(1, counter());
    assert_eq!(2, counter());
}
counter每次会将i自增1，以实现计数效果，而这个i，就是这个闭包的状态。

闭包被声明为mut，就代表它是可变状态的闭包，对于外部值，它会使用&mut借用来捕获
如果闭包没有被声明为mut，就代表它是不可变状态的闭包，对于外部值，它会使用&借用来捕获
我想拥有外部的变量#
考虑下面的代码：

fn main() {
let mut string = String::from("123");

    let mut task = || {
        string.push_str("4");
        string.push_str("5");
        string.push_str("6");
    };
    thread::spawn(task);

    println!("{}", string);
}
thread::spawn创建了一个新线程，同时传递了一个闭包，这个闭包将作为任务在新线程中执行。闭包task中改变了string，同时，main中后面的代码也使用了string变量。

这看似已经违反了Rust的所有权规则，因为task在新线程中执行，Rust编译器并不能在像单线程环境中那样分析借用的生命周期，因为task的执行时间并不确定。很有可能，task闭包在可变借用string的同时，main还在读取string，这是Rust不允许出现的。

如果你把thread:spawn(task);换成对该闭包的普通调用task();，代码就能通过编译，因为Rust编译器可以分析出可变借用的生命周期在main后面再次使用前结束了。

为了避免这种情况出现，在创建线程时，你传递的闭包不能借用任何外部值，如果你非要使用，那就将所有权转移到闭包中。

所以，上面的代码即使去掉最后的println!语句也无法通过编译，唯一的办法就是，使用一种一旦捕获了外部状态，就将该状态转移到闭包中的闭包。你可以使用move关键字做到这一点：

fn main() {
let mut string = String::from("123");

    let mut task = move || { // 注意这个move关键字，它表示将所有捕获的外部状态的所有权移动到闭包中
        string.push_str("4");
        string.push_str("5");
        string.push_str("6");
    };
    thread::spawn(task);
}
暂时的总结#
我们目前见到了三种闭包，它们对捕获状态的处理方式不同：

不可变状态的闭包：以不可变借用方式借用捕获到的状态，所以，闭包的生命周期不能大于所捕获变量的生命周期，闭包中捕获的状态无法修改
可变状态的闭包：以可变借用方式借用捕获到的状态，所以，闭包的生命周期不能大于所捕获变量的生命周期，闭包中可以修改捕获到的状态（但要满足Rust的借用规则）
获取所有权的闭包：对于所有需要捕获的状态，获取其所有权。闭包的生命周期不再受所捕获的变量限制，对其内部状态有完整的使用权限。
复杂一点，把闭包用在函数上#
激动人心的时候来了，我们如何将闭包应用在函数参数和返回值上呢？因为只有这样，我们才能将其传递出去。

闭包类型说明#
当你想将闭包放到方法参数或返回值上时，你面对的第一个问题就是，我该使用什么类型来说明它？

对于我们上面说到的三种闭包，Rust提供了三种Trait：

Fn：对应不可变状态闭包
FnMut：对应可变状态闭包
FnOnce：对应获取所有权的闭包，特别注意，由于捕获变量所有权转移到了闭包中，所以该闭包只能调用一次，之后其捕获状态就被销毁，这可能也是FnOnce得名的原因吧
你在传递闭包时，可以用这些trait来描述参数或返回值，你定义的闭包自动实现了这些trait。由于使用了trait，所以描述的时候需要用到泛型，有点恶心

Fn的使用示例#
fn foreach_indexed<F, E>(vector: Vec<E>, f: F)
where F: Fn(usize, &E) {
for i in 0..vector.len() {
f(i, vector.get(i).unwrap());
}
}
上面，我们定义了一个foreach_indexed函数，它接收一个vector和一个Fn（以不可变方式借用所捕获变量的闭包），对于vector中的第i个元素e，它将以i, &e两个参数调用闭包。

由于Fn只能不可变借用外部变量，所以，你无法在该方法中获取vector中的一个元素的所有权，你只能通过vector.get(i).unwrap()获取其不可变引用。

貌似是Kotlin中，原生就支持了foreach_indexed这个函数以在函数式遍历时顺便获取下标。这其实是个很有用的函数，但很多语言都不支持，而我们上面的函数，无法以vector.foreach_indexed的形式调用，只能以foreach_indexed(vector, ...)的形式调用，你可以理解为一个是对象式风格，一个是过程式风格，在使用函数式编程时，我们扩展的这个方法可能造成编码风格的不统一。我们可以使用自定义trait来对某些不属于自己的结构进行横向功能扩展：

trait ForEachIndexed {
type Item;
fn foreach_indexed<F>(&self, f: F) where F: Fn(usize, &Self::Item);
}

impl<E> ForEachIndexed for Vec<E> {
type Item = E;

    fn foreach_indexed<F>(&self, f: F) where F: Fn(usize, &Self::Item) {
        for i in 0..self.len() {
            f(i, self.get(i).unwrap());
        }
    }
}

fn main() {
vec![1, 2, 3, 4, 5].foreach_indexed(|i, e| println!("{}. {}", i, e));
}
注意：Rust规定，当trait A和要实现trait A的结构B都不在当前crate下时，你才不能impl A for B。所以，你只需要在你的crate中定义trait，其中包含你要扩展的功能，并在你自己的crate中impl这个trait，你就可以横向扩展其它人的结构。这和Kotlin中带接收者类型的方法所带来的横向扩展很像。

注意：虽然Rust没有直接提供foreach_indexed，但你可以使用enumerate()：

vec![1, 2, 3, 4, 5].iter()
.enumerate()
.for_each(|(i, e)| println!("{}. {}", i, e));
FnMut的使用示例#
emmm，我们来看下标准库的for_each：

fn main() {
let mut sum = 0;
vec![1, 2, 3].iter().for_each(|x| sum += x);
println!("{}", sum);
}
我们可以在for_each中使用本地变量进行累加，这要求闭包可以改变它所捕获的变量。FnMut和FnOnce都可以，但是FnOnce有一个致命的缺陷，就是它只能用一次，明显不满足我们这里的要求。

这里，我们仅仅将foreach_indexed中的泛型参数F从Fn类型改成了FnMut类型，遗憾的是，这个代码无法通过编译：

fn foreach_indexed<F, E>(vector: Vec<E>, f: F)
where F: FnMut(usize, &E) {
for i in 0..vector.len() {
f(i, vector.get(i).unwrap());
}
}
fn main() {
let mut sum = 0;
foreach_indexed(vec![1, 2, 3], |i, x| sum += x);
println!("{}", sum);
}
从下面的编译错误来看，我们尝试可变的借用f，但是f是一个不可变的变量：

img

闭包是有状态的，你若想闭包内的状态可变，闭包必须是可变的，回想本篇文章之前我们说的。这里相当于将创建出的闭包的所有权直接交给了foreach_indexed，那实际上我们可以直接在foreach_indexed中让它变成可变的，这样的话，也无需创建借用了：

fn foreach_indexed<F, E>(vector: Vec<E>, mut f: F) // f是可变的
where F: FnMut(usize, &E) {
for i in 0..vector.len() {
f(i, vector.get(i).unwrap()); // f是mut的，可以直接改变状态
}
}
fn main() {
let mut sum = 0;
foreach_indexed(vec![1, 2, 3], |i, x| sum += x);
println!("{}", sum);
}
作者想了好久，FnMut有什么应用场景呢？不用FnMut，for_each和map这种方法也照样能实现啊。后来我注意到标准库里的for_each和map都使用了FnMut，并且文档上给了一个在闭包中修改外部变量的例子。确实，想不出直接需要FnMut的场景，但是只要你的API不想限制用户在闭包中修改它本地的变量，你就得用FnMut。

FnOnce的示例#
这个我就不示例了，一般就用在多线程、异步任务里。

作者：Yudoge

出处：https://www.cnblogs.com/lilpig/p/17022845.html