# Ghost Cell与StaticRC的试用体验
写于2021年9月3日。最早发表于[知乎](https://www.zhihu.com/question/482473609/answer/2101373347)。

---
试用了下GhostCell[^1][^2] + StaticRc[^3]，用来写链表。总体感觉非常不错，很创新的一个概念，但如果能内置到编译器层面，可能使用上会更加方便。

## `GhostCell` ——安全、零开销的内部可变性机制
`GhostCell`的思想大致是：通过一个`GhostToken`来控制一系列与之生命周期相等的`GhostCell`数据的读写行为，从而实现安全、零开销的内部可变性。

`GhostCell`文档首页给出的一个样例可以说明其使用方法。 
```rust,ignore
use ghost_cell::{GhostToken, GhostCell};

let n = 42;

let value = GhostToken::new(|mut token| {
    let cell = GhostCell::new(42);

    let vec: Vec<_> = (0..n).map(|_| &cell).collect();

    *vec[n / 2].borrow_mut(&mut token) = 33;

    *cell.borrow(&token)
});

assert_eq!(33, value);
```
首先通过`GhostToken::new`方法创建一个`token`，但这个`token`并非直接创建出来，而是绑定在一个局部作用域（即传入的闭包）内。

所有装入`GhostCell`内的值，均可通过`GhostCell::borrow`与`GhostCell::borrow_mut`两个方法来读写。这两个方法的签名分别是：
```rust
pub fn borrow<'a>(&'a self, _: &'a GhostToken<'brand>) -> &'a T;
pub fn borrow_mut<'a>(&'a self, _: &'a mut GhostToken<'brand>) -> &'a mut T;
```
 可见，不论读写，都只需要`GhostCell`的 **不可变借用** ，不同的是数据读取需要`&token`，而数据写入需要`&mut token`。 两种借用方式借得的`&T`或`&mut T`生命周期与`cell`和`token`同时绑定（即函数签名中的`'a`）。 

由于`token`的生命周期与作用域内创建的所有`GhostCell`的生命周期都与`token`相等（`GhostCell<'id, T>`与`GhostToken<'id>`内的生命周期标识`'id`都是 **不变 (invariant) ）** 的，因此可以确保在`token`被借用时，或者允许多个`cell`的读取（通过`&token`），或者允许单独某个`cell`写入（通过`&mut token`）。 

回到文档中的样例：
```rust,ignore
    let cell = GhostCell::new(42);
    let vec: Vec<_> = (0..n).map(|_| &cell).collect();
```
这里将同一个`cell`的只读引用复制了`n`遍放到一个数组中。如果数组未放到`GhostCell`内，就不可能在Safe Rust范围内改写数组内的元素。原本修改`cell`内的值需要可变借用`&mut cell`，而引入`GhostCell`后，即使只能拿到`&cell`，只需独占借用`&mut token`，即可实现安全的内部可变性，类似一种「可变借用转移」的效果。

不同于`RefCell`的是，`GhostCell`是零开销的。`RefCell`虽然也可以通过`RefCell::borrow_mut(&cell)`来实现内部可变，但其内部会记录当前值的借用状态，并在运行时检查。`GhostCell`的检查是在编译期完成的，因为`token`的读写是互斥的，所以在当前`token`的生命周期内，所有的`GhostCell`的读写都是安全的，并且是`Send + Sync`的。 

[[^1]] 中给出了GhostCell安全性的证明。 

## `StaticRc`——零开销的共享所有权机制

`ghost_cell`库中提到了`static_rc` [^3] 库，实现了一种编译期检查的零开销共享所有权机制。其思想是：将`Box`的所有权分开，引入有理数（大于0，不大于1）范围意义的所有权。`StaticRc<T, const NUM: usize, const DEN: usize>`的所有权值为有理数`NUM/DEN`（`NUM <= DEN`）。当`NUM/DEN` = 1时，代表该指针拥有完全的所有权，即等价于`Box`；当`NUM/DEN` < 1时，代表与其他指针共享所有权；指同一对象的所有指针，其所有权值之和不大于`1`。

引入`split`与`join`两个操作。其中`split`操作将一个指针一分为二，分为两个所有权值更小的指针（二者所有权值之和等于分裂前的所有权值）；`join`操作将两个指针合二为一，合成一个新指针，其所有权值之和等于合成后指针的所有权值（不大于一）。

在`StaticRc`中，仅独占所有权的指针（即所有权值为1）具有完全操作所指对象的权限（读(`&ptr`)、写(`&mut ptr`)、消耗(`ptr`)），等同于`Box`；共享指针（即所有权值小于1）仅具有只读权限(`&ptr`)。

该库的安全性还有待证明。我不太熟悉生命周期证明的那套理论，仅从直觉上看没有什么大问题。不过看了一下该库的实现，共享所有权指针的`Drop`是静默的，仅独占所有权的指针会被释放，这可能很容易引起内存泄露，所以我认为将`StaticRc`设计成线性类型可能更好（即至少且至多被使用一次），对共享指针而言，必须将其合倂为独占指针再释放，否则报编译错误。 

## 安全、零开销的链表实现尝试
[[^1]] 中已经提出了一种基于arena的双向链表实现方式，[[^4]] 中还给出了一种基于`GhostCell` + `Arc`的链表实现。我认为前者更优，因为后者需要`Arc`额外开销，所以基本上不会考虑这种实现。而前者虽然无额外开销，但arena的实现可能在节点删除后不能及时的释放，所以我就尝试了如下的第三种方式。

考虑到（双向）链表中每个节点恰好由两个指针共享持有，因此想到可以用`StaticRc`来实现共享所有权，用`GhostCell`实现内部可变性，若`StaticRc`保持安全，便可实现安全、零开销的双向链表。

定义链表的数据结构如下：
```rust,ignore
pub struct List<'id, T> {
    head: Option<NodePtr<'id, T>>,
    tail: Option<NodePtr<'id, T>>,
    len: usize,
}

struct Node<'id, T> {
    next: Option<NodePtr<'id, T>>,
    prev: Option<NodePtr<'id, T>>,
    elem: T,
}

type NodePtr<'id, T> = Half<GhostCell<'id, Node<'id, T>>>;

type Half<T> = StaticRc<T, 1, 2>;
type Full<T> = StaticRc<T, 2, 2>;
```
其中，`head`,`tail`,`next`,`prev`都是共享所有权（1/2）的指针（`Half`），套上`GhostCell`。每个节点由一前一后两个指向其的指针共享所有权；链表首/尾节点则由`head`/`tail`指针共享另一半所有权。 `Half`指针只能拿到不可变引用，但可借助`GhostCell`实现内部可变性，看起来十分完美，于是我就写了一个简单的`push` / `pop`操作：
```rust,ignore
    pub fn push_back(&mut self, side: usize, elem: T, token: &mut GhostToken<'id>) {
        // 创建新节点后即一分为二。
        let (left, right) = Full::split(Full::new(GhostCell::new(Node::new(elem))));
        match self.tail.take() {
            Some(tail) => {
                // 链接其左半边指针。若当前链表为空，则链接`self.head`。
                tail.deref().borrow_mut(token).next = Some(left);
                // 将插入前的`self.tail`所有权转移至新插入节点。
                right.deref().borrow_mut(token).prev = Some(tail);
            }
            None => self.head = Some(left),
        }
        // 链接其右半边指针到新的`self.tail`。
        self.tail = Some(right);
    }
    pub fn pop_back(&mut self, side: usize, token: &mut GhostToken<'id>) -> Option<T> {
        // 从`self.tail`取其右半边指针（若为空则返回`None`）。
        let right = self.tail.take()?;
        let left = match right.deref().borrow_mut(token).prev.take() {
            Some(tail) => {
                // 若`self.tail`的前一节点不为空，则从其`next`中取出左半边指针，
                // 否则从`self.head`取出左半边指针。
                let left = tail.deref().borrow_mut(token).next.take().unwrap();
                // 重新链接`self.tail`
                self.tail = Some(tail);
                left
            }
            None => self.head.take().unwrap(),
        };
        // 将左右指针合二为一，并返回弹出的节点元素。弹出的元素在包含`Box`内，
        // 用完后会随`Box`一起回收。
        Some(Full::into_box(Full::join(left, right)).into_inner().elem)
    }
```
至此都一切顺利，直到我想实现一个可变的迭代器。一开始按照标准的迭代器写法：
```rust,ignore
pub struct IterMut<'id, 'iter, T> {
    head: Option<&'iter NodePtr<'id, T>>,
    tail: Option<&'iter NodePtr<'id, T>>,
    len: usize,
    token: &'iter mut GhostToken<'id>,
}

impl<'id, 'iter, T> Iterator for IterMut<'id, 'iter, T> {
    type Item = &'iter mut T;

    fn next(&mut self) -> Option<Self::Item> {
        let current = self.head?;
        self.head = current.deref().borrow(self.token).next;
        self.len -= 1;
        // 编译错误：token先被不可变借用，后被可变借用，且借用周期都等于`'iter`。
        Some(&current.deref().borrow_mut(self.token).elem)
    }
}
```
结果未能通过借用检查规则。原因很简单，遍历时获取每个节点的`next`指针需要不可变借用`token`，而返回节点中元素的可变引用，则需要可变借用`token`，且借出的`token`周期均为`'iter`（即整个迭代器的生命周期），自然不可能通过借用检查。

