# この文書について

この文書は僕がSwiftの[Enforce Exclusive Access to Memory](https://github.com/apple/swift-evolution/blob/master/proposals/0176-enforce-exclusive-access-to-memory.md)を読解する際に作った和訳です。

元にしたバージョンは [51d9e7991e42b94e49fce2327f63e12e0cc33042](https://github.com/apple/swift-evolution/blob/51d9e7991e42b94e49fce2327f63e12e0cc33042/proposals/0176-enforce-exclusive-access-to-memory.md) です。

```
Commits on May 12, 2017
 @rjmccall
Fix typo noted by T.J. Usiyan
rjmccall committed on GitHub 44 minutes ago 
51d9e79 
```

うまく訳せていない変な日本語になっている部分については助けてくれると嬉しいです。

ライセンスは[Swift Evolutionリポジトリ本体のApache License](https://github.com/apple/swift-evolution/blob/master/LICENSE.txt)を継承しています。

---

# Enforce Exclusive Access to Memory
# メモリへの排他的なアクセスの強制

* Proposal: [SE-0176](0176-enforce-exclusive-access-to-memory.md)
* Authors: [John McCall](https://github.com/rjmccall)
* Review Manager: [Ben Cohen](https://github.com/airspeedswift)
* Status: **Active review (May 2...May 8)**
* Previous Revision: [1](https://github.com/apple/swift-evolution/blob/7e6816c22a29b0ba9bdf63ff92b380f9e963860a/proposals/0176-enforce-exclusive-access-to-memory.md)
* Previous Discussion: [Email Thread](https://lists.swift.org/pipermail/swift-evolution/Week-of-Mon-20170501/036308.html)

## Introduction
## 導入

In Swift 3, it is possible to modify a variable while it's being used or
modified by another part of the program.  This can lead to unexpected and
confusing results.  It also forces a great deal of conservatism onto the
implementation of the compiler and the standard libraries, which must
generally ensure the basic soundness of the program (no crashes or
undefined behavior) even in unusual circumstances.

Swift3では、
変数がプログラムの他の箇所から使われているか変更されている最中に、
それを変更することが可能である。
これは予想外だったり混乱する結果をもたらしうる。
それはさらに、コンパイラと標準ライブラリの実装に対して、
多大な保守を強いる。それらは稀な状況下でもプログラムの基本的な健全性
(クラッシュや未定義の動作が起こらない事)
を一般に保証せねばならないからである。

We propose that Swift should instead enforce a general rule that potential
modifications of variables must be exclusive with any other access to that
variable.

Swiftはそのかわりに、
変数を変更しうるときはその他のどんなアクセスも排除されねばらない、
という一般的な規則を強制するべきだと提案する。

This proposal is a core part of the Ownership feature, which was described
in the [ownership manifesto](https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md).
That document presents the high-level objectives of Ownership, develops
a more rigorous theory of memory access in Swift, and applies it in detail
to a variety of different language features.  In that document, the rule
we're proposing here is called the Law of Exclusivity.  We will not be
going into that level of detail in this proposal.  Instead, we will
lay out the basic rule, how it will be enforced, and the implications for
programming in Swift.  It should be possible to understand this proposal
without actually having read the ownership manifesto at all.  That said,
if you are interested in the technical details, that document is probably
the right place to turn.

この提案はオーナーシップマニフェストという文書で説明されている
オーナーシップ機能の重要な部分だ。
その文書はオーナーシップの高レベルな目的を提示し、
Swiftのメモリアクセスのより厳格な理論を発展させ、
それらを様々な異なる言語機能に詳細に適用する。
ここで提案するその文書に出てくる法則を、
排他則 (Law of Exclusivity) と呼ぶ。
この提案では詳細なレベルまでは進まない。
代わりに、
それがどのように強制されるのかという基本的なルールと、
Swiftでのプログラミングへの影響を提示する。
オーナーシップマニフェストを実際には全く読んでいなくても、
この提案を理解することはできるだろう。
それはそうだが、
もしあなたが技術的な詳細に興味があるなら、
マニフェストに戻るのは正しいだろう。

## Motivation
## 動機

### Instantaneous and non-instantaneous accesses
### instantaneous と non-instantaneous なアクセス

On a basic level, Swift is an imperative language which allows programmers
to directly access mutable memory.

基本的なレベルでは、
Swiftは命令形言語で、
プログラマはミュータブルなメモリに直接アクセスできる。

Many of the language features that access memory, like simply loading
from or assigning to a variable, are "instantaneous".  This means that,
from the perspective of the current thread, the operation completes
without any other code being able to interfere.  For example, when you
assign to a stored property, the current value is just replaced with
the new value.  Because arbitrary other code can't run during an
instantaneous access, it's never possible for two instantaneous accesses
to overlap each other (without introducing concurrency, which we'll
talk about later).  That makes them very easy to reason about.

メモリにアクセスする多くの言語機能において、
変数からシンプルに読み出したり代入したりするものは、
「instantaneous」(即座)だ。
これは、現在のスレッドの視点では、
いかなる他のコードも干渉する事ができずに
操作が完了する事を意味する。
例えば、あなたが stored property に代入する時、
現在の値はただ新しい値で置き換えられる。
どんな他のコードも instantaneous なアクセスの間には実行できないので、
2つの instantaneous なアクセスがお互いに重なる事は起こり得ない。
(並行性を考えない限り。それについては後述する。)
よって instantaneous について理解するのは簡単だ。

However, not all accesses are instantaneous.  For example, when you
call a ``mutating`` method on a stored property, it's really one long
access to the property: ``self`` just becomes another way of referring
to the property's storage.  This access isn't instantaneous because
all of the code in the method executes during it, so if that code
manages to access the same property again, the accesses will overlap.
There are several language features like this already in Swift, and
Ownership will add a few more.

しかし、全てのアクセスが instantaneous ではない。
例えば、
ある stored property の mutating methodを呼ぶ時、
これは実際にはそのプロパティへの一つの長いアクセスだ: 
`self` はちょうどそのプロパティのストレージを参照するの他の手段となる。
メソッドの中の全てのコードはそのアクセスの間に実行されるので、
これは instantaneous なアクセスではなく、
もしそれらのコードが同じプロパティへの再度のアクセスを扱うなら、
アクセスは重なる。
Swiftにはこれと同じような言語機能がいくつかあるから、
オーナーシップはさらに少し追加する。

### Examples of problems due to overlap
### 重なりによる問題の例

Here's an example:
例:

```swift
// These are simple global variables.
// シンプルなグローバル変数がある。
var global: Int = 0
var total: Int = 0

extension Int {
  // Mutating methods access the variable they were called on
  // for the duration of the method.
  // メソッドが呼ばれている間の変数にアクセスする mutating method
  mutating func increaseByGlobal() {
    // Any accesses they do will overlap the access to that variable.
    // ここでのどんなアクセスもその変数へのアクセスと重なりうる

    total += self // Might access 'total' through both 'total' and 'self'
    // `total` と `self` の両方を通して `total` にアクセスしうる
    self += global // Might access 'global' through both 'global' and 'self'
    // `global` と `self` の両方を通して `global` にアクセスしうる
  }
}
```

If ``self`` is ``total`` or ``global``, the low-level semantics of this
method don't change, but the programmer's high-level understanding
of it almost certainly does.  A line that superficially seems to not
change 'global' might suddenly start doubling it!  And the data dependencies
between the two lines instantly go from simple to very complex.  That's very
important information for someone maintaining this method, who might be
tempted to re-arrange the code in ways that seem equivalent.  That kind of
maintenance can get very frustrating because of overlap like this.

このメソッドのローレベルな意味論は変わらなくても、
もし `self` が `total` か `global` であるとき、
プログラマのハイレベルな理解はほぼ確実に変わる。
表面的には `global` を変更しないように見える行が、
突然それを2倍にし始める！
そして2つの行の間のデータの依存関係は単純なものからたちまちとても複雑になる。
これは同じように見えるコードへの再整理を試みるかもしれない、
このメソッドを保守している誰かにとってとても重要な情報だ。
このような重なりのせいでこの種の保守はとても苛立たしいものになる。

The same considerations apply to the language implementation.
The possibility of overlap means the language has to make
pessimistic assumptions about the loads and stores in this method.
For example, the following code avoids a seemingly-redundant load,
but it's not actually equivalent because of overlap:

同じ考察が言語の実装にも適用される。
重なりの可能性があるということは、
言語はメソッドの中での読み書きについて悲観的な仮定をもたねばならない事を意味する。
例えば、
以下のコードは冗長に見える読み込みを避けるが、
重なりのせいで実際には等価ではない:

```swift
    let value = self
    total += value
    self = value + global
```

Because these variables just have type ``Int``, the cost of this pessimism
is only an extra load.  If the types were more complex, like ``String``,
it might mean doing extra copies of the ``String`` value, which would
translate to extra retains and releases of the string's buffer; in a more
complex example, that could even lead to the underlying data being copied
unnecessarily.

これらの変数はただの Int 型なので、この悲観のコストは余分な読み込みだけだ。
もし `String` のようなもっと複雑な型なら、
これは `Stirng` の値の余分なコピーを意味し、
それは文字列バッファの余分な retain と release になる;
さらに複雑な例では、これは内部データの不必要なコピーを引き起こすかもしれない。

In the above examples, we've made the potentially-overlapping accesses
obvious, but they don't have to be.  For example, here is another method
that takes a closure as an argument:

上記の例では、
重なる可能性のあるアクセスを明らかにしたが、
それをする必要はない。
例えば、ここにクロージャを引数に取る他のメソッドを示す:

```swift
extension Array {
  mutating func modifyElements(_ closure: (inout Element) -> ()) {
    var i = startIndex
    while i != endIndex {
      closure(&self[i])
      i = index(after: i)
    }
  }
}
```

This method's implementation seems straightforwardly correct, but
unfortunately it doesn't account for overlap.  Absolutely nothing
prevents the closure from modifying ``self`` during the iteration,
which means that ``i`` can suddenly become an invalid index, which
could lead to all sorts of unwanted behavior.  Even if this never
happen in reality, the fact that it's *possible* means that the
implementation is blocked from pursuing all sorts of important
optimizations.

このメソッドの実装は素直は正しく見えるが、
残念ながら重なりを考慮できていない。
イテレーションの間にクロージャが `self` を変更する事を防ぐものは何もないので、
これは `i` が突然不正なインデックスになってしまう事を意味し、
これはあらゆる種類の望まない振る舞いを引き起こす。
例え現実的にそれが起こらないとしても、
それが「起こりうる」という事実が意味するのは、
実装があらゆる種類の重要な最適化を達成する事を妨害されるという事だ。

For example, the compiler has an optimization that "hoists" the
uniqueness check on a copy-on-write collection from the inside of
a loop (where it's run on each iteration) to the outside (so that
it's only checked once, before the loop begins).  But that optimization
can't be applied in this example because the closure might change or
copy ``self``.  The only realistic way to tell the compiler that
that can't happen is to enforce exclusivity on ``self``.

例えば、
コンパイラは COWコレクションのユニーク性の検査を
ループの中から (イテレーションごとに実行される) 外側へ 
「巻き上げる」(だからループの前に1度だけチェックする)
最適化を持っている。
しかしその最適化はこの例には適用できない、
なぜならクロージャが `self` を変更したりコピーしたりするかもしれないからだ。
コンパイラにそのような最適化をさせる唯一の現実的な方法は、
`self` の排他性を強制してそのような事が起こらないようにする事だ。

The same considerations that apply to ``self`` in a ``mutating``
method also apply to ``inout`` parameters.  For example:

`mutating` method における `self` に適用したのと同じ考察が `inout` 引数にも適用される。
例えば:

```swift
open class Person {
  open var title: String
}

func collectTitles(people: [Person], into set: inout Set<String>) {
  for person in people {
    set.insert(person.title)
  }
}
```

This function mutates a set of strings, but it also repeatedly
calls a class method.  The compiler cannot know how this method
is implemented, because it is ``open`` and therefore overridable
from an arbitrary module.  Therefore, because of overlap, the
compiler must pessimistically assume that each of these method
calls might somehow find a way to modify the original variable
that ``set`` was bound to.  (And if the method did manage to do
so, the resulting strange behavior would probably be seen as a bug
by the caller of ``collectTitles``.)

この関数は string の set を変更するが、それだけでなくクラスメソッドを繰り返し呼ぶ。
コンパイラはこのメソッドがどのように実装されているか知ることができない、
なぜならこれが `open` で、
だからどんなモジュールからでもこれをオーバライドできるからだ。
重なりがあるので、それゆえに、
これらのメソッド呼び出しそれぞれが
`set` が束縛されている元々の変数をどうにかして変更する方法をもっていることを、
悲観的に仮定しなければならない。
(そしてもしそのメソッドがそのような事をするなら、
その変な振る舞いの結果は `collectTitles` の caller にとってはバグのように見えるだろう)

### Eliminating non-instantaneous accesses?
### non-instantaneous なアクセスを排除するか？

If non-instantaneous accesses create all of these problems with
overlapping accesses, should we just eliminate non-instantaneous
accesses completely?  Well, no, and there's two big reasons why not.
In order to make something like a ``mutating`` method not access
the original storage of ``self`` for the duration of the method,
we would need to make it access a temporary copy instead, which
we would assign back to the storage after the method is complete.
That is, suppose we had the following Swift code:

もし non-instantaneous なアクセスが
全ての 重なったアクセスによる問題を生み出すなら、
non-instantaneous なアクセスをただ完全に排除すべきか？
いや、すべきではない、それには2つの大きな理由がある。
`mutating` method のようなものが
メソッドの間に `self` の元々のストレージにアクセスしないようにするためには、
代わりに、
テンポラリなコピーにアクセスし、
メソッドが終わった後でそのストレージに代入して戻す
ようにする必要があるだろう。
つまり、次のようなSwiftのコードがあったとする:

```swift
var numbers = [Int]()
numbers.appendABunchOfStuff()
```

Currently, behind the scenes, this is implemented somewhat like
the following C code:

現在、舞台裏では、以下のCコードのようなものが実装される:

```c
struct Array numbers = _Array_init();
_Array_appendABunchOfStuff(&numbers);
```

You can see clearly how ``_Array_appendABunchOfStuff`` will be working
directly with the storage of ``numbers``, creating the abstract
possibility of overlapping accesses to that variable.  To prevent
this in general, we would need to pass a temporary copy instead:

`_Array_appendABunchOfStuff` がどのようにして、
 `numbers` のストレージを直接取扱い、
その変数へのアクセスが重なる理論上の可能性を生み出すのかはっきりとわかる。
一般にこれを防ぐためには、テンポラリなコピーを代わりに渡す必要があろう:

```c
struct Array numbers = _Array_init();
struct Array temp = _Array_copy(numbers);
_Array_appendABunchOfStuff(&temp);
_Array_assign(&numbers, temp);
```

Like we said, there's two big problems with this.

述べたように、これには大きな2つの問題がある。

The first problem is that it's awful for performance.  Even for a
normal type, doing extra copies is wasteful, but doing it with ``Array``
is even worse because it's a copy-on-write type.  The extra copy here
means that there will be multiple references to the buffer, which
means that ``_Array_appendABunchOfStuff`` will be forced to copy the
buffer instead of modifying it in place.  Removing these kinds of
copies, and making it easier to reason about when they happen, is a
large part of the goal of the Ownership feature.

第一の問題は、これはひどいパフォーマンスになることだ。
例え普通の型でも、余分なコピーをするのは無駄だが、
`Array` は COW型だからさらに悪い。

ここでの余分なコピーはバッファへの複数の参照をもたらす事を意味し、
それは `_Array_appendABunchOfStuff` がインプレースでバッファを書き換える代わりに
コピーする事を強制することを意味する。
このような種類のコピーを除去し、
それが起きる時にはわかりやすくすることは、
オーナーシップ機能の目標の大きな部分だ。

The second problem is that it doesn't even eliminate the potential
confusion.  Suppose that ``_Array_appendABunchOfStuff`` somehow
reads or writes to ``numbers`` (perhaps because ``numbers`` is
captured in a closure, or it's actually a global variable or a class
property or something else that can be potentially accessed from
anywhere).  Because the method is now modifying the copy in ``temp``,
any reads it makes from ``numbers`` won't see any of the changes
it's made to ``temp``, and any changes it makes to ``numbers`` will
be silently lost when it returns and the caller unconditionally
overwrites ``numbers`` with ``temp``.

第二の理由は混乱する可能性を除去できていないことだ。
`_Array_appendABunchOfStuff` がどうにかして `numbers` を読み書きする事を想定しよう。
(もしかすると `number` はクロージャでキャプチャされているか、
または実際にはグローバル変数かクラスプロパティかその他の何かで、
どこからでもアクセスされる可能性をもつ)
メソッドは今 `temp` の中のコピーを変更するので、
`temp` に起こったいかなる変更も `numbers` を読み取っても見えないし、
`numbers` に起こったどんな変更も
メソッドが終わって caller が無条件に `numbers` を `temp` 
で上書きする時に
黙って失われる。

### Consequences of non-instantaneous accesses
### non-instantaneous access の結論

So we have to accept that accesses can be non-instantaneous.  That
means programmers can write code that would naturally cause overlapping
accesses to the same variable.  We currently allow this to happen
and make a best effort to live with the consequences.  The costs,
in general, are a lot of complexity and lost performance.

よってアクセスが non-instantaneous になりうることを受け入れねばならない。
これはプログラマが同じ変数への重なったアクセスを自然に引き起こすコードを書ける事を意味する。
現状ではそれが起きる事を許し、その結果とうまくやるために最大の努力をしている。
一般にそのコストは大きな複雑性とパフォーマンスの低下である。   

For example, the ``Array`` type has an optimization in its ``subscript``
operator which allows callers to directly access the storage of array
elements.  This is a very important optimization which, among other
things, allows arrays to efficiently hold values of copy-on-write types.
However, because the caller can execute arbitrary code while they're
working with the array element storage, and that code might do something
like assign a new value to the original array variable and therefore drop
the last reference to the array buffer, this optimization has to create a
new strong reference to the buffer until the caller is done with the element,
which itself causes a whole raft of complexity.

例えば、 `Array` 型は caller が配列の要素のストレージに直接アクセスできるように
`subscript` オペレータの最適化を持っている。
この最適化はとりわけとても重要で、配列がCOW型の値を効率的に保持できるようにしている。
しかし、
caller が配列の要素のストレージを取り扱っている間に、
任意のコードを実行することができて、
そのコードは元々の配列変数に新しい値を代入するようなこととか、
それによって配列バッファへの最後の参照を取り除いたりするかもしれないので、
この最適化では caller が要素の扱いを終えるまでの間、
バッファへの新しい強参照を作る必要があり、
この事が大きな複雑性をもたらしている。

Similarly, when the compiler is optimizing a ``mutating`` method, it has
to assume that an arbitrary call might completely rewrite ``self``.
This makes it very difficult to perform any meaningful optimization
at all, especially in generic code.  It also means that the compiler
must generally emit a large number of conservative copies just in case
things are modified in unexpected ways.

同様に、コンパイラが `mutating` method を最適化するとき、
どんな呼び出しも `self` を完全に書き換えうることを仮定せねばならない。
これは有意義な最適化をすることをとにかくとても難しくする、、特にジェネリックなコードで。
これはまた、
コンパイラが
何かが予想外の方法で変更される場合にそなえて、
多くの保守的なコピーを一般に生成せねばならない事を意味する。

Furthermore, the possibility of overlapping accesses has a continued
impact on language evolution.  Many of the features laid out in the
Ownership manifesto rely on static guarantees that Swift simply cannot
make without stronger rules about when a variable can be modified.

さらに、アクセスの重なりの可能性は言語の進化に影響を与え続ける。
オーナーシップマニフェストに提示されている多くの機能は、
Swiftに
変数が変更できる時についてのより強い規則がなしでは
作ることができない
静的な保証に依存している。

Therefore we think it best to simply disallow overlapping accesses
as best as we can.

だからできるかぎりアクセスの重なりをただ禁止するのが良いであろう。

## Proposed solution
## 提案するソリューション

We should add a rule to Swift that two accesses to the same variable
are not allowed to overlap unless both accesses are reads.  By
"variable", we mean any kind of mutable memory: global variables,
local variables, class and struct properties, and so on.

同じ変数への2つのアクセスが両方 reads でないかぎり重なる事を禁止する規則を
Swiftに追加すべきだろう。
「variable」とは、あらゆる変更可能なメモリを意味する:
グローバル変数、ローカル変数、クラスと struct のプロパティ、など。

This rule should be enforced as strongly as possible, depending on
what sort of variable it is:

このルールは変数の種類に応じてできるかぎり強力に強制される:

* Local variables, inout parameters, and struct properties can
  generally enforce the rule statically.  The compiler can analyze
  all the accesses to the variable and emit an error if it sees
  any conflicts.

  ローカル変数、inout引数、そして struct プロパティは一般に静的に規則を強制される。
  コンパイラは変数への全てのアクセスを解析し、何か衝突を見つければエラーを吐くことができる。

* Class properties and global variables will have to enforce the
  rule dynamically.  The runtime can keep track of what accesses
  are underway and report any conflicts.  Local variables will
  sometimes have to use dynamic enforcement when they are
  captured in closures.

  クラスプロパティとグローバル変数は動的に規則を強制されるべきだろう。
  ランタイムはどんなアクセスが動作中かを追跡しつづける事ができて、
  何か衝突があれば報告する。
  ランタイムは動作中のアクセスを追跡し、何か衝突があれば報告することができる。
  ローカル変数がクロージャでキャプチャされているときは、
  動的な強制を使わねばならないときもあろう。

* Unsafe pointers will not use any active enforcement; it is the
  programmer's responsibility to follow the rule.

  アンセーフポインタはどんな強制も使われない;
  規則に従うのはプログラマの責任だ。

* No enforcement is required for immutable memory, like a ``let``
  binding or property, because all accesses must be reads.

  let束縛やプロパティなどの変更不可能なメモリにはどんな強制も必要ない、
  なぜなら全てのアクセスが reads だからだ。

Examples:
例:

```swift
var x = 0, y = 0

// NOT A CONFLICT.  These two accesses to 'x' are both reads.
// Each completes instantaneously, so the accesses do not overlap and
// therefore do not conflict.  Even if they were not instantaneous, they
// are both reads and therefore do no conflict.
//
// 衝突しない。
// これらの 'x' への2つのアクセスは両方とも read だ。
// それぞれ即座に終わり、だからアクセスは重ならない、だから衝突しない。 
// たとえこれらが即座でなくても、両方読み取りだから衝突しない。
let z = x + x

// NOT A CONFLICT.  The right-hand side of the assignment is a read of
// 'x' which completes instantaneously.  The assignment is a write to 'x'
// which completes instantaneously.  The accesses do not overlap and
// therefore do not conflict.
//
// 衝突しない。
// 代入の右辺は 'x' の read で即座に終わる。
// 代入は 'x' への write で即座に終わる。
// アクセスは重なっていないので衝突しない。
x = x

// NOT A CONFLICT.  The right-hand side is a read of 'x' which completes
// instantaneously.  Calling the operator involves passing 'x' as an inout
// argument; this is a write access for the duration of the call, but it does
// not begin until immediately before the call, after the right-hand side is
// fully evaluated.  Therefore the accesses do not overlap and do not conflict.
// 
// 衝突しない。
// 右辺は 'x' の read で即座に終わる。
// オペレータの呼び出しは 'x' をinout引数として渡す事を引き起こす;
// これは呼び出しの間の書き込みアクセスだが、
// 右辺が完全に評価された後の、
// 呼び出しの直前まで始まらない。
// だからアクセスが重なる事はなく衝突しない。
x += x

// CONFLICT.  Passing 'x' as an inout argument is a write access for the
// duration of the call.  Passing the same variable twice means performing
// two overlapping write accesses to that variable, which therefore conflict.
// 
// 衝突する。
// 'x' をinout引数に渡すのは呼び出しの間の書き込みアクセスだ。
// 同じ変数を2回渡すのはその変数への2つの重なった書き込みアクセスをもたらすので、
// それは衝突する。
swap(&x, &x)

extension Int {
  mutating func assignResultOf(_ function: () -> Int) {
    self = function()  
  }
}

// CONFLICT.  Calling a mutating method on a value type is a write access
// that lasts for the duration of the method.  The read of 'x' in the closure
// is evaluated while the method is executing, which means it overlaps
// the method's formal access to 'x'.  Therefore these accesses conflict.
//
// 衝突する。
// 値型の mutating method を呼び出すのはメソッド呼び出しの間続く書き込みアクセスだ。
// クロージャの中での 'x' の読み取りはメソッドが実行されている間に評価され、
// これはメソッドの 'x' への形式的アクセスと重なることになる。
// だからこれらのアクセスは衝突する。
x.assignResultOf { x + 1 }
```

## Detailed design
## 詳細設計

### Concurrency
### 並行性

Swift has always considered read/write and write/write races on the same
variable to be undefined behavior.  It is the programmer's responsibility
to avoid such races in their code by using appropriate thread-safe
programming techniques.

Swiftは同じ変数への read/write と write/write のレースは常に未定義の動作とみなす。
コードの中でのこうしたレースを
適切なスレッドセーフプログラミングのテクニックを使って回避するのはプログラマの責任だ。

We do not propose changing that.  Dynamic enforcement is not required to
detect concurrent conflicting accesses, and we propose that by default
it should not make any effort to do so.  This should allow the dynamic
bookkeeping to avoid synchronizing between threads; for example, it
can track accesses in a thread-local data structure instead of a global
one protected by locks.  Our hope is that this will make dynamic
access-tracking cheap enough to enable by default in all programs.

これを変更する提案はしない。
動的な強制は並行アクセスの衝突を検出する必要はなく、
それをするための一切の努力もデフォルトではしないことを提案する。
これは動的なアクセス記録をスレッド間で同期する事を避けることを許すだろう;
例えば、ロックによって保護されたグローバルなものを使う代わりに、
スレッドローカルデータ構造の中にアクセスを記録できる。
これが動的なアクセス追跡を全てのプログラムでデフォルト有効にできるほど軽量にすると期待する。

The implementation should still be *permitted* to detect concurrent
conflicting accesses, of course.  Some programmers may wish to use an
opt-in thread-safe enforcement mechanism instead, at least in some
build configurations.

もちろん、並行コンフリクトアクセスも検出できる実装もまた「許可されて」いる。
いくらかのプログラマは、
代わりにオプトインのスレッドセーフな強制機構を使うことを望むかもしれない、
少なくともいくつかのビルド設定によって。

Any future concurrency design in Swift will have the elimination
of such races as a primary goal.  To the extent that it succeeds,
it will also define away any specific problems for exclusivity.

Swiftの並行設計の未来においてはこうした競合の除去も目標となりうる。
これを進める領域においては、特定の排他性の問題をまた定義するかもしれない。

### Value types
### 値型

Calling a method on a value type is an access to the entire value:
a write if it's a ``mutating`` method, a read otherwise.  This is
because we have to assume that a method might read or write an
arbitrary part of the value.  Trying to formalize rules like
"this method only uses these properties" would massively complicate
the language.

値型のメソッドの呼び出しは値全体へのアクセスだ:
もしそれが `mutating` メソッドならそれは write で、そうでなければ read だ。
これはメソッドが値の任意の部分への読み込みか書き込みを行うという事を
仮定せねばならない理由となる。
「このメソッドはこれらのプロパティしか使わない」といった規則を形式化しようとすれば、
言語はとても複雑になってしまうだろう。

For similar reasons, using a computed property or subscript on a
value type generally has to be treated as an access to the entire
value.  Whether the access is a read or write depends on how the
property/subscript is used and whether either the getter or the setter
is ``mutating``.

似たような理由で、
値型の computed property や subscript も
一般には値全体へのアクセスとして扱わねばならない。
アクセスが読み込みか書き込みかなのかは プロパティ/subscript が
どのように使われるかと getter か setter が `mutating` かどうかによる。

Accesses to different stored properties of a ``struct`` or different
elements of a tuple are allowed to overlap.  However, note that
modifying part of a value type still requires exclusive access to
the entire value, and that acquiring that access might itself prevent
overlapping accesses.  For example:

`struct` の異なる stored property かタプルの異なる要素へのアクセスは重なっても良い。
しかし、
値型の一部の変更はそれでも値全体への排他的アクセスを要求する事に注意せよ、
なのでそうしたアクセスの取得はそれ自身を重なるアクセスから阻害する。

```swift
struct Pair {
  var x: Int
  var y: Int
}

class Paired {
  var pair = Pair(x: 0, y: 0)
}

let object = Paired()
swap(&object.pair.x, &object.pair.y)
```

Here, initiating the write-access to ``object.pair`` for the first
argument will prevent the write-access to ``object.pair`` for the
second argument from succeeding because of the dynamic enforcement
used for the property.  Attempting to make dynamic enforcement
aware of the fact that these accesses are modifying different
sub-components of the property would be prohibitive, both in terms
of the additional performance cost and in terms of the complexity
of the implementation.

ここで、
プロパティに使われる動的強制によって、
第一引数の `object.pair` への書き込みアクセスの初期化は、
続く第二引数の `object.pair` への書き込みアクセスを阻害する。
動的強制がこれらのアクセスが
プロパティの異なる部分要素を変更しているという事実を
認識できるようにする試みは禁止したい。
追加のパフォーマンスコストの観点と、実装の複雑さの観点からだ。

However, this limitation can be worked around by binding
``object.pair`` to an ``inout`` parameter:

しかし、この制限は `object.pair` を `inout` 引数に束縛する事で対処できる。

```swift
func modifying<T>(_ value: inout T, _ function: (inout T) -> ()) {
  function(&value)
}

modifying(&object.pair) { pair in swap(&pair.x, &pair.y) }
```

This works because now there is only a single access to
``object.pair`` and because, once the the ``inout`` parameter is
bound to that storage, accesses to the parameter within the
function can use purely static enforcement.

これが動く理由は、
いま `object.pair` へのアクセスはたったひとつしか無く、
`inout` 引数がそのストレージに束縛されれば、
関数の中での引数へのアクセスは純粋に静的な強制が使われるからだ。

We expect that workarounds like this will only rarely be required.

このような回避策はまれにしか必要にならないと期待する。

Note that two different properties can only be assumed to not
conflict when they are both known to be stored.  This means that,
for example, it will not be allowed to have overlapping accesses
to different properties of a resilient value type.  This is not
expected to be a significant problem for programmers.

2つの異なるプロパティは、
それらが両方共 stored だとわかっている時にだけ衝突しないと
仮定できる事に注意せよ。
これが意味するのは例えば、
レジリエントな値型の異なるプロパティには同時にアクセスできないということだ。
これはプログラマにとって影響の大きな問題にはならないと期待する。

### Arrays
### 配列

Collections do not receive any special treatment in this proposal.
For example, ``Array``'s indexed subscript is an ordinary computed
subscript on a value type.  Accordingly, mutating an element of an
array will require exclusive access to the entire array, and
therefore will disallow any other simultaneous accesses to the
array, even to different elements.  For example:

コレクションはこの提案においてどんな特別な扱いも受けない。
例えば、 `Array` のインデックスされた subscript は
値型の普通の computed subscript だ。
結果的に、配列の要素の変更は配列全体への排他的なアクセスを要求し、
よって他のどんな同時のアクセスも許さない、
たとえそれが異なる要素であっても。例えば:

```swift
var array = [[1,2], [3,4,5]]

// NOT A CONFLICT.  These accesses to the elements of 'array' each
// complete instantaneously and do not overlap each other.  Even if they
// did overlap for some reason, they are both reads and therefore
// do not conflict.
//
// 衝突しない。
// これらの 'array' の要素へのアクセスはいずれも即座に終わり、
// お互いに重ならない。
// たとえこれらがなんらかの理由で重なるとしても、
// 両方とも読み取りだから衝突しない。
print(array[0] + array[1])

// NOT A CONFLICT.  The access done to read 'array[1]' completes
// before the modifying access to 'array[0]' begins.  Therefore, these
// accesses do not conflict.
//
// 衝突しない。
// 'array[1]' の read のアクセスは
// 'array[0]' を 変更するアクセスが始まる前に終わる。
// だからこれらのアクセスはコンフリクトしない。
array[0] += array[1]

// CONFLICT.  Passing 'array[i]' as an inout argument performs a
// write access to it, and therefore to 'array', for the duration of
// the call.  This call makes two such accesses to the same array variable,
// which therefore conflict.
//
// 衝突する。
// 'array[i]' をinout引数として渡すことはそれへの書き込みアクセスを起こすので、
// よってそれは呼び出しの間の 'array' へのアクセスとなる
// この呼び出しは同一の配列変数へのそのような2つのアクセスを作るので、
// それらはコンフリクトする。
swap(&array[0], &array[1])

// CONFLICT.  Calling a non-mutating method on 'array[0]' performs a
// read access to it, and thus to 'array', for the duration of the method.
// Calling a mutating method on 'array[1]' performs a write access to it,
// and thus to 'array', for the duration of the method.  These accesses
// therefore conflict.
//
// 衝突する。
// 'array[0]' への non-mutating method の呼び出しは、
// それへの読み取りアクセスを起こし、
// それはメソッドの間の 'array' への読み取りアクセスとなる。
// 'array[1]' への mutating method の呼び出しはそれへの書き込みアクセスを起こし、
// それはメソッドの間の 'array' への書き込みアクセスとなる。
// だからこれらのアクセスはコンフリクトする。
array[0].forEach { array[1].append($0) }
```

It's always been somewhat fraught to do simultaneous accesses to
an array because of copy-on-write.  The fact that you should not
create an array and then fork off a bunch of threads that assign
into different elements concurrently has been independently
rediscovered by a number of different programmers.  (Under this
proposal, we will still not be reliably detecting this problem
by default, because it is a race condition; see the section on
concurrency.)  The main new limitation here is that some idioms
which did work on a single thread are going to be forbidden.
This may just be a cost of progress, but there are things we
can do to mitigate the problem.

COWのせいで配列への同時のアクセスはいつもいくらか困難だ。
配列を作り
異なる要素へ並行に代入するたくさんのスレッドを分岐させるべきではない
という事実はたくさんの異なるプログラマによって独立に再発見されてきた。
(この提案においても、デフォルトではこの問題はいまだ確実に検出されない、
なぜならこれはレースコンディションだからだ; 並行性についての章を見よ。)
ここでの主な新しい制限はシングルスレッドで動くいくつかのイディオムを禁止することだ。
これはただの進めるコストになるかもしれないが、問題を軽減できる事がある。

In the long term, the API of ``Array`` and other collections
should be extended to ensure that there are good ways of achieving
the tasks that exclusivity enforcement has made difficult.
It will take experience living with exclusivity in order to
understand the problems and propose the right API additions.
In the short term, these problems can be worked around with
``withUnsafeMutableBufferPointer``.

長期的には、`Array` や他のコレクションのAPIは、
排他性の強制が難しいという課題を解決する良い方法を提供するように拡張されるだろう。
問題を理解して正しいAPIの追加を提案するために排他性を取り扱う経験が必要だろう。
短期的には、そうした問題は `withUnsafeMutableBufferPointer` で対応できるだろう。

We do know that swapping two array elements will be problematic,
and accordingly we are (separately proposing)[https://github.com/apple/swift-evolution/blob/master/proposals/0173-swap-indices.md] to add a
``swapAt`` method to ``MutableCollection`` that takes two indices
rather than two ``inout`` arguments.  The Swift 3 compatibility
mode should recognize the swap-of-elements pattern and automatically
translate it to use ``swapAt``, and the 3-to-4 migrator should
perform this rewrite automatically.

2つの配列の要素を入れ替えることが問題になることはわかっていたので、
だから2つの `inout` 引数ではなく2つのインデックスを取る
`swapAt` メソッドを `MutableCollection` に追加した(分割された提案 SE-0173)。
Swift3互換モードは要素の入れ替えパターンを
検出して `swapAt` を使うように自動変換するべきだ、
そして 3-to-4 マイグレータはこの書き換えを自動的にやるべきだ。

### Class properties
### クラスプロパティ

Unlike value types, calling a method on a class doesn't formally access
the entire class instance.  In fact, we never try to enforce exclusivity
of access on the whole object at all; we only enforce it for individual
stored properties.  Among other things, this means that an access to a
class property never conflicts with an access to a different property.

値型と違って、
クラスのメソッドの呼び出しは
クラスインスタンス全体への形式的なアクセスでは無い。
事実、オブジェクト全体へのアクセスの排他の強制は全く試みない;
個別の stored property への強制だけを行う。
何よりも、
これはクラスプロパティへのアクセスが
異なるプロパティへのアクセスと絶対にコンフリクトしない事を意味する。

There are two major reasons for this difference between value
and reference types.

値型と参照型の間のこの違いには大きな2つの理由がある。

The first reason is that it's important to allow overlapping method
calls to a single class instance.  It's quite common for an object
to have methods called on it concurrently from different threads.
These methods may access different properties, or they may synchronize
their accesses to the same properties using locks, dispatch queues,
or some other thread-safe technique.  Regardless, it's a widespread
pattern.

第一の理由は一つのクラスインスタンスへの
重なったメソッド呼び出しを許すことが重要だからだ。
オブジェクトにとって異なるスレッドから
並行に呼び出されるメソッドを持つことはとてもよくある。
これらのメソッドは異なるプロパティにアクセスするだろう、
または、
ロックやディスパッチキューやその他のスレッドセーフテクニックを使って
同じプロパティへのアクセスを同期化するだろう。
とにかく、これはよくあるパターンだ。

The second reason is that there's no benefit to trying to enforce
exclusivity of access to the entire class instance.  For a value
type to be mutated, it has to be held in a variable, and it's
often possible to reason quite strongly about how that variable
is used.  That means that the exclusivity rule that we're proposing
here allows us to make some very strong guarantees for value types,
generally making them an even tighter, lower-cost abstraction.
In contrast, it's inherent to the nature of reference types that
references can be copied pretty arbitrarily throughout a program.
The assumptions we want to make about value types depend on having
unique access to the variable holding the value; there's no way
to make a similar assumption about reference types without knowing
that we have a unique reference to the object, which would
radically change the programming model of classes and make them
unacceptable for the concurrent patterns described above.

第二の理由はクラスインスタンス全体へのアクセスの排他性を強制する事を
試みることに何の利益もないからだ。
値型が変更される時、
これは変数に保持され、
変数がどのように使われるか非常に強力に考えることがしばしば可能だ。
これはここで提案する排他のルールが、
値型について低コストの抽象化で一般にそれらを十分に厳しく、
とても強い保証を作ることができることを意味する。
対して、参照がプログラムの任意のいたるところで行儀よくコピーできる事は、
参照型の本来の性質だ。
値型について作りたい仮定は値を保持している変数へのユニークなアクセスに依存している。
参照型についてはオブジェクトへのユニークな参照を持つことを知ることなしに
同様な仮定をおく方法はなく、
これはクラスについてのプログラミングモデルを根本的に変更するし、
上述したような並行パターンにおいてはこれらは受け入れられない。

### Disabling dynamic enforcement.
### 動的強制の無効化

We could add an attribute which allows dynamic enforcement to
be downgraded to an unsafe-pointer-style undefined-behavior rule
on a variable-by-variable basis.  This would allow programmers to
opt out of the expense of dynamic enforcement when it is known
to be unnecessary (e.g. because exclusivity is checked at some
higher level) or when the performance burden is simply too great.

動的強制を変数は変数であるという原理にもとづいて
アンセーフポインタスタイルの未定義動作に
ダウングレードさせる修飾子を追加してもよいだろう。
これはプログラマが必要ない(排他性のチェックはいくらか高いレベルで行われるので)とわかっている場合や
パフォーマンスの負担が大きすぎる場合に動的強制のコストをオプトアウトできるようにする。

There is some concern that adding this attribute might lead to
over-use and that we should only support it if we are certain
that the overheads cannot be reduced in some better way.

この修飾子を追加する事でこれの過剰使用をもたらす懸念があり、
なにかの良い方法でオーバーヘッドを減らすことができない事が明白な場合にだけこれをサポートするべきだろう。

Since the rule still applies, and it's merely no longer being checked,
it makes sense to borrow the "checked" and "unchecked" terminology
from the optimizer settings.

それでも規則は適用されるので、
ただ単にチェックされなくなるだけであり、
最適化設定の「checked」と「unchecked」という用語を借りるのが筋だろう。

```swift
class TreeNode {
  @exclusivity(unchecked) var left: TreeNode?
  @exclusivity(unchecked) var right: TreeNode?
}
```

### Closures
### クロージャ

A closure (including both local function declarations and closure
expressions, whether explicit or autoclosure) is either "escaping" or
"non-escaping".  Currently, a closure is considered non-escaping
only if it is:

クロージャ(ローカル関数宣言と、明示的かオートクロージャによるクロージャ式の両方)は「escaping」か「non-escaping」のどちらかだ。
現在、クロージャはもし次の通りのときだけ non-escaping となる:

- a closure expression which is immediately called,

  クロージャ式が即座に呼ばれている,

- a closure expression which is passed as a non-escaping function
  argument, or

  クロージャ式が non-escaping な関数の引数として渡されている、または

- a local function which captures something that is not allowed
  to escape, like an ``inout`` parameter.

  ローカル関数が `inout` 引数のようなエスケープできないなにかをキャプチャしている

It is likely that this definition will be broadened over time.

この定義は時間をかけて広がっていっただろう。

Local variables which are captured in escaping closures will
generally require dynamic enforcement instead of static enforcement
because the implementation cannot locally reason about when the
closure will be called and thus when the variable will be accessed.
Swift may be able to use static enforcement, but only as a best-effort
improvement for certain patterns of access.

エスケープするクロージャにキャプチャされているローカル変数は
一般に静的強制の代わりに動的強制が必要となるだろう、
なぜなら実装がそのクロージャがいつ呼び出され、そしてその変数にいつアクセスするのか、
その場で知ることができないからだ。
アクセスのいくつかのパターンを改善する努力として、
Swiftは静的強制を使うことができる場合もある。

Local variables which are only captured in non-escaping closures will
always use static enforcement.  (In order to achieve this, we
must impose a new restriction on recursive uses of non-escaping
closures; see below.)  This guarantee aligns a number of related
semantic and performance goals.  For example, a local variable which
is not captured in an escaping closure does not need to be allocated
on the heap; this guarantee therefore ensures that it will have
roughly C-like performance.  This guarantee also ensures that only
static enforcement is needed for ``inout`` parameters, which cannot
be captured in escaping closures; this substantially simplifies
the implementation model for capturing ``inout`` parameters.

ローカル変数が non-escaping なクロージャにのみキャプチャされているときは
いつも静的強制を使う。
(これを達成するため、
non-escaping なクロージャの再帰的使用についての新しい制約を課す必要がある;
以下を見よ)
この保証は多くの関係する意味的とパフォーマンスの目標を調整する。
例えば、エスケープするクロージャにキャプチャされていないローカル変数は、
ヒープ上に確保する必要はない;
この保証はだからおおよそC言語のようなパフォーマンスを確保する。
この保証は
エスケープするクロージャによってキャプチャできない `inout` 引数について
静的保証のみが必要となることを保証する;
これは `inout` 引数をキャプチャする実装モデルを大幅にシンプルにする。

### Diagnosing dynamic enforcement violations statically
### 動的強制違反の静的な診断

In general, Swift is permitted to upgrade dynamic enforcement to
static enforcement when it can prove that two accesses either
always or never conflict.  This is analogous to Swift's rules
about integer overflow.

一般にSwiftは
2つのアクセスが絶対に衝突するかしないかを証明できる場合に、
動的強制を静的強制に昇格することが許されている。
これはSwiftの整数オーバーフローについての規則に似ている。

For example, if Swift can prove that two accesses to a global
variable will always conflict, then it can report that error
statically, even though global variables use dynamic enforcement:

例えば、もしSwiftがあるグローバル変数への2つのアクセスが常に衝突する事を証明できるなら、
グローバル変数は動的強制を使うものであるけれども、そのエラーを静的に報告できる。

```swift
var global: Int
swap(&global, &global) // Two overlapping modifications to 'global'
// 2つの 'global' の変更が重なっている
```

Swift is not required to prove that both accesses will actually
be executed dynamically in order to report a violation statically.
It is sufficient to prove that one of the accesses cannot ever
be executed without causing a conflict.  For example, in the
following example, Swift does not need to prove that ``mutate``
actually calls its argument function:

Swiftは違反を静的に報告するために、
その2つのアクセスが実際に動的に実行される事を証明する必要はない。
あるアクセスが衝突を起こさずに実行する事が絶対にできないことを証明できれば十分だ。
例えば、以下の例で、
Swift は `mtuate` が実際にその引数の関数を呼び出す事を証明する必要はない。

```swift
// The inout access lasts for the duration of the call.
// inout アクセスが呼び出しの間続く。
global.mutate { return global + 1 }
```

When a closure is passed as a non-escaping function argument
or captured in a closure that is passed as a non-escaping function
argument, Swift may assume that any accesses made by the closure
will be executed during the call, potentially conflicting with
accesses that overlap the call.

あるクロージャが non-escaping な関数の引数として渡されているか、
non-escaping な関数の引数に渡されているクロージャによってキャプチャされているとき、
Swiftはそのクロージャによって作られるどんなアクセスも
呼び出しの間に実行され、
重なった呼び出しのアクセスが衝突する可能性があることを仮定するかもしれない。

### Restrictions on recursive uses of non-escaping closures
### non-escaping なクロージャの再帰的使用の制限

In order to achieve the goal of guaranteeing the use of static
enforcement for variables that are captured only by non-escaping
closures, we do need to impose an additional restriction on
the use of such closures.  This rule is as follows:

non-escaping なクロージャだけによってキャプチャされる
変数に対する静的強制を使用する事を保証する目標を達成するために、
そうしたクロージャの使用に追加の制限を課す必要がある。
このルールは以下のものだ:

  A non-escaping closure ``A`` may not be recursively invoked
  during the execution of a non-escaping closure ``B`` which
  captures the same local variable or ``inout`` parameter unless:

  non-escaping なクロージャ `A` が、
  同じローカル変数か `inout` 引数をキャプチャする 
  non-escaping なクロージャ `B` 
  の実行の間に
  再帰的に実行できない、
  以下でない限り:

    - ``A`` is defined within ``B`` or

      `A` が `B` の中で定義されている,または

    - ``A`` is a local function declaration which is referenced
      directly by ``B``.

      `A` は `B` によって直接参照されるローある関数宣言である

This rule is sufficient to establish that the variables captured
by ``B`` will not be interfered with unless ``B`` delegates
to something which is locally known by ``B`` to have access to
those variables.  This, together with the fact that the uses of
``B`` itself can be statically analyzed by its defining function,
is sufficient to allow static enforcement for the variables it
captures.

この規則は
`B` が自身がローカルに知る何かにそれらの変数へのアクセスを与えなければ、
`B` によってキャプチャされたその変数が
干渉されない事を確立するためには十分だ。
だから、
`B` の使用それ自体が
それを定義する関数によって静的に解析される事と合わせて、
それがキャプチャする変数に静的強制を使えることを満たす。

Because of the tight restrictions on how non-escaping closures
can be used in Swift today, it's actually quite difficult to
violate this rule.  The following user-level restrictions are
sufficient:

現在のSwiftでどのように non-escaping クロージャが使用できるかについての
厳しい制限のおかげで、
実際にはこれらの規則を犯す事はとてもむずかしい。
次のユーザーレベルの制限は十分だ:

  - A function may not call a non-escaping function parameter
    passing a non-escaping function parameter as an argument.

    関数は non-escaping な関数を引数として渡しつつ、
    引数で受けた non-escaping な関数を呼び出すことはできない。

    For the purposes of this rule, a closure which captures a
    non-escaping function parameter is treated as if it were
    that parameter.

    この規則の目的では、
    引数の non-escaping な関数をキャプチャするクロージャは、
    それがまるでその引数であるかのように扱われる。

  - Programmers using ``withoutActuallyEscaping`` should take
    care not to allow the result to be recursively invoked.

    `withoutActuallyEscaping` を使うプログラマは
    結果が再帰的に呼び出されないように注意せねばならない。    

This restriction is a conservative over-approximation.  Code
that needs to work around it can declare the function parameter
``@escaping`` or use ``withoutActuallyEscaping``, assuming
that the closures do not violate the broader rule above.

この制約は保守的で過剰な近似だ。
これを回避する必要があるコードは
クロージャが上記の広い規則に違反していない事を過程して、
関数引数を `@escaping` に宣言するか `withoutActuallyEscaping` を使用できる。

## Source compatibility
## ソース互換性

In order to gain the performance and language-design benefits of
exclusivity, we will have to enforce it in all language modes.
Therefore, exclusivity will eventually demand a source break.

排他性によるパフォーマンスと言語設計上の利益を得るため、
言語の全てのモードにこれを強制する必要がある。
よって、排他性は最終的にソース破壊をもたらす。

We can mitigate some of the impact of this break by implicitly migrating
code matching certain patterns to use different patterns that are known
to satisfy the exclusivity rule.  For example, it would be straightforward
to automatically translate calls like ``swap(&array[i], &array[j])`` to
``array.swapAt(i, j)``.  Whether this makes sense for any particular
migration remains to be seen; for example, ``swap`` does not appear to be
used very often in practice outside of specific collection algorithms.

いくらかのパターンとマッチするコードから
排他則を満たす知られた異なるパターンの使用への
暗黙なマイグレーションによって
この破壊による影響のいくらかを和らげることができる。
例えば、
`swap(&array[i], &array[j])` のような呼び出しは
`array.swapAt(i, j)` への素直な自動変換ができる。
考えられる他の特定のマイグレーションがあろうとなかろうとこれは筋が通る;
例えば、swapは特定のコレクションアルゴリズムの外では実際にはめったに使われない。

Overall, we do not expect that a significant amount of code will violate
exclusivity.  This has been borne out so far by our testing.  Often the
examples that do violate exclusivity can easily be rewritten to avoid
conflicts.  In some of these cases, it may make sense to do the rewrite
automatically to avoid source-compatibility problems.

概して、排他則を犯すコードはそんなにたくさん無いと期待する。
テストした結果そんなにたくさんでなかった。
排他則を犯す例はしばしば簡単に回避するように書き換えられる。
いくつかのケースでは、ソース互換性の問題を避けるために自動書き換えするのが良いだろう。

## Effect on ABI stability and resilience
## ABI 安定性とレジリエンスへの影響

In order to gain the performance and language-desing benefits of
exclusivity, we must be able to assume that it is followed faithfully
in various places throughout the ABI.  Therefore, exclusivity must be
enforced before we commit to a stable ABI, or else we'll be stuck with
the current conservatism around ``inout`` and ``mutating`` methods
forever.

排他性のパフォーマンスと言語設計上の利益を得るため、
ABIの様々ないたるところでこの法則にきちんとしたがうことを仮定できるべきだ。
だから、排他性はABIが安定する前に強制されねばならず、
そうしなければ現在の `inout` と `mutating` 
メソッドの保守性の前に永遠に行き詰まってしまうだろう。