后来又另辟溪径，参考[[^4]]中的一个迭代器实现，他的可变迭代并不像一般的设计模式那样返回一个迭代器结构，而是直接采用类似`Iterator::for_each`的方式，在闭包内完成可变借用，从而把可变借出的生命周期缩短。

于是我也实现了一个`for_each_mut`的接口：
```rust,ignore
    pub fn for_each_mut(&self, token: &mut GhostToken<'id>, mut f: impl FnMut(&mut T)) {
        let mut current = self.head.as_ref();
        while let Some(node) = current {
            let node = node.deref().borrow_mut(token);
            f(&mut node.elem);
            current = node.deref().next.as_ref();
        }
    }
```
不幸地是，虽然看起来这个函数中只有一次`borrow_mut`，且只在循环体内，应该会在单次循环后归还。但由于每次获得`next`指针时依赖于上一次的借用，该借用的生命周期硬生生地被从循环体内拉长到了循环体外。 因此出现了如下的编译错误
```
error[E0499]: cannot borrow `*token` as mutable more than once at a time
   --> src/experiments.rs:182:48
    |
182 |             let node = node.deref().borrow_mut(token);
    |                                                ^^^^^ `*token` was mutably borrowed here in the previous iteration of the loop
```
基于arena或`Arc`的实现则不存在这一问题。基于`arena`的实现中，所有的`next`或`prev`都是引用，且有单独的生命周期，因此在迭代时不存在需要借用才能拿到`next`的情况； 基于`Arc`的实现中，共享指针`Arc`可以在运行时复制一份指针，因此也不需要通过借用来获取`next`指针。而`StaticRc`并不支持在运行时临时新增一份引用，只能通过`split`的方式将其原有的半份所有权分裂开来。这种方式不仅涉及`next`字段的修改（需要用`Option::take`将整个`Half`指针按值的方式取出来，才能调用`split`），且在所有权一分为二后，又存在合井困难的问题（因为`next`指针的值仍然借用于分裂后的指针，会阻止分裂指针合井）。

结论：该方案（`StaticRc` + `GhostCell`）在描述定义链表，或操作链表中的单个节点时虽能胜任安全零开销的实现。但对链表作遍历或其他更复杂的操作时，需要将链表的结构可变性与数据可变性作进一步分离，才能实现全功能的安全零开销链表。

---
总之，我比较看好`GhostCell`的思想，将大有可能改进下一个Rust Edition中的借用检查的体验（就像2015的Lexical Lifetime到Non-Lexical Lifetime的改进一样）， 进一步平衡安全、性能、易用的三角。

`GhostCell`的不足之处：写起来比较繁琐，会需要多写不少的`GhostToken::new`、`GhostCell::new`、`borrow(token)`、`borrow_mut(token)`之类的代码。如果能集成到编译器内，通过更简洁的语法书写，使用体验会好很多。

拭目以待。

---
### 参考资料

[^1]: <http://plv.mpi-sws.org/rustbelt/ghostcell/paper.pdf>

[^2]: <https://docs.rs/ghost-cell/0.2.2/ghost_cell/>

[^3]: <https://docs.rs/static-rc/0.4.1/static_rc/>

[^4]: <https://gitlab.mpi-sws.org/FP/ghostcell/-/blob/master/ghostcell/examples/dlist_arc.rs>