# この文書について

この文書は僕がSwiftの[Ownership Manifesto](https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md)を読解する際に作った和訳です。

元にしたバージョンは [7eb7d5b10977696c1f822ef55caaa65278c9aee8](https://github.com/apple/swift/blob/7eb7d5b10977696c1f822ef55caaa65278c9aee8/docs/OwnershipManifesto.md) です。

```
Commits on Apr 19, 2017
 @practicalswift
[gardening] Fix 100 typos.
practicalswift committed 23 days ago  
7eb7d5b  
```

うまく訳せていない変な日本語になっている部分については助けてくれると嬉しいです。

ライセンスは[Swiftリポジトリ本体のApache License](https://github.com/apple/swift/blob/master/LICENSE.txt)を継承しています。

---

# Ownership
# オーナーシップ

## Introduction
## 導入

Adding "ownership" to Swift is a major feature with many benefits
for programmers.  This document is both a "manifesto" and a
"meta-proposal" for ownership: it lays out the basic goals of
the work, describes a general approach for achieving those goals,
and proposes a number of specific changes and features, each of
which will need to be separately discussed in a smaller and more
targeted proposal.  This document is intended to provide a framework
for understanding the contributions of each of those changes.

「オーナーシップ」をSwiftに追加することはプログラマにとって多くの利益がある主要な機能だ。
この文書はオーナーシップについての「声明」であり「メタ提案」だ。
作業の基本的な目標を提示し、
それを達成する一般的なアプローチを説明し、
多くの仕様の変更と機能を提案する。
それらはより小さく個別の提案を目指して分割して議論される必要があるだろう。
この文書はそれらの変更それぞれの貢献を理解するための枠組みを提供する事を意図している。

### Problem statement
### 問題の提示

The widespread use of copy-on-write value types in Swift has generally
been a success.  It does, however, come with some drawbacks:

Swiftで広く使われているCOW値型が一般に成功してきた。けれども、いくらかの難点もある。

* Reference counting and uniqueness testing do impose some overhead.

  参照カウントとユニーク性の検査はいくらかのオーバーヘッドを与える。

* Reference counting provides deterministic performance in most cases,
  but that performance can still be complex to analyze and predict.

  参照カウントはほとんどの場合でdeterministic(決定論的な)な性能を提供するが、
  しかしそれでも解析と予測が複雑になってしまうこともある。

* The ability to copy a value at any time and thus "escape" it forces
  the underlying buffers to generally be heap-allocated.  Stack allocation
  is far more efficient but does require some ability to prevent, or at
  least recognize, attempts to escape the value.

  いつでも値をコピーできる、つまり「エスケープ」できる能力は、
  下請けバッファが一般にヒープ確保されることを強制する。
  スタック確保はそれよりはるかに効率的だが、
  それは値をエスケープする試みを阻むか、
  少なくとも認識するなんらかの能力を必要とする。

Certain kinds of low-level programming require stricter performance
guarantees.  Often these guarantees are less about absolute performance
than *predictable* performance.  For example, keeping up with an audio
stream is not a taxing job for a modern processor, even with significant
per-sample overheads, but any sort of unexpected hiccup is immediately
noticeable by users.

いくらかの種類のローレベルプログラミングではより厳密な性能保証を必要とする。
それはしばしば絶対性能よりも「予測可能な」性能を求める。
例えば、音声ストリームを維持しつづけるのは、
たとえサンプルごとのオーバヘッドが大きなものであっても、
現代のプロセッサには重い処理ではないが、
予想外のひっかかりはどんな種類のものでもユーザーに即座に気づかれる。

Another common programming task is to optimize existing code when something
about it falls short of a performance target.  Often this means finding
"hot spots" in execution time or memory use and trying to fix them in some
way.  When those hot spots are due to implicit copies, Swift's current
tools for fixing the problem are relatively poor; for example, a programmer
can fall back on using unsafe pointers, but this loses a lot of the
safety benefits and expressivity advantages of the library collection types.

その他のよくあるプログラミングタスクとして、
ターゲットの性能に達しないとき、
既存のコードを最適化することがある。
これはしばしば実行時間やメモリ使用の「ホットスポット」を探して、
なんらかの方法でそれを修正する作業だ。
これらのホットスポットが暗黙のコピーのせいであるとき、
この問題に対処するための現在のSwiftの機能は相対的に貧弱だ。
例えば、プログラマはアンセーフポインタの利用にフォールバックしうるが、
これは多くの安全性の利益とライブラリのコレクション型の表現性の優位性を失うことになる。

We believe that these problems can be addressed with an opt-in set of
features that we collectively call *ownership*.

「オーナーシップ」とまとめて名付けたオプトインの機能群によって、
こうした対処できるようになると信じる。

### What is ownership?
### オーナーシップとは

*Ownership* is the responsibility of some piece of code to
eventually cause a value to be destroyed.  An *ownership system*
is a set of rules or conventions for managing and transferring
ownership.

「オーナーシップ」は最終的に値の破棄をもたらすいくらかのコード片がもつ責任のことだ。
「オーナーシップシステム」はオーナーシップを管理、転送する規則や規約の集合のことだ。

Any language with a concept of destruction has a concept of
ownership.  In some languages, like C and non-ARC Objective-C,
ownership is managed explicitly by programmers.  In other
languages, like C++ (in part), ownership is managed by the
language.  Even languages with implicit memory management still
have libraries with concepts of ownership, because there are
other program resources besides memory, and it is important
to understand what code has the responsibility to release
those resources.

破棄の概念を持ついかなる言語もオーナーシップの概念を持つ。
CとnonARCのObjectiveCのようないくつかの言語では、
オーナーシップはプログラマによって明示的に管理される。
C++(のある部分において)のようなその他の言語では、
オーナーシップは言語によって管理される。
たとえ暗黙のメモリ管理をもつ言語であっても、
オーナーシップの概念をもつライブラリは持っている。
なぜならメモリ以外のプログラムリソースが存在するし、
どのコードがそうしたリソースを解放する責任を担っているのか理解する事は重要だからだ。

Swift already has an ownership system, but it's "under the covers":
it's an implementation detail that programmers have little
ability to influence.  What we are proposing here is easy
to summarize:

Swiftはオーナーシップシステムを既に持っているが、「ベッドの下に隠れている」
これは実装詳細で、プログラマは影響を与える術をわずかしかもたない。
この提案を簡単にまとめると:

- We should add a core rule to the ownership system, called
  the Law of Exclusivity, which requires the implementation
  to prevent variables from being simultaneously accessed
  in conflicting ways.  (For example, being passed `inout`
  to two different functions.)  This will not be an opt-in
  change, but we believe that most programs will not be
  adversely affected.

  「排他則」(Law of Exclusivity) というオーナーシップシステムの核となる規則を追加し、
  変数への複数の方法(例えば、2つの異なる関数に `inout` を渡す)
  による衝突する同時アクセスを防ぐための実装を与えたい。
  この変更はオプトインにはしないだろうが、
  ほとんどのプログラマに悪影響は無いと信じる。
  
- We should add features to give programmers more control over
  the ownership system, chiefly by allowing the propagation
  of "shared" values.  This will be an opt-in change; it
  will largely consist of annotations and language features
  which programmers can simply not use.

  プログラマにオーナーシップシステムをより制御するための機能を与えたい。
  これは主として「shared」な値を伝搬させる機能だ。
  この変更はオプトインだろう。
  これはアノテーションと言語機能からなる大きなもので、
  プログラマは単に使用しない事ができる。

- We should add features to allow programmers to express
  types with unique ownership, which is to say, types that
  cannot be implicitly copied.  This will be an opt-in
  feature intended for experienced programmers who desire
  this level of control; we do not intend for ordinary
  Swift programming to require working with such types.

  プログラマにユニークオーナーシップな型を記述する機能を与えたい。
  つまり、暗黙にコピーできない型という事だ。
  これはこのレベルの制御を求める経験豊富なプログラマを想定したオプトインの機能だろう。
  普通のSwiftプログラミングではこうした型を取り扱う必要は無いと想定している。

These three tentpoles together have the effect of raising
the ownership system from an implementation detail to a more
visible aspect of the language.  They are also somewhat
inseparable, for reasons we'll explain, although of course they
can be prioritized differently.  For these reasons, we will
talk about them as a cohesive feature called "ownership".

これらの3つの柱が一緒になって
オーナーシップシステムを実装詳細から、
言語として見える姿に昇格する。
これらは互いに分離することができないが、
後に説明するとおり、もちろん優先順位は異なる。
こうした理由により、
これらの密接した機能をまとめて
「オーナーシップ」と呼ぶ。

### A bit more detail
### 詳細をもう少し

The basic problem with Swift's current ownership system
is copies, and all three tentpoles of ownership are about
avoiding copies.

Swiftの現在のオーナーシップシステムの基本的な問題はコピーで、
オーナーシップの3つの柱はコピー回避に関するものだ。

A value may be used in many different places in a program.
The implementation has to ensure that some copy of the value
survives and is usable at each of these places.  As long as
a type is copyable, it's always possible to satisfy that by
making more copies of the value.  However, most uses don't
actually require ownership of their own copy.  Some do: a
variable that didn't own its current value would only be
able to store values that were known to be owned by something
else, which isn't very useful in general.  But a simple thing
like reading a value out of a class instance only requires that
the instance still be valid, not that the code doing the
read actually own a reference to it itself.  Sometimes the
difference is obvious, but often it's impossible to know.
For example, the compiler generally doesn't know how an
arbitrary function will use its arguments; it just falls
back on a default rule for whether to pass ownership of
the value.  When that default rule is wrong, the program
will end up making extra copies at runtime.  So one simple
thing we can do is to allow programs to be more explicit at
certain points about whether they need ownership or not.

値はプログラムにおいて多くの異なる場所で使われるだろう。
実装はある値のコピーがそうした場所それぞれで生存し使用可能であることを保証する必要がある。
型がコピー可能である以上は、
さらなる値のコピー作ることで、
それを満たすことはいつでも可能だ。
しかし、殆どの使用において
専用のコピーの所有権は
実際には必要としない。
やられる事として、
特に使いみちはないが、
自身は現在の値をもっていないある変数が、
他の誰かに保持されている事がわかっている値を保持できてほしいだけ、
ということがある。
シンプルにクラスインスタンスを読み出すような事だが、
インスタンスがいまだ有効である事を求めているだけで、
コードは実際にはそれが保持する参照自体を読むことはない。
違いが明白なときもあるが、それを知ることができないこともある。
例えば、
コンパイラは一般に任意の関数がその引数をどのように使うかは知らない。
だから値のオーナーシップを渡すかどうかはデフォルトの規則にフォールバックするだけだ。
デフォルトのルールが間違っている時は、
プログラムは実行時に余分なコピーを作ることになる。
そこでできる一つのシンプルな事は、
プログラムがいくらかの点において
オーナーシップが必要なのかどうかを
もっと明示的に指定できるようにすることだ。

That closely dovetails with the desire to support non-copyable
types.  Most resources require a unique point of destruction:
a memory allocation can only be freed once, a file can only
be closed once, a lock can only be released once, and so on.
An owning reference to such a resource is therefore naturally
unique and thus non-copyable.  Of course, we can artificially
allow ownership to be shared by, say, adding a reference count
and only destroying the resource when the count reaches zero.
But this can substantially increase the overhead of working
with the resource; and worse, it introduces problems with
concurrency and re-entrancy.  If ownership is unique, and the
language can enforce that certain operations on a resource
can only be performed by the code that owns the resource,
then by construction only one piece of code can perform those
operations at a time.  As soon as ownership is shareable,
that property disappears.  So it is interesting for the
language to directly support non-copyable types because they
allow the expression of optimally-efficient abstractions
over resources.  However, working with such types requires
all of the abstraction points like function arguments to
be correctly annotated about whether they transfer ownership,
because the compiler can no longer just make things work
behind the scenes by adding copies.

それはコピー不能な型をサポートする欲求と密接に一致している。
ほとんどのリソースは破棄する単一の箇所を要求する。
確保したメモリはただ一度だけしか解放できないし、
ファイルは1度しか閉じれない、
ロックは1度しか外せない、など。
そうしたリソースを所有する参照は、
それゆえに自然にユニークだしコピー不可だ。
もちろん、人工的にオーナーシップを共有することはできる、
例えば、参照カウントを追加して、
カウントが0になったときだけリソースを解放することで。
しかしこれはリソースを取り扱うオーバーヘッドを大幅に増加させる。
さらに悪い事に、これは並行性と再入性についての問題を導入する。
もしオーナーシップがユニークで、
リソースへのいくらかの命令を
リソースを保有しているコードからしか行えないことを
言語が強制できるなら、
一つのコード片を作ることによってそうした命令を同時に行う事ができるだろう。
オーナーシップが共有できるならば、その価値はすぐに無くなる。
言語がコピー不可能な型を直接サポートすることは興味深いのは、
なぜならリソースの最適効率な抽象化が表現できるようになるからだ。
しかし、そうした型を取り扱うには
関数の引数のような全ての抽象点が、
そこでオーナーシップを転送するかどうかを正しく指定できる事が必要だ。
なぜならコンパイラはコピーを追加するという
水面下の仕事がもはやできないからだ。

Solving either of these problems well will require us to
also solve the problem of non-exclusive access to variables.
Swift today allows nested accesses to the same variable;
for example, a single variable can be passed as two different
`inout` arguments, or a method can be passed a callback that
somehow accesses the same variable that the method was called on.
Doing this is mostly discouraged, but it's not forbidden,
and both the compiler and the standard library have to bend
over backwards to ensure that the program won't misbehave
too badly if it happens.  For example, `Array` has to retain
its buffer during an in-place element modification; otherwise,
if that modification somehow reassigned the array variable,
the buffer would be freed while the element was still being
changed.  Similarly, the compiler generally finds it difficult
to prove that values in memory are the same at different points
in a function, because it has to assume that any opaque
function call might rewrite all memory; as a result, it
often has to insert copies or preserve redundant loads
out of paranoia.  Worse, non-exclusive access greatly
limits the usefulness of explicit annotations.  For example,
a "shared" argument is only useful if it's really guaranteed
to stay valid for the entire call, but the only way to
reliably satisfy that for the current value of a variable
that can be re-entrantly modified is to make a copy and pass
that instead.  It also makes certain important patterns
impossible, like stealing the current value of a variable
in order to build something new; this is unsound if the
variable can be accessed by other code in the middle.
The only solution to this is to establish a rule that prevents
multiple contexts from accessing the same variable at the
same time.  This is what we propose to do with the Law
of Exclusivity.

これらの問題のいずれかをうまく解決するには
変数への排他的でないアクセスの問題もまた解決する必要がある。
今のSwiftは同じ変数へのネストしたアクセスを許している。
例えば、一つの変数を2つの異なる `inout` 引数に渡すことができたり、
メソッドに
そのメソッドが呼ばれているのと同じ変数になんからの方法でアクセスする
コールバックを渡すことができる。
そんな事をするのはたいていやめたほうがよいが、
しかし禁止はされていないから、
コンパイラと標準ライブラリは
もしそれがされていたとしても、
プログラムが最悪にも不正な動作をしないことを保証するために、
後ろ向きな努力をする必要がある。
例えば、 `Array` はインプレースな要素変更の最中に自身のバッファをretainせねばならない。
そうしないと、その変更処理がどうにかして配列変数に再代入したとき、
要素がまだ変更されている最中にもかかわらずバッファが解放されてしまうかもしれないからだ。
同様に、
コンパイラにとって
関数の異なるところにあるメモリ上の値が同一である事を証明するのは
一般に難しい。
なぜならあらゆるオペークな関数呼び出しが
メモリ全てを書き換えうることを仮定する必要があるからだ。
結果として、
パラノイアから抜け出し、コピーか冗長な読み出し結果の保持を
しばしば挿入する必要がある。
悪いことに、排他的でないアクセスは
明示的な指定の有用性を著しく制限する。
例えば、「shared」引数は
呼び出し全体の間有効であることを本当に保証できるときだけ有用だ。
しかし
再入して変更されうる変数の現在の値が
それを確実に満たす唯一の方法は
コピーを作って代わりにそれを渡すことだけだ。
これは
何かを新しく組み立てるために変数の現在値を盗むような
いくつかの重要なパターンをも不可能にする。
これは
もし変数が途中で他のコードからアクセスしうる場合に
不安定だ。
これの唯一の解決法は
同時に同じ変数に複数のコンテキストからアクセスする事を防ぐための
規則を確立する事だ。
これを「排他則」として提案する。

All three of these goals are closely linked and mutually
reinforcing.  The Law of Exclusivity allows explicit annotations
to actually optimize code by default and enables mandatory
idioms for non-copyable types.  Explicit annotations create
more optimization opportunities under the Law and enable
non-copyable types to function.  Non-copyable types validate
that annotations are optimal even for copyable types and
create more situations where the Law can be satisfied statically.

これら3つの目標は密接につながっており相互に補強し合う。
排他則はデフォルトで実際にコードを最適化するための明示的指定を可能にし、
コピー不可能な型についての必須のイディオムを使えるようにする。
明示的指定は法則化でのより多くの最適化の機会を作り出し、
コピー不可能な型の機能を有効化する。
コピー不可能な型は
コピー可能な型として考えても指定が最適であるか検証し、
法則が静的に満たされるより多くの場面を作り出す。

### Criteria for success
### 成功の基準

As discussed above, it is the core team's expectation that
ownership can be delivered as an opt-in enhancement to Swift.
Programmers should be able to largely ignore ownership and not
suffer for it.  If this expectation proves to not be satisfiable,
we will reject ownership rather than imposing substantial
burdens on regular programs.

既に述べたように、
コアチームの期待は
オーナーシップがSwiftのオプトインの改良として供給できる事だ。
プログラマはオーナーシップを大きく無視して、
それに苦しまないようにできるのが良いだろう。
もしこの期待が満たせない事が判明した場合、
無視できない大量の負担を普通のプログラムに負わせるよりは、
オーナーシップを却下するだろう。

The Law of Exclusivity will impose some new static and dynamic
restrictions.  It is our belief that these restrictions will only
affect a small amount of code, and only code that does things
that we already document as producing unspecified results.
These restrictions, when enforced dynamically, will also hurt
performance.  It is our hope that this will be "paid for" by
the improved optimization potential.  We will also provide tools
for programmers to eliminate these safety checks where necessary.
We will discuss these restrictions in greater detail later in this
document.

排他則はいくらかの新しい静的な制限と動的な制限を課すだろう。
そうした制限は
少量のコードと、
すでに文書化した未定義の結果を引き起こすようなコード
だけに影響を与えると信じている。
これらの制限が動的に強制されるとき、パフォーマンスの負担もまたある。
これは最適化のポテンシャルが改善する事で「支払われる」事を望んでいる。
また、プログラマが必要ならばそうした安全性検査を除去するツールを提供したい。
これらの制限についてのより詳細な議論はこの文書で後述する。

## Core definitions
## 核となる定義

### Values
### 値

Any discussion of ownership systems is bound to be at a lower
level of abstraction.  We will be talking a lot about
implementation topics.  In this context, when we say "value",
we mean a specific instance of a semantic, user-language value.

オーナーシップシステムのいかなる議論も抽象度の低いレベルに縛られる。
これから多くの実装上のトピックについて述べる。
この文脈において、「値」は、ユーザー言語の値の意味で特定の実体を指す。

For example, consider the following Swift code:
例として、次のSwiftコードを考える。

```
var x = [1,2,3]
var y = x
```

People would often say that `x` and `y` have the same value
at this point.  Let's call this a *semantic value*.  But at
the level of the implementation, because the variables `x` and `y`
can be independently modified, the value in `y` must be a
copy of the value in `x`.  Let's call this a *value instance*.
A value instance can be moved around in memory and remain the
same value instance, but a copy always yields a new value instance.
For the remainder of this document, when we use "value" without any
qualification, we mean it in this low-level sense of value instance.

この時点で、しばしば `x` と `y` は同じ値を持っていると言うだろう。
これを「semantic value」と呼ぶ。
しかし実装のレベルでは、変数 `x` と `y` は独立に変更できるので、
`y` の値は `x` の値のコピーである。
これを「value instance」と呼ぶ。
value instanceはメモリの中を動き回り同じvalue instanceのままである、
ただしコピーはいつも新しいvalue instanceを生み出す。
以降この文書では、ただ「値」と使う際には、
このvalue instanceのローレベルの趣旨を指す。

What it means to copy or destroy a value instance depends on the type:

value instanceをコピーまたは破棄するといった際の意味は型による。

* Some types do not require extra work besides copying their
  byte-representation; we call these *trivial*.  For example,
  `Int` and `Float` are trivial types, as are ordinary `struct`s
  and `enum`s containing only such values.  Most of what we have
  to say about ownership in this document doesn't apply to the
  values of such types.  However, the Law of Exclusivity will still
  apply to them.

  いくつかの型はそれのバイト表現をコピーする以外に余分な作業を必要としない。
  これを「trivial」と呼ぶ。
  例えば、 `Int` と `Float` は trivial な型で、
  そうした値しか含まない普通の `struct` と `enum` もそうだ。
  この文書でオーナーシップに関して述べたことのほとんどは、
  これらの型の値には適用されない。
  それでもなお排他則はこれらに適用される。

* For reference types, the value instance is a reference to an object.
  Copying the value instance means making a new reference, which
  increases the reference count.  Destroying the value instance means
  destroying a reference, which decreases the reference count.  Decreasing
  the reference count can, of course, drop it to zero and thus destroy
  the object, but it's important to remember that all this talk about
  copying and destroying values means manipulating reference counts,
  not copying the object or (necessarily) destroying it.

  参照型においては、value instanceはオブジェクトへの参照だ。
  value instanceのコピーは新しい参照の生成を意味し、
  参照カウントを増加させる。
  value instanceの破棄は参照の破棄を意味し、
  参照カウントを減少させる。
  参照カウントの減少は、もちろん、それが0になった時にはオブジェクトの破棄をするが、
  ここで覚えておくべき大事な事は、
  このコピーと破棄に関する話の全てが参照カウントの操作を意味する事で、
  オブジェクトのコピーではないし(必要ならば)それを破棄する事だ。

* For copy-on-write types, the value instance includes a reference to
  a buffer, which then works basically like a reference type.  Again,
  it is important to remember that copying the value doesn't mean
  copying the contents of the buffer into a new buffer.

  COW型については、value instanceはバッファへの参照を含み、
  これは基本的には参照型のように働く。
  ここでもやはり覚えておくべき大事なことは、
  値のコピーはバッファの中身を新しいバッファにコピーする事を意味しない事だ。

There are similar rules for every kind of type.

全ての種類の型について、このような規則がある。

### Memory
### メモリ

In general, a value can be owned in one of two ways: it can be
"in flight", a temporary value owned by a specific execution context
which computed the value as an operand, or it can be "at rest",
stored in some sort of memory.

一般に、値は2つのうちどちらかの方法で保有できる。
一つは「in flight」で、テンポラリな値が特定の実行コンテキストによって保有され、
オペランドとして値が計算される。
もう一つは「at rest」で、いくらかのメモリの中に保存される。

We don't need to focus much on temporary values because their
ownership rules are straightforward.  Temporary values are created
as the result of some expression; that expression is used in some
specific place; the value is needed in that place and not
thereafter; so the implementation should clearly take all possible
steps to forward the value directly to that place instead of
forcing it to be copied.  Users already expect all of this to
happen, and there's nothing really to improve here.

テンポラリな値には多くを着目する必要はない。
これらのオーナーシップ則は素直だからだ。
テンポラリな値はなんらかの式の結果として作られる。
その式はなんらかの特定の箇所で使用される。
その値はその場所で必要で、その後では必要ない。
だから実装は
値を強制的にコピーする代わりに
その場所へ値を直接送るための全てのありえるステップを
明らかに取るべきだ。
ユーザは全てのそれの発生を予期していて、
ここに改善するものは本当に何もない。

Therefore, most of our discussion of ownership will center around
values stored in memory.  There are five closely related concepts
in Swift's treatment of memory.

よって、オーナーシップについての議論のほぼ全ては、
メモリ中に保持される値に集中する。
ここにSwiftのメモリの取扱に密接に関係する5つの概念がある。

A *storage declaration* is the language-syntax concept of a declaration
that can be treated in the language like memory.  Currently, these are
always introduced with `let`, `var`, and `subscript`.  A storage
declaration has a type.  It also has an implementation which defines
what it means to read or write the storage.  The default implementation
of a `var` or `let` just creates a new variable to store the value,
but storage declarations can also be computed, and so there needn't
be any variables at all behind one.

「storage declaration」はメモリのように言語中で扱われる宣言の言語構文上の概念だ。
現在、これらはいつも `let`, `var`, `subscript` によって導入される。
storage declarationは型を持つ。
これはそのストレージから何が読み書きできるかを定義する実装を持つ。
`var` と `let` のデフォルト実装は値を保存するための新しい変数をただ作るが、
storage declarationをcomputedにすることもでき、
その場合はそれの背後には全く何の変数も必要ない。

A *storage reference expression* is the syntax concept of an expression
that refers to storage.  This is similar to the concept from other
languages of an "l-value", except that it isn't necessarily usable on
the left side of an assignment because the storage doesn't have to be
mutable.

「storage reference expression」はストレージを参照する式の構文上の概念だ。
これは「l-value」という他の言語の概念に似ているが、
代入文の左側で使用可能である必要は無く、
なぜならストレージはmutableでなくても良いからだ。

A *storage reference* is the language-semantics concept of a fully
filled-in reference to a specific storage declaration.  In other
words, it is the result of evaluating a storage reference expression
in the abstract, without actually accessing the storage.  If the
storage is a member, this includes a value or storage reference
for the base.  If the storage is a subscript, this includes a value
for the index.  For example, a storage reference expression like
`widgets[i].weight` might abstractly evaluate to this storage reference:

「storage reference」は特定のstorage declarationを完全に代理する参照の言語意味論上の概念だ。
言い換えれば、理論上は storage reference expression を評価した結果で、
ストレージへの実際のアクセスは無い。
もしストレージがメンバならば、
それは値もしくはその基点へのstorage referenceを含む。
もしストレージがsubscriptならば、
それはインデックスの値を含む。
例えば、`widgets[i].weight` という storage reference expression は
次のstorage referenceとして抽象的に評価されうるだろう。

* the storage for the property `var weight: Double` of

  `var weight: Double` プロパティのストレージ、of

* the storage for the subscript `subscript(index: Int)` at index value `19: Int` of

  subscript `subscript(index: Int)` のインデックス `19: Int` のストレージ、of

* the storage for the local variable `var widgets: [Widget]`

  ローカル変数 `var widgets: [Widget]` のストレージ

A *variable* is the semantics concept of a unique place in
memory that stores a value.  It's not necessarily mutable, at least
as we're using it in this document.  Variables are usually created for
storage declarations, but they can also be created dynamically in
raw memory, e.g. using `UnsafeRawPointer`.  A variable always has a
specific type.  It also has a *lifetime*, i.e. a point in the language
semantics where it comes into existence and a point (or several)
where it is destroyed.

「variable」は値を保持するメモリ上のユニークな場所の意味論的な概念だ。
この文書でそのように扱うように、少なくともこれは可変でなくともよい。
variable は通常は storage declarationとして生成されるが、
生のメモリ上に動的に生成することもまたできる、例えば `UnsafeRawPointer` を使って。
variable はいつも特定の型をもつ。
これはまた「lifetime」を持つ、すなわち、
言語の意味論的に、
存在する状態になる一つの点と、
破棄される一つの(または数個の)点がある。

A *memory location* is a contiguous range of addressable memory.  In
Swift, this is mostly an implementation concept.  Swift does not
guarantee that any particular variable will have a consistent memory
location throughout its lifetime, or in fact be stored in a memory
location at all.  But a variable can sometimes be temporarily forced
to exist at a specific, consistent location: e.g. it can be passed
`inout` to `withUnsafeMutablePointer`.

「memory location」はアドレス可能なメモリの連続した範囲だ。
Swiftでは、これはほとんど実装上の概念だ。
Swiftはどんな特定の variable もその lifetime の間ずっと一貫したメモリ位置を持つことや、
それどころかメモリの中に保存される事すら全く保証しない。
しかし variable は時々特定の一貫した場所に存在することを一時的に強制される。
たとえばそれが `withUnsafeMutablePointer` に `inout` で渡されるときだ。

### Accesses
### アクセス

A particular evaluation of a storage reference expression is
called an access.  Accesses come in three kinds: *reads*,
*assignments*, and *modifications*.  Assignments and modifications
are both *writes*, with the difference being that an assignment
completely replaces the old value without reading it, while a
modification does rely on the old value.

特定の storage reference expression の評価はアクセスと呼ばれる。
アクセスは3種類がある。
「reads」「assignments」「modifications」。
assignments と modifications はどちらも「writes」で、
その違いは assignment は それの古い値を読むことなしに完全に置き換えるが、
modification は古い値に依存する。

All storage reference expressions are classified into one of these
three kinds of access based on the context in which the expression
appears.  It is important to note that this decision is superficial:
it relies only on the semantic rules of the immediate context, not
on a deeper analysis of the program or its dynamic behavior.
For example, a storage reference passed as an `inout` argument
is always evaluated as a modification in the caller, regardless
of whether the callee actually uses the current value, performs
any writes to it, or even refers to it at all.

全ての storage reference expression はその式が現れたコンテキストに応じて
これらの3種類のアクセスのいずれかに分類される。
この決定は表面的であることに注意することが大事だ。
即時のコンテキストの意味論的な規則によってのみ決まり、
プログラムの深い解析やそれの動的な振る舞いには関係がない。
例えば、 `inout` 引数として渡される storage reference は
常にその caller において modification として評価され、
callee が実際に現在値を使うかどうか、
どんな書き込みが行われるか、
それに参照しているかどうかさえ、
全く関係がない。

The evaluation of a storage reference expression is divided into
two phases: it is first formally evaluated to a storage reference,
and then a formal access to that storage reference occurs for some
duration.  The two phases are often evaluated in immediate
succession, but they can be separated in complex cases, such as
when an `inout` argument is not the last argument to a call.
The purpose of this phase division is to minimize the duration of
the formal access while still preserving, to the greatest extent
possible, Swift's left-to-right evaluation rules.

storage reference expression の評価は2つのフェーズに分けられる。
はじめに storage reference へと形式的に評価され、
その後形式的なアクセスがある期間の間そのストレージへと発生する。
この2つめのフェーズは
`inout` 引数が呼び出しの最終引数では無い場合などでは、
複雑な場合に分割することができる、
即時の連なりの中で評価される。
このフェーズ分割の目的は
Swiftの左から右への評価の規則で、ありうる最大の範囲で
それが維持されている間の形式的アクセスの期間を最小化することだ。

## The Law of Exclusivity
## 排他則

With all of that established, we can succinctly state the first
part of this proposal, the Law of Exclusivity:

ここまでに確率した全てを用いると、
この提案の最初の部分で簡潔に述べた排他則とは

> If a storage reference expression evaluates to a storage
> reference that is implemented by a variable, then the formal
> access duration of that access may not overlap the formal
> access duration of any other access to the same variable
> unless both accesses are reads.

> もし storage reference expression が variable によって実装される
> storage reference への評価されるなら、
> それへのアクセスの形式的な期間は、
> 同じ variable への他のあらゆるアクセスの期間と重ならないかもしれない、
> それが両方共 reads でない限り。

This is intentionally vague: it merely says that accesses
"may not" overlap, without specifying how that will be
enforced.  This is because we will use different enforcement
mechanisms for different kinds of storage.  We will discuss
those mechanisms in the next major section.  First, however,
we need to talk in general about some of the implications of
this rule and our approach to satisfying it.

これは意図的に曖昧である。
これは単に
どのように強制するか指定することなしには
アクセスが重なら「ないかもしれない」、
と言っている。
これはストレージの種類に応じで異なる強制機構を使うかもしれないだからだ。
次の章でそれらの機構について議論したい。
しかしまずはじめに、
このルールの幾つかの意味あいと
それを満たすための手法について
一般に述べる。

### Duration of exclusivity
### 排他の期間

The Law says that accesses must be exclusive for their entire
formal access duration.  This duration is determined by the
immediate context which causes the access; that is, it's a
*static* property of the program, whereas the safety problems
we laid out in the introduction are *dynamic*.  It is a general
truth that static approaches to dynamic problems can only be
conservatively correct: there will be dynamically-reasonable
programs that are nonetheless rejected.  It is fair to ask how
that general principle applies here.

法則はアクセス間の形式的なアクセス期間全体が排他でなければならないとしている。
この期間はアクセスが起きる即時コンテキストによって決定される。
すなわち、これはプログラムの「静的な」特性だ。
一方で、冒頭で提示した安全性の問題は「動的」だ。
動的な問題に対する静的なアプローチは保守的にしか正しくできないというのが一般的な真実だ。
これは動的には妥当なプログラムにもかかわらず却下しうることになる。
その一般原理をどのようにここに適用するかを問う事は正しい。

For example, when storage is passed as an `inout` argument, the
access lasts for the duration of the call.  This demands
caller-side enforcement that no other accesses can occur to
that storage during the call. Is it possible that this is too
coarse-grained?  After all, there may be many points within the
called function where it isn't obviously using its `inout`
argument.  Perhaps we should track accesses to `inout` arguments
at a finer-grained level, within the callee, instead of attempting
to enforce the Law of Exclusivity in the caller.  The problem
is that that idea is simply too dynamic to be efficiently
implemented.

例えば、ストレージが `inout` 引数として渡されている時、
アクセスは呼び出しの間続く。
これは呼び出しの間にストレージへの他のアクセスが存在できないようにする
caller側での強制を必要とする。
これは粒度が荒すぎるだろうか？
どのみち、呼ばれる関数の中にはその `inout` 引数を
明らかに使用しない箇所があるかもしれない。
もしかするとcalleeの中において、
`inout` 引数へのアクセスをきめ細かい粒度で追跡すべきかもしれない、
caller側で排他則を強制する事を試みる代わりに。
その場合の問題は、
このアイデアは単純に複雑すぎて効率的に実装できないということだ。

A caller-side rule for `inout` has one key advantage: the
caller has an enormous amount of information about what
storage is being passed.  This means that a caller-side rule
can often be enforced purely statically, without adding dynamic
checks or making paranoid assumptions.  For example, suppose
that a function calls a `mutating` method on a local variable.
(Recall that `mutating` methods are passed `self` as an `inout`
argument.)  Unless the variable has been captured in an
escaping closure, the function can easily examine every
access to the variable to see that none of them overlap
the call, thus proving that the rule is satisfied.  Moreover,
that guarantee is then passed down to the callee, which can
use that information to prove the safety of its own accesses.

`inout` についてのcaller側での規則は一つの重要な優位性がある。
callerは渡そうとしているストレージについての膨大な量の情報を持っていることだ。
これはcaller側での規則がしばしば純粋に静的に強制でき、
動的な検査の追加や偏執的な仮定を作る必要が無いという事を意味する。
例えば、ローカル変数の `mutating` メソッドを呼び出す関数を想像せよ。
(`mutating` メソッドは `self` を `inout` 引数として渡す事を思い出せ。)
変数がエスケープするクロージャにキャプチャされないかぎり、
その関数は変数への全てのアクセスが
いずれもお互いに重なって呼び出していないか簡単に調べられる。
加えて、この保証はcalleeに渡すことができ、
calleeは自身のアクセスの安全性を証明するのにその情報を使うことができる。

In contrast, a callee-side rule for `inout` cannot take
advantage of that kind of information: the information is
simply discarded at the point of the call.  This leads to the
widespread optimization problems that we see today, as
discussed in the introduction.  For example, suppose that
the callee loads a value from its argument, then calls
a function which the optimizer cannot reason about:

対して、`inout` についてのcallee側の規則はそのような種類の情報を活用することができない。
情報は呼び出しの時点でただ捨てられている。
これは導入で議論したようなこんにち見られる幅広い最適化上の問題を引き起こす。
例えば、calleeがその引数から値を読む事を想像せよ、
そしてオプティマイザがそれについてわからない関数を呼ぶとする。

```
extension Array {
  mutating func organize(_ predicate: (Element) -> Bool) {
    let first = self[0]
    if !predicate(first) { return }
    ...
    // something here uses first
    // ここに first を使用する何か
  }
}
```

Under a callee-side rule, the optimizer must copy `self[0]`
into `first` because it must assume (paranoidly) that
`predicate` might somehow turn around and modify the
variable that `self` was bound to.  Under a caller-side
rule, the optimizer can use the copy of value held in the
array element for as long as it can continue to prove that
the array hasn't been modified.

callee側での規則では、
オプティマイザは `self[0]` を `first` にコピーしなければならない、
なぜなら `predicate` がどうにかして振り向いて
`self` が紐付けられている変数を変更することを
仮定しなければならないからだ。
caller側での規則では、
配列が変更されない事がわかりつづける間、
オプティマイザは配列の要素の中に保持される値のコピーを使うことができる。

Moreover, as the example above suggests, what sort of code
would we actually be enabling by embracing a callee-side rule?
A higher-order operation like this should not have to worry
about the caller passing in a predicate that re-entrantly
modifies the array.  Simple implementation choices, like
making the local variable `first` instead of re-accessing
`self[0]` in the example above, would become semantically
important; maintaining any sort of invariant would be almost
inconceivable.  It is no surprise that Swift's libraries
generally forbid this kind of re-entrant access.  But,
since the library can't completely prevent programmers
from doing it, the implementation must nonetheless do extra
work at runtime to prevent such code from running into
undefined behavior and corrupting the process.  Because it
exists solely to work around the possibility of code that
should never occur in a well-written program, we see this
as no real loss.

上記の例で示唆されたように、
さらにどのような種類のコードを caller 側の規則で包むことを実際に有効化するのが
良いだろうか？
このような高階関数は caller が配列を再入して変更するようなpredicateを
渡す事を心配する必要はない。
単純な実装上の選択として、
上記の例のように `self[0]` に再アクセスする代わりにローカル変数 `first` を作ることは、
重要な意味を持つだろう。
Swiftのライブラリがこのような再入アクセスを一般に禁止するのはありえないことではない。
しかし、ライブラリはプログラマにそれをすることを完全に防ぐことはできず、
実装は実行時の余分な処理が生じるにもかかわらず、
未定義の動作やプロセスの崩壊へと進むことをそのようなコードから防がねばならない。
これはたんによく書かれるプログラムでは絶対に現れないコードの可能性に対処するためだけに存在するので、
実際の損失としては見えない。

Therefore, this proposal generally proposes access-duration
rules like caller-side `inout` enforcement, which allow
substantial optimization opportunities at little semantic
cost.

だから、この提案では
一般に caller側の `inout` 強制のようなアクセス期間規則を提案し、
かなりの最適化の目的のために少しの意味論上のコストを許す。

### Components of value and reference types
### 値型と参照型の構成要素

We've been talking about *variables* a lot.  A reader might
reasonably wonder what all this means for *properties*.

「variable」について多くを述べてきた。
「property」についてはどうなるのか疑問に思うのは道理だ。

Under the definition we laid out above, a property is a
storage declaration, and a stored property creates a
corresponding variable in its container.  Accesses to that
variable obviously need to obey the Law of Exclusivity, but are
there any additional restrictions in play due to the fact
that the properties are organized together into a container?
In particular, should the Law of Exclusivity prevent accesses
to different properties of the same variable or value from
overlapping?

ここまでに示した定義のもとでは、
property は storage declaration だ、
そして stored property はそのコンテナの中に対応する variable を作る。
その variable へのアクセスは明らかに排他則に従うべきだろう、
しかし、 property はコンテナの中にまとめて入っている事実によって、
そこに何か新しい制限が生じるだろうか？
特に、排他則は同じ variable か重なった値の
異なるプロパティへのアクセスを防ぐべきだろうか？

Properties can be classified into three groups:

プロパティは3種類に分けられる

- instance properties of value types,

  値型のインスタンスプロパティ

- instance properties of reference types, and

  参照型のインスタンスプロパティ、そして、

- `static` and `class` properties on any kind of type.

  あらゆる種類の型の `static` と `class` プロパティ

We propose to always treat reference-type and `static` properties
as independent from one another other, but to treat value-type
properties as generally non-independent outside of a specific
(but important) special case.  That's a potentially significant
restriction, and it's reasonable to wonder both why it's necessary
and why we need to draw this distinction between different
kinds of property.  There are three reasons.

参照型と `static` プロパティについてはお互いに独立していると常に取り扱うよう提案する。
しかし 値型のプロパティは 特定の(しかし重要な)特殊な場合を除いて一般に独立していないとする。
これは潜在的にかなりの制限だ、
なぜそれが必要なのか疑問に思うのは合理的で、
ことなるプロパティの種類の間のこの違いについて説明する必要がある。
ここには3つの理由がある。

#### Independence and containers
#### 独立性とコンテナ

The first relates to the container.

1つ目はコンテナと関係する。

For value types, it is possible to access both an individual
property and the entire aggregate value.  It is clear that an
access to a property can conflict with an access to the aggregate,
because an access to the aggregate is essentially an access to
all of the properties at once.  For example, consider a variable
(not necessarily a local one) `p: Point` with stored properties
`x`, `y`, and `z`.  If it were possible to simultaneously and
independently modify `p` and `p.x`, that would be an enormous
hole in the Law of Exclusivity.  So we do need to enforce the
Law somehow here.  We have three options.

値型では、
個別のプロパティと集まった値全体の両方にアクセスすることができる。
あるプロパティへのアクセスが全体へのアクセスと衝突することは明らかで、
だから全体へのアクセスは本質的に全てのプロパティに同時にアクセスする事だ。
例えば、ある変数(ローカル変数でなくてもよい) `p: Point` について考える。
これは stored property `x`, `y`, `z` を持つとする。
もし `p` と `p.x` を同時に独立に変更することができたら、
これは排他則に巨大な穴をあけることになる。
だから法則をどうにかここに強制する必要がある。
3つの選択肢がある。

(This may make more sense after reading the main section
about enforcing the Law.)

(これは法則の強制についてのメインの章を読んだあとの方がもっと理解できるかもしれない)

The first option is to simply treat `p.x` as also an access to `p`.
This neatly eliminates the hole because whatever enforcement
we're using for `p` will naturally prevent conflicting accesses
to it.  But this will also prevent accesses to different
properties from overlapping, because each will cause an access
to `p`, triggering the enforcement.

第一の選択肢は `p.x` を単純に `p` へのアクセスとみなすことだ。
これは穴をきちんと消去する、なぜなら `p` に対するどんな強制でも
それへの衝突するアクセスを自然と防ぐからだ。
しかしこれは重なった異なるプロパティへのアクセスも防いでしまう、
なぜならそれぞれが `p` へのアクセスを引き起こし、
強制を引き起こすからだ。

The other two options involve reversing that relationship.
We could split enforcement out for all the individual stored
properties, not for the aggregate: an access to `p` would be
treated as an access to `p.x`, `p.y`, and `p.z`.  Or we could
parameterize enforcement and teach it to record the specific
path of properties being accessed: "", ".x", and so on.
Unfortunately, there are two problems with these schemes.
The first is that we don't always know the full set of
properties, or which properties are stored; the implementation
of a type might be opaque to us due to e.g. generics or
resilience.  An access to a computed property must be treated
as an access to the whole value because it involves passing
the variable to a getter or setter either `inout` or `shared`;
thus it does actually conflict with all other properties.
Attempting to make things work despite that by using dynamic
information would introduce ubiquitous bookkeeping into
value-type accessors, endangering the core design goal of
value types that they serve as a low-cost abstraction tool.
The second is that, while these schemes can be applied to
static enforcement relatively easily, applying them to
dynamic enforcement would require a fiendish amount of
bookkeeping to be carried out dynamically; this is simply
not compatible with our performance goals.

他の2つの選択肢は関係性を逆転させる。
強制を全ての個別の stored property に分割して、集めない。
`p` へのアクセスは `p.x`, `p.y`, `p.z` へのアクセスとみなす。
もしくは、強制をパラメータ化して、
アクセスしているプロパティへのパスを記録して教える。
"", ".x" など。
残念ながら、これらの構想には2つの問題がある。
第一にはプロパティの全てのセットや、どれが stored なのかがいつもわかるわけではないことだ。
型の実装は不可視かもしれない、例えば ジェネリクスやレジリエンスのせいで。
computed propertyへのアクセスは値全体へのアクセスとして扱わねばならない、
なぜなら変数をゲッターかセッターに `inout` か `shared` として渡すからだ。
だから他の全てのプロパティと実際に衝突する。
動的な情報を使ってそれが動くようにすることを試みるなら、
値型のアクセサのあちこちで記録をつける仕組みとなるが、
値型を低コストな抽象度の道具として提供するという根本的な設計目標を危険にさらす。
第二にはこれらの構想を静的な強制を適用するのは相対的に簡単だが、
動的な強制を適用するのはひどい量の記録を動的にもたらす必要がある。
これは単純に性能目標に応えられない。

Thus, while there is a scheme which allows independent access
to different properties of the same aggregate value, it
requires us to be using static enforcement for the aggregate
access and to know that both properties are stored.  This is an
important special case, but it is just a special case.
In all other cases, we must fall back on the general rule
that an access to a property is also an access to the aggregate.

よって、同じ集まった値の異なるプロパティへの個別のアクセスをゆるす構想は存在するが、
集まったアクセスの静的強制とどのプロパティが stored かしる必要がある。
これは重要な特別な場合だが、ただの特別な場合だ。
それ以外のすべての場合では、
あるプロパティへのアクセスが集まったアクセスであるという
一般的なルールにフォールバックせねばならない。

These considerations do not apply to `static` properties and
properties of reference types.  There are no language constructs
in Swift which access every property of a class simultaneously,
and it doesn't even make sense to talk about "every" `static`
property of a type because an arbitrary module can add a new
one at any time.

これらの考察は `static` プロパティと参照型のプロパティには適用しない。
Swiftにはクラスの全てのプロパティに同時にアクセスする言語の機能はなく、
ある型の「全ての」 `static` プロパティについて話すことは実に意味がない、
なぜなら任意のモジュールがいつでも新しいものを追加できるからだ。

#### Idioms of independent access
#### 独立したアクセスのイディオム

The second relates to user expectations.

第二はユーザーの期待と関係している。

Preventing overlapping accesses to different properties of a
value type is at most a minor inconvenience.  The Law of Exclusivity
prevents "spooky action at a distance" with value types anyway:
for example, calling a method on a variable cannot kick off a
non-obvious sequence of events which eventually reach back and
modify the original variable, because that would involve two
conflicting and overlapping accesses to the same variable.

値型の異なるプロパティへの重なったアクセスを防止するのはよくてもわずかに不便だ。
排他則は「少し離れた不気味な動作」を値型についてとにかく防ぐ。
例えば、変数へのメソッド呼び出しは
最終的に戻ってもとの変数を変更する
明らかではないイベントの連続を開始できない、
なぜなら同じ変数への2つの衝突する重なったアクセスをもたらすからだ。

In contrast, many established patterns with reference types
depend on exactly that kind of notification-based update.  In
fact, it's not uncommon in UI code for different properties of
the same object to be modified concurrently: one by the UI and
the other by some background operation.  Preventing independent
access would break those idioms, which is not acceptable.

対して、参照型についての多くの確立したパターンは、
通知ベースの更新のようなものにまさに依存している。
実際、UIコードでは並行に同じオブジェクトの異なるプロパティが変更されるのは珍しくはない。
片方はUIで、もう片方は何かのバックグラウンド処理など。
独立したアクセスを防ぐことはこれらのイディオムを破壊するので受け入れられない。

As for `static` properties, programmers expect them to be
independent global variables; it would make no sense for
an access to one global to prevent access to another.

`static` プロパティに関しては、
プログラマはこれらを独立したグローバル変数と期待している。
ある一つグローバルへのアクセスがほかへのアクセスを防ぐのは意味が通らないだろう。

#### Independence and the optimizer
#### 独立性とオプティマイザ 

The third relates to the optimization potential of properties.

第三はプロパティの最適化の可能性に関係する。

Part of the purpose of the Law of Exclusivity is that it
allows a large class of optimizations on values.  For example,
a non-`mutating` method on a value type can assume that `self`
remains exactly the same for the duration of the method.  It
does not have to worry that an unknown function it calls
in the middle will somehow reach back and modify `self`,
because that modification would violate the Law.  Even in a
`mutating` method, no code can access `self` unless the
method knows about it.  Those assumptions are extremely
important for optimizing Swift code.

排他則の目的の一部は値の最適化の大きな規模で可能にすることだ。
例えば、値型のnon-`mutating` メソッドは `self` がメソッド呼び出しの間
全く同じままであることを仮定できる。
これはその中で呼ばれる未知の関数がどうにかして向き直して `self` を変更することについて
心配する必要を無くす。
なぜなら変更は法則に違反するからだ。
`mutating` メソッドでさえも、そのメソッドがそれについて知らない限り、
どんなコードも `self` にアクセスすることはできない。
これらの仮定は Swift コードを最適化する上でとても重要だ。

However, these assumptions simply cannot be done in general
for the contents of global variables or reference-type
properties.  Class references can be shared arbitrarily,
and the optimizer must assume that an unknown function
might have access to the same instance.  And any code in
the system can potentially access a global variable (ignoring
access control).  So the language implementation would
gain little to nothing from treating accesses to different
properties as non-independent.

しかし、これらの仮定はグローバル変数と参照型のプロパティの中身には一般にできない。
クラスの参照は任意に共有できるし、
オプティマイザは未知の関数が同じインスタンスへのアクセスをもちうることを仮定せねばならない。
そしてシステム内のどんなコードもグローバル変数(アクセス制御を無視する)にアクセスする可能性を持つ。
だから言語の実装は異なるプロパティへのアクセスが独立に扱われるところでは
得るものは少ししかないか何もない。

#### Subscripts
#### subscript

Much of this discussion also applies to subscripts, though
in the language today subscripts are never technically stored.
Accessing a component of a value type through a subscript
is treated as accessing the entire value, and so is considered
to overlap any other access to the value.  The most important
consequence of this is that two different array elements cannot
be simultaneously accessed.  This will interfere with certain
common idioms for working with arrays, although some cases
(like concurrently modifying different slices of an array)
are already quite problematic in Swift.  We believe that we
can mitigate the majority of the impact here with targeted
improvements to the collection APIs.

これらの議論の多くが subscript にも適用できるが、
現在言語において subscript は技術的には全く stored ではない。
subscript を通した値型の要素へのアクセスは
値全体へのアクセスとして扱われるので、
その値への他のあらゆるアクセスと重なるとみなされる。
この最も重要な結論は２つの異なる配列の要素に同時にアクセスすることはできないという事だ。
これは配列を取り扱ういくつかのよくあるイディオムを妨げるが、
いくつかのケース
(配列の異なるスライスを並行に変更するような)
はSwiftではすでにかなり問題がある。
ここでの大半の影響はコレクションAPIをそれを目的に改良することで緩和することができると信じる。

## Enforcing the Law of Exclusivity
## 排他則の強制

There are three available mechanisms for enforcing the Law of
Exclusivity: static, dynamic, and undefined.  The choice
of mechanism must be decidable by a simple inspection of the
storage declaration, because the definition and all of its
direct accessors must agree on how it is done.  Generally,
it will be decided by the kind of storage being declared,
its container (if any), and any attributes that might be
present.

排他則を強制する３つの機構が利用できる。
静的、動的、未定義だ。
機構の選択は storage declaration を単純に見て
決定できなければならない、
なぜならこれら全てへの直接アクセスはそれがどうなされるかについて合意せねばならないからだ。
一般に、これはストレージがどのような種類に宣言されたか、
それのコンテナ (もしあれば)、
そして指定されるかもしれない修飾子によって決まる。

### Static enforcement
### 静的強制

Under static enforcement, the compiler detects that the
law is being violated and reports an error.  This is the
preferred mechanism where possible because it is safe, reliable,
and imposes no runtime costs.

静的強制下では、
コンパイラは規則が違反しているのを検出してエラーを報告する。
これは安全性、信頼性、実行時コストがかからないという理由で望ましい機構だ。

This mechanism can only be used when it is perfectly decidable.
For example, it can be used for value-type properties because
the Law, recursively applied, ensures that the base storage is
being exclusively accessed.  It cannot be used for ordinary
reference-type properties because there is no way to prove in
general that a particular object reference is the unique
reference to an object.  However, if we supported
uniquely-referenced class types, it could be used for their
properties.

この機構はそれが完全に決定可能なときにだけ使える。
例えば、これは値型のプロパティに使える、
なぜなら、法則は再帰的に適用され、
根本のストレージが排他的にアクセスされることが保証されているからだ。
これは普通の参照型プロパティには使えない、
なぜなら一般に特定のオブジェクトへの参照がユニークであることを
証明する方法がないからだ。
しかし、もしユニーク参照されるクラス型をサポートするなら、
それのプロパティについては使えるだろう。

In some cases, where desired, the compiler may be able to
preserve source compatibility and avoid an error by implicitly
inserting a copy instead.  This likely something we would only
do in a source-compatibility mode.

いくつかの望まれるケースで、
コンパイラはソース互換性を保持し、
暗黙のコピーを挿入することでエラーを報告するのを回避する
ことができるかもしれない。
これはソース互換モードだけで行うだろう。

Static enforcement will be used for:

静的強制が使われるのは

- immutable variables of all kinds,

  すべての種類のイミュータブル変数

- local variables, except as affected by the use of closures
  (see below),

  ローカル変数、ただしクロージャの使用の影響下にあるものを除く(後述)

- `inout` arguments, and

  `inout` 引数、それと、

- instance properties of value types.

  値型のインスタンスプロパティ

### Dynamic enforcement
### 動的強制

Under dynamic enforcement, the implementation will maintain
a record of whether each variable is currently being accessed.
If a conflict is detected, it will trigger a dynamic failure.
The compiler may emit an error statically if it detects that
dynamic enforcement will always detect a conflict.

動的強制下では、実装は変数それぞれが現在アクセスされているかの記録を維持するだろう。
もし衝突が検出されれば、それは動的失敗を送出する。
コンパイラは動的強制が必ず衝突を検出することを検出した場合は静的にエラーを吐いても良い。

The bookkeeping requires two bits per variable, using a tri-state
of "unaccessed", "read", and "modified".  Although multiple
simultaneous reads can be active at once, a full count can be
avoided by saving the old state across the access, with a little
cleverness.

その記録には変数ごとに2ビット必要だ、
"unaccessed", "read", "modified" の 3状態を表すために。
同時に複数の reads が有効になれるが、
古い状態をアクセスをまたいで保存する少し賢いやりかたで、
完全にカウントすることを避けられる。

The bookkeeping is intended to be best-effort.  It should reliably
detect deterministic violations.  It is not required to detect
race conditions; it often will, and that's good, but it's not
required to.  It *is* required to successfully handle the case
of concurrent reads without e.g. leaving the bookkeeping record
permanently in the "read" state.  But it is acceptable for the
concurrent-reads case to e.g. leave the bookkeeping record in
the "unaccessed" case even if there are still active readers;
this permits the bookkeeping to use non-atomic operations.
However, atomic operations would have to be used if the bookkeeping
records were packed into a single byte for, say, different
properties of a class, because concurrent accesses are
allowed on different variables within a class.

記録はベストエフォートになる事が意図される。
これは決定論的な違反を確実に検出する。
レースコンディションを検出する必要はない。
それはしばしばあるし、できたらよいが、必須ではない。
例えば状態記録が "read" になっていなくても
並行読み出しが成功することが求め *られる* 。
しかしまだ有効なリーダーがいるとしても状態記録が
"unaccessed" になることは受け入れられる。
これは記録に非アトミック操作を使うことを許す。
しかし、状態記録がもし1バイトにまとめられているなら、
アトミック操作を使わなければならないだろう、
なぜなら、クラスの異なるプロパティでは、
そのクラスの中で異なる変数に並行アクセスすることが許されているからだ。

When the compiler detects that an access is "instantaneous",
in the sense that none of the code executed during the access
can possibly cause a re-entrant access to the same variable,
it can avoid updating the bookkeeping record and instead just
check that it has an appropriate value.  This is common for
reads, which will often simply copy the value during the access.
When the compiler detects that all possible accesses are
instantaneous, e.g. if the variable is `private` or `internal`,
it can eliminate all bookkeeping.  We expect this to be fairly
common.

コンパイラがあるアクセスを "instantaneous" だと検出するとき、
つまりアクセス中に同じ変数への再入アクセスを起こす可能性があるどんなコードも実行できないとき、
状態記録を更新する事を回避して、ただ期待する値になっていることをチェックするだけにできる。
これはアクセスの間に値をただコピーするだけの reads でしばしばおｙくある。
コンパイラがありうる全てのアクセスが instantaneous だと検出するとき、
例えば変数が `private` か `internal` な時、
全ての状態記録を削除できる。
これはかなりよくあると期待する。

Dynamic enforcement will be used for:

動的強制が使われるであろうのは

- local variables, when necessary due to the use of closures
  (see below),

  ローカル変数、ただしクロージャの使用の必要がない時(後述)

- instance properties of class types,

  クラス型のインスタンスプロパティ

- `static` and `class` properties, and

  `static` と `class` プロパティ、そして

- global variables.

  グローバル変数

We should provide an attribute to allow dynamic enforcement
to be downgraded to undefined enforcement for a specific
property or class.  It is likely that some clients will find
the performance consequences of dynamic enforcement to be
excessive, and it will be important to provide them an opt-out.
This will be especially true in the early days of the feature,
while we're still exploring implementation alternatives and
haven't yet implemented any holistic optimizations.

特定のプロパティやクラスに対して動的強制を未定義強制にダウングレードさせる
修飾子を提供するべきだろう。
動的強制のもたらすパフォーマンスが過度なことを見つけたクライアントなどに対して、
それのオプトアウトを提供することは重要だろう。
特に機能が出てすぐはそうだろう、
まだ他の実装を探求しているし、
全体的な最適化がまだ実装されていないからだ。

Future work on isolating class instances may allow us to
use static enforcement for some class instance properties.

将来のクラスインスタンスの分離作業は
いくつかのクラスインスタンスのプロパティに
静的強制を使うことを可能にするかもしれない。

### Undefined enforcement
### 未定義強制

Undefined enforcement means that conflicts are not detected
either statically or dynamically, and instead simply have
undefined behavior.  This is not a desirable mechanism
for ordinary code given Swift's "safe by default" design,
but it's the only real choice for things like unsafe pointers.

未定義強制とは静的にも動的にも衝突を検出しないことを意味する。
そして単純に未定義の動作をする。
これは 「デフォルトで安全」なSwiftの設計からなる普通のコードにのは望まれない機構だが、
アンセーフポインタなどのようなものに対する唯一の現実的な選択だ。

Undefined enforcement will be used for:

未定義強制が使われであろうのは

- the `memory` properties of unsafe pointers.

  アンセーフポインタの `memory` プロパティ

### Enforcement for local variables captured by closures
### クロージャによってキャプチャされるローカル変数に対する強制

Our ability to statically enforce the Law of Exclusivity
relies on our ability to statically reason about where
uses occur.  This analysis is usually straightforward for a
local variable, but it becomes complex when the variable is
captured in a closure because the control flow leading to
the use can be obscured.  A closure can potentially be
executed re-entrantly or concurrently, even if it's known
not to escape.  The following principles apply:

排他則を静的に強制することは
それを使用する場所を静的にわかることができるかどうかに依存している。
この解析はローカル変数については通常は素直ですが、
クロージャにその変数がキャプチャされているときには複雑になる。
なぜなら制御フローが使用を見えなくするからだ。
クロージャは再入や並行に実行される可能性がある、
たとえそれがエスケープしないとわかっていても。
以下の原則が適用される。

- If a closure `C` potentially escapes, then for any variable
  `V` captured by `C`, all accesses to `V` potentially executed
  after a potential escape (including the accesses within `C`
  itself) must use dynamic enforcement unless all such accesses
  are reads.

  もしクロージャ `C` がエスケープしうるなら、
  `C` にキャプチャされるどんな変数 `V` についても、
  エスケープした後で実行されうる `V` への全てのアクセスは
  (`C` それ自体へのアクセスも含む)
  それらのアクセスが全て reads でない限り
  動的強制をせねばならない。

- If a closure `C` does not escape a function, then its
  use sites within the function are known; at each, the closure
  is either directly called or used as an argument to another
  call.  Consider the set of non-escaping closures used at
  each such call.  For each variable `V` captured by `C`, if
  any of those closures contains a write to `V`, all accesses
  within those closures must use dynamic enforcement, and
  the call is treated for purposes of static enforcement
  as if it were a write to `V`; otherwise, the accesses may use
  static enforcement, and the call is treated as if it were a
  read of `V`.

  もしクロージャ `C` がエスケープしないなら、
  それが関数の中で使われる場所がわかる。
  それぞれ、クロージャは直接呼び出されるか、
  他の呼び出しの引数として使われるかだ。
  そのような呼び出しごとのエスケープしないクロージャの集合を考えよ。
  `C` にキャプチャされる `V` それぞれについて、
  もしそれらのクロージャのどれかが `V` への書き込みを含むなら、
  それらのクロージャの中の全てのアクセスは動的強制せねばならず、
  呼び出しは `V` への書き込みがあるかのように
  静的強制の対象として扱われる。
  それ以外では、アクセスは静的強制だろう、
  そして呼び出しは `V` への読み出しのように扱われる。

It is likely that these rules can be improved upon over time.
For example, we should be able to improve on the rule for
direct calls to closures.

これらの規則は後々改良されるかもしれない。
例えば、クロージャの直接呼び出しの規則を改良することができるだろう。

## Explicit tools for ownership
## オーナーシップのための明示的なツール

### Shared values
### shared 値

A lot of the discussion in this section involves the new concept
of a *shared value*.  As the name suggests, a shared value is a
value that has been shared with the current context by another
part of the program that owns it.  To be consistent with the
Law of Exclusivity, because multiple parts of the program can
use the value at once, it must be read-only in all of them
(even the owning context).  This concept allows programs to
abstract over values without copying them, just like `inout`
allows programs to abstract over variables.

この章の多くの議論では「shared value」という新しい概念を含む。
その名前が示唆するように、shared valueは
それを保持しているプログラムの他の部分によって、
現在のコンテキストで共有される。
排他則と一致するように、
プログラムの複数の部分が値を同時に使えるので、
それ全ては読み込みのみでなければならない。
(たとえ所有しているコンテキストにおいても)
この概念はプログラムがそれらの値をコピーしないことを示す、
ちょうど `inout` が変数にたいして示すように。

(Readers familiar with Rust will see many similarities between
shared values and Rust's concept of an immutable borrow.)

(Rustに親しんだ読者は shared value と Rustのイミュータブルボローの概念と
多くの似ている点を見るだろう。)

When the source of a shared value is a storage reference, the
shared value acts essentially like an immutable reference
to that storage.  The storage is accessed as a read for the
duration of the shared value, so the Law of Exclusivity
guarantees that no other accesses will be able to modify the
original variable during the access.  Some kinds of shared
value may also bind to temporary values (i.e. an r-value).
Since temporary values are always owned by the current
execution context and used in one place, this poses no
additional semantic concerns.

shared value が storage reference から得られる時、
shared value はそのストレージへのイミュータブルリファレンスのように
本質的に働く。
shared value のある間、ストレージは read としてアクセスされるので、
排他則はそのアクセスの間、
他のアクセスが元の変数を変更できないことを保証する。
いくつかの種類の shared value はテンポラリな値に紐付いているかもしれない。
(r-value とも言う)
テンポラリな値はいつも現在の実行コンテキストによって保持され、
1箇所で使われるので、追加の意味的な考慮はもたらさない。

A shared value can be used in the scope that binds it
just like an ordinary parameter or `let` binding.
If the shared value is used in a place that requires
ownership, Swift will simply implicitly copy the value --
again, just like an ordinary parameter or `let` binding.

share value はちょうど通常の引数や let 束縛のようなスコープで使うことができる。
もしその shared value がオーナーシップが必要な場所で使われているなら、
Swift は単に暗黙のコピーをする。
かさねて、これはちょうど通常の引数や let 束縛のようだ。

#### Limitations of shared values
#### shared value の制限

This section of the document describes several ways to form
and use shared values.  However, our current design does not
provide general, "first-class" mechanisms for working with them.
A program cannot return a shared value, construct an array
of shared values, store shared values into `struct` fields,
and so on.  These limitations are similar to the existing
limitations on `inout` references.  In fact, the similarities
are so common that it will be useful to have a term that
encompasses both: we will call them *ephemerals*.

文書のこの章では share value と作ったり使ったりするいくつかの方法を説明する。
しかし、現在の設計は一般に提供をしない、これらを扱う「第一級の」機構を。
プログラムは shared value を返せない、
shared value の配列を作れない、
shared value を `struct` のフィールドに保存できない、など。
これらの制限は既存の `inout` 参照の制限と似ている。
実際、この類似性はとても共通しているので両方を包む用語をもつのが便利だ。
これを「ephemerals」と呼ぶ。

The fact that our design does not attempt to provide first-class
facilities for ephemerals is a well-considered decision,
born from a trio of concerns:

ephemeralsの第一級の融通を提供することを試みない設計にする事は、
よく考慮された設計であるのが事実だ。
3つの考察にもとづいている。

- We have to scope this proposal to something that can
  conceivably be implemented in the coming months.  We expect this
  proposal to yield major benefits to the language and its
  implementation, but it is already quite broad and aggressive.
  First-class ephemerals would add enough complexity to the
  implementation and design that they are clearly out of scope.
  Furthermore, the remaining language-design questions are
  quite large; several existing languages have experimented
  with first-class ephemerals, and the results haven't been
  totally satisfactory.

  この提案を場合によっては実装に数ヶ月かかるものだと見込んでいる。
  この提案が多くの利益を言語とそれによる実装にもたらすと期待しているが、
  これはとても広くて挑戦的でもある。
  第一級の sphemeral は実装と設計に範囲から外すには明らかに十分な複雑性を追加する。
  さらに、残された言語設計上の疑問はとても大きくなる。
  いくつかの既存の言語は第一級の ephemeral を持っているが、
  その結果は全く満足できるものにはなっていない。

- Type systems trade complexity for expressivity.
  You can always accept more programs by making the type
  system more sophisticated, but that's not always a good
  trade-off.  The lifetime-qualification systems behind
  first-class references in languages like Rust add a lot
  of complexity to the user model.  That complexity has real
  costs for users.  And it's still inevitably necessary
  to sometimes drop down to unsafe code to work around the
  limitations of the ownership system.  Given that a line
  does have to drawn somewhere, it's not completely settled
  that lifetime-qualification systems deserve to be on the
  Swift side of the line.

  型システムは複雑性と引き換えに表現性を手に入れる。
  型システムがより組織化されることでより多くのプログラムをいつも受け入れられる、
  しかしこれはいつも良いトレードオフではない。
  Rustのような言語は第一級の参照の上に生存期間修飾システムをもつが、
  ユーザーモデルに多くの複雑性をもたらす。
  この複雑さはユーザにとって実際のコストだ。
  そしてオーナーシップシステムの制限を回避するためにときどき
  安全ではないコードに落とす必要がそれでも回避できていない。
  そのような行をどこかしら書く必要がでるのは、
  生存期間修飾機構が完全に解決していない事で、
  Swiftにはそれは望まれていない。

- A Rust-like lifetime system would not necessarily be
  as powerful in Swift as it is in Rust.  Swift intentionally
  provides a language model which reserves a lot of
  implementation flexibility to both the authors of types
  and to the Swift compiler itself.

  Rustのような生存期間システムはRustの中でのそれほどSwiftの中では
  強力ではなく必要がないだろう。
  Swiftは意図的に型の作者とSwiftコンパイラそれ自身に
  実装の柔軟性を用意しておく言語モデルを提供している。

  For example, polymorphic storage is quite a bit more
  flexible in Swift than it is in Rust.  A
  `MutableCollection` in Swift is required to implement a
  `subscript` that provides accessor to an element for an index,
  but the implementation can satisfy this pretty much any way
  it wants.  If generic code accesses this `subscript`, and it
  happens to be implemented in a way that provides direct access
  to the underlying memory, then the access will happen in-place;
  but if the `subscript` is implemented with a computed getter
  and setter, then the access will happen in a temporary
  variable and the getter and setter will be called as necessary.
  This only works because Swift's access model is highly
  lexical and maintains the ability to run arbitrary code
  at the end of an access.  Imagine what it would take to
  implement a loop that added these temporary mutable
  references to an array -- each iteration of the loop would
  have to be able to queue up arbitrary code to run as a clean-up
  when the function was finished with the array.  This would
  hardly be a low-cost abstraction!  A more Rust-like
  `MutableCollection` interface that worked within the
  lifetime rules would have to promise that the `subscript`
  returned a pointer to existing memory; and that wouldn't
  allow a computed implementation at all.

  例えば、ポリモーフィックなストレージはSwiftではRustにくらべて
  かなり大きな柔軟性を持っている。
  Swift の `MutableCollection` は `subscript` の実装を求めていて、
  これによってインデックスの要素へのアクセスを提供するが、
  望むまさにどんな方法でも実装を満たすことができる。
  もしジェネリックコードがこの `subscript` にアクセスするなら、
  そしてそれが下請けのメモリに直接のアクセスを提供する方法で実装されていれば、
  アクセスはその場で発生する。
  しかしもし `subscript` が computed な getter と setter　で実装されていれば、
  アクセスはテンポラリな変数の中で起こり、getter と setter が必要に応じで呼ばれる。
  これが動くのは Swift のアクセスモデルが高度にレキシカルで、
  アクセスの終了までにどんなコードでも実行できる能力があるからだ。
  それらのテンポラリなミュータブル参照を配列に追加するループを作ることを想像せよ、
  それぞれのループのイテレーションで、
  配列と共に関数が終了するときにクリンアップとして実行する任意のコードを列に並べる事ができる。
  これはとても低コストの抽象化ではない。
  Rustのような言語での `MutableCollection` インターフェースは
  ライフタイム規則の中で動くので、
  `subscript` が存在するメモリへのポインタを返すことを約束するだろう。
  そして computed　な実装は全くできない。

  A similar problem arises even with simple `struct` members.
  The Rust lifetime rules say that, if you have a pointer to
  a `struct`, you can make a pointer to a field of that `struct`
  and it'll have the same lifetime as the original pointer.
  But this assumes not only that the field is actually stored
  in memory, but that it is stored *simply*, such that you can
  form a simple pointer to it and that pointer will obey the
  standard ABI for pointers to that type.  This means that
  Rust cannot use layout optimizations like packing boolean
  fields together in a byte or even just decreasing the
  alignment of a field.  This is not a guarantee that we are
  willing to make in Swift.

  似たような問題はシンプルな `struct` のメンバでさえ起こる。
  Rustのライフタイム規則が言うには、
  `struct` へのポインタを持っているなら、
  `struct` のフィールドへのポインタを作ることができて、
  元々のポインタと同じライフタイムをもつ。
  しかしこれはフィールドが実際にメモリ上に保存されていることだけでなく、
  これが「シンプルに」保持されていることを仮定する、
  それへの単純なポインタを作ることができて、
  その型のポインタへの標準ABIに従うことになる。
  これはRustが
  boolean フィールドを一つのバイトにまとめたり、
  フィールドのアライメントを減らすなどの、
  レイアウト最適化を使えないことを意味する。
  これはSwiftに作ることを考えていない保証だ。

For all of these reasons, while we remain theoretically
interested in exploring the possibilities of a more
sophisticated system that would allow broader uses of
ephemerals, we are not proposing to take that on now.  Since
such a system would primarily consist of changes to the type
system, we are not concerned that this will cause ABI-stability
problems in the long term.  Nor are we concerned that we will
suffer from source incompatibilities; we believe that any
enhancements here can be done as extensions and generalizations
of the proposed features.

これらの理由全てによって、
もっと組織化されたシステムの可能性を探求することに理論的な興味はあるが、
いまそれを取ることは提案しない。
こうしたシステムは型システムに対する一級の変更からなるが、
長期的にABI安定性の問題を引き起こすとは考えていない。
そしてソースの非互換に苦しめられるとも考えていない。
これにたいするいかなる改良も拡張と機能提案の一般化になると信じている。

### Local ephemeral bindings
### ローカル ephemeral 束縛

It is already a somewhat silly limitation that Swift provides
no way to abstract over storage besides passing it as an
`inout` argument.  It's an easy limitation to work around,
since programmers who want a local `inout` binding can simply
introduce a closure and immediately call it, but that's an
awkward way of achieving something that ought to be fairly easy.

Swiftはそれを `inout` 引数に渡すこと以外で、
storageに対する抽象化の手段を提供しない、
いくらか愚かな制限がすでにある。
回避するのは簡単な制限だが、
ローカル `inout` 束縛をしたいプログラマは、
ただクロージャを作ってそれを即座に呼ぶだけだが、
何かを達成するには無様なやりかたで、
これを簡単にすべきであろう。

Shared values make this limitation even more apparent, because
a local shared value is an interesting alternative to a local `let`:
it avoids a copy at the cost of preventing other accesses to
the original storage.  We would not encourage programmers to
use `shared` instead of `let` throughout their code, especially
because the optimizer will often be able to eliminate the copy
anyway.  However, the optimizer cannot always remove the copy,
and so the `shared` micro-optimization can be useful in select
cases.  Furthermore, eliminating the formal copy may also be
semantically necessary when working with non-copyable types.

shared value はこの制限をよりいっそう明白にする、
なぜならローカル shared value は ローカル `let` の興味深い代替だからだ。
これは元のストレージへのその他のアクセスを防ぐコストがかかるコピーを避ける。
プログラマコードのいたるところで `let` の代わりに `shared` を使うことを
推奨はしない。
なぜなら特にオプティマイザがとにかくコピーを削除することがしばしば可能だからだ。
しかし、オプティマイザがいつもコピーを削除できるわけではなく、
特定のケースでは `shared` による小さな最適化が便利に鳴る。
さらに、形式的なコピーを除去することは、
コピー不可能な型を取り扱うときに意味的に必要になるだろう。

We propose to remove this limitation in a straightforward way:

この制限を素直な方法で取り除くことを提案する。

```
inout root = &tree.root

shared elements = self.queue
```

The initializer is required and must be a storage reference
expression.  The access lasts for the remainder of the scope.

初期化値が必要で storage reference expression でなければならない。
アクセスはスコープの残りの間続く。

### Function parameters
### 関数の引数

Function parameters are the most important way in which
programs abstract over values.  Swift currently provides
three kinds of argument-passing:

関数の引数はプログラムが値について抽象化する最も重要な方法だ。
Swiftは現在3種類の引数の渡し方を提供している。

- Pass-by-value, owned.  This is the rule for ordinary
  arguments.  There is no way to spell this explicitly.

  値渡し、owned。
  これは普通の引数のルールだ。
  これを明示的に指定する方法は無い。

- Pass-by-value, shared.  This is the rule for the `self`
  arguments of `nonmutating` methods.  There is no way to
  spell this explicitly.

  値渡し、shared。
  これは `nonmutating` メソッドの `self` 引数の規則だ。
  これを明示的に指定する方法は無い。

- Pass-by-reference.  This is the rule used for `inout`
  arguments and the `self` arguments of `mutating` methods.

  参照渡し。
  `inout` 引数と、 `mutating` メソッドの `self` 引数で使われる規則だ。

Our proposal here is just to allow the non-standard
cases to be spelled explicitly:

この提案では非標準の場合を明示的に指定できるようにする。

- A function argument can be explicitly declared `owned`:

  関数の引数は `owned` と明示できる。

  ```
  func append(_ values: owned [Element]) {
    ...
  }
  ```

  This cannot be combined with `shared` or `inout`.

  これは `shared` と `inout` と組み合わせられない。

  This is just an explicit way of writing the default, and
  we do not expect that users will write it often unless
  they're working with non-copyable types.

  これはデフォルトを明示的に示すだけのもので、
  ユーザーがこれをしばしば書くことを想定していない、
  コピー不可能な型を取り扱わない限り。

- A function argument can be explicitly declared `shared`.

  関数の引数は `shared` と明示できる。

  ```
  func ==(left: shared String, right: shared String) -> Bool {
    ...
  }
  ```

  This cannot be combined with `owned` or `inout`.

  これは `owned` と `inout` と組み合わせられない。

  If the function argument is a storage reference expression,
  that storage is accessed as a read for the duration of the
  call.  Otherwise, the argument expression is evaluated as
  an r-value and that temporary value is shared for the call.
  It's important to allow temporary values to be shared for
  function arguments because many function parameters will be
  marked as `shared` simply because the functions don't
  actually benefit from owning that parameter, not because it's in
  any way semantically important that they be passed a
  reference to an existing variable.  For example, we expect
  to change things like comparison operators to take their
  parameters `shared`, because it needs to be possible to
  compare non-copyable values without claiming them, but
  this should not prevent programmers from comparing things
  to literal values.

  もし関数の引数が storage reference expression ならば、
  ストレージはその呼び出しの間は read としてアクセスされる。
  そうでなければ、引数の式が r-value として評価され、
  そのテンポラリな値は呼び出しの間 shared となる。
  関数の引数でテンポラリな値を shared にできることは重要で、
  なぜなら多くの関数引数はシンプルに `shared` を指定できるからで、
  なぜなら関数が値を保有する利益は実際なく、
  存在する変数への参照として渡されることが意味的にとにかく重要ではないからだ。
  たとえば、比較演算子のようなものがその引数を `shared` を取るように変更することを想定している、
  なぜならこれはそれらを所有することなしに、
　コピー不可能な値について比較できる必要があるからで、
  これはプログラマがそれをリテラル値と比較するのを防がないだろう。

  Like `inout`, this is part of the function type.  Unlike
  `inout`, most function compatibility checks (such as override
  and function conversion checking) should succeed with a
  `shared` / `owned` mismatch.  If a function with an `owned`
  parameter is converted to (or overrides) a function with a
  `shared` parameter, the argument type must actually be
  copyable.

  `inout` のように、これは関数の型の一部だ。
  `inout` と異なり、ほとんどの関数の互換性検査 (オーバライドや変換検査)
  は `shared` / `owned` の違いについては通るはずだ。
  `owned` 引数をもつ関数が `shared` 引数の関数に変換する (またはオーバーライド)
  ためには、引数の型がコピー可能でなければならない。

- A method can be explicitly declared `consuming`.

  メソッドは `consuming` を明示的に指定できる。

  ```
  consuming func moveElements(into collection: inout [Element]) {
    ...
  }
  ```

  This causes `self` to be passed as an owned value and therefore
  cannot be combined with `mutating` or `nonmutating`.

  これは `self` を owned として渡すので、
  `mutating` と `nonmutating` と組み合わせられない。

  `self` is still an immutable binding within the method.

  `self` はこのメソッドの中ではいまだイミュータブル束縛だ。

### Function results
### 関数の返り値

As discussed at the start of this section, Swift's lexical
access model does not extend well to allowing ephemerals
to be returned from functions.  Performing an access requires
executing storage-specific code at both the beginning and
the end of the access.  After a function returns, it has no
further ability to execute code.

この章の議論の開始点として、
Swiftのレキシカルアクセスモデルは関数から ephemeral を返せるように拡張はしない。
アクセスを実行することは
アクセスの開始と終了のところでストレージ固有なコードを求める。
関数が返った後、コードをさらに実行する余地は無い。

We could, conceivably, return a callback along with the
ephemeral, with the expectation that the callback will be
executed when the caller is done with the ephemeral.
However, this alone would not be enough, because the callee
might be relying on guarantees from its caller.  For example,
considering a `mutating` method on a `struct` which wants
to returns an `inout` reference to a stored property.  The
correctness of this depends not only on the method being able
to clean up after the access to the property, but on the
continued validity of the variable to which `self` was bound.
What we really want is to maintain the current context in the
callee, along with all the active scopes in the caller, and
simply enter a new nested scope in the caller with the
ephemeral as a sort of argument.  But this is a well-understood
situation in programming languages: it is just a kind of
co-routine.  (Because of the scoping restrictions, it can
also be thought of as sugar for a callback function in
which `return`, `break`, etc. actually work as expected.)

考えられる限りでは、ephemeral な callback を返すのは、
ephemeral と共に caller が終了した時に callback が実行されることを期待する。
しかし、これだけで十分ではなく、
callee はその caller からの保証に依存するかもしれない。
例えば、
`struct` の `mutating` メソッドを考えよ、
それが stored property への `inout` 参照を返すような。
これの正しさはプロパティへのアクセスのあとメソッドがクリンアップできるかだけではなく、
`self` が束縛されている変数が継続して有効かどうかに依存する。
callerの全ての有効なスコープとともに、
calleeの中で現在のコンテキストの維持を求めるなら、
引数のような ephemeral のある caller の中に新しいネストしたスコープをシンプルに作る。
しかしこれはプログラミング言語のなかでよく理解された状況だ。
これはちょうどコルーチンのようなものだ。
(スコープの制限のおかげで、
`return` や `break` などをコールバック関数のシュガーと考えることができて、
期待するように実際に動く。)

In fact, co-routines are useful for solving a number of
problems relating to ephemerals.  We will explore this
idea in the next few sub-sections.

実際、コルーチンは ephemeral に関する多くの問題を解決するのに便利だ。
次のいくつかの節でこのアイデアについて調査する。

### `for` loops
### `for` ループ

In the same sense that there are three interesting ways of
passing an argument, we can identify three interesting styles
of iterating over a sequence.  Each of these can be expressed
with a `for` loop.

引数を渡すための興味深い3つの方法があり、それと同じ意味で、
シーケンスをイテレートするときの3つの興味深いスタイルを明らかにする。
これらは `for` ループとともに説明できる。

#### Consuming iteration
#### consuming イテレーション

The first iteration style is what we're already familiar with
in Swift: a consuming iteration, where each step is presented
with an owned value.  This is the only way we can iterate over
an arbitrary sequence where the values might be created on demand.
It is also important for working with collections of non-copyable
types because it allows the collection to be destructured and the
loop to take ownership of the elements.  Because it takes ownership
of the values produced by a sequence, and because an arbitrary
sequence cannot be iterated multiple times, this is a
`consuming` operation on `Sequence`.

第一のイテレーションスタイルはSwiftで既によくられたものだ。
consumingイテレーションでは、各ステップはowned valueと共に提供される。
これは値がその場で作られるかもしれないときに
どんなシーケンスについてもイテレートできる唯一の方法だ。
これはまたコピー不可能な型のコレクションを取り扱う時に重要で、
なぜならコレクションが破棄し、要素のオーナーシップをループが取得することを可能にするからだ。
これはシーケンスから提供された値のオーナーシップを取得するため、
全てのシーケンスで複数回イテレートすることはできない。
これは `Sequence` 上の `consuming` 処理だ。

This can be explicitly requested by declaring the iteration
variable `owned`:

これは `owned` イテレーション変数を宣言することによって明示的に要求できる。

```
for owned employee in company.employees {
  newCompany.employees.append(employee)
}
```

It is also used implicitly when the requirements for a
non-mutating iteration are not met.  (Among other things,
this is necessary for source compatibility.)

また、non-mutating イテレーションが求められてるがそれがない場合に、
暗黙に使われる。
(特に、ソース互換性のために必要だ)

The next two styles make sense only for collections.

次の2つのスタイルはコレクションにちょうど筋が通る。

#### Non-mutating iteration
#### non-mutating イテレーション

A non-mutating iteration simply visits each of the elements
in the collection, leaving it intact and unmodified.  We have
no reason to copy the elements; the iteration variable can
simply be bound to a shared value.  This is a `nonmutating`
operation on `Collection`.

non-mutating イテレーションは単純にコレクションの要素それぞれを訪れ、
それを傷つけず変更せずに去る。
要素のコピーをする理由は無い。
イテレーション変数は単純に shared value に束縛される。
これは `Collection` に対する `nonmutating` 処理となる。

This can be explicitly requested by declaring the iteration
variable `shared`:

これはイテレーション変数 `shared` として宣言することで明示的に要求できる。

```
for shared employee in company.employees {
  if !employee.respected { throw CatastrophicHRFailure() }
}
```

It is also used by default when the sequence type is known to
conform to `Collection`, since this is the optimal way of
iterating over a collection.

これはシーケンス型が `Collection` とわかっているときにデフォルトで使われる。
なぜなら collection をイテレートする時の最適な方法だからだ。

```
for employee in company.employees {
  if !employee.respected { throw CatastrophicHRFailure() }
}
```

If the sequence operand is a storage reference expression,
the storage is accessed for the duration of the loop.  Note
that this means that the Law of Exclusivity will implicitly
prevent the collection from being modified during the
iteration.  Programs can explicitly request an iteration
over a copy of the value in that storage by using the `copy`
intrinsic function on the operand.

もしシーケンスのオペランドが storage reference expression なら、
そのストレージはループの間アクセスされる。
これは排他則がイテレーションの間コレクションが変更されることを暗黙に防ぐことに注意せよ。
プログラムはオペランドに `copy` という組み込み関数を使うことで
ストレージの値をコピーしながらイテレーションすることを明示的に要求できる。

#### Mutating iteration
#### mutating イテレーション

A mutating iteration visits each of the elements and
potentially changes it.  The iteration variable is an
`inout` reference to the element.  This is a `mutating`
operation on `MutableCollection`.

mutating イテレーションはそれぞれの要素を訪問し、
変更する可能性がある。
イテレーション変数は要素への `inout` 参照だ。
これが `MutableCollection` への `mutating` 操作だ。

This must be explicitly requested by declaring the
iteration variable `inout`:

これはイテレーション変数を `inout` で宣言することで明示的に要求できる。

```
for inout employee in company.employees {
  employee.respected = true
}
```

The sequence operand must be a storage reference expression.
The storage will be accessed for the duration of the loop,
which (as above) will prevent any other overlapping accesses
to the collection.  (But this rule does not apply if the
collection type defines the operation as a non-mutating
operation, which e.g. a reference-semantics collection might.)

シーケンスオペランドは storage reference expression でなければならない。
ストレージはループの間アクセスされ、
コレクションへのあらゆる他の重なったアクセスを防ぐ。
(しかしこのルールはもしコレクションがnon-mutating操作として定義しているときには適用されない、
例えば参照意味論的なコレクションがそうかもしれない)

#### Expressing mutating and non-mutating iteration
#### mutating と non-mutating イテレーションの記述

Mutating and non-mutating iteration require the collection
to produce ephemeral values at each step.  There are several
ways we could express this in the language, but one reasonable
approach would be to use co-routines.  Since a co-routine does
not abandon its execution context when yielding a value to its
caller, it is reasonable to allow a co-routine to yield
multiple times, which corresponds very well to the basic
code pattern of a loop.  This produces a kind of co-routine
often called a generator, which is used in several major
languages to conveniently implement iteration.  In Swift,
to follow this pattern, we would need to allow the definition
of generator functions, e.g.:

mutatingとnon-mutatingイテレーションはコレクションが各ステップで
ephemeral な値を提供する必要がある。
これを言語の中で表現するにはいくつかの方法があるが、
一つの妥当な方法はコルーチンを使うことだろう。
コルーチンはその caller に値を渡す時その実行コンテキストを捨てないので、
コルーチンが複数回値を出せるようにするのは妥当だ。
これはループの基本コードパターンととても良く対応する。
この種類のコルーチンはしばしばジェネレータと呼ばれ、
いくつかの主要な言語でイテレーションを便利に実装するのに使われている。
Swiftでもこのパターンに従い、
ジェネレータ関数を定義できるようにする必要があろう。

```
mutating generator iterateMutable() -> inout Element {
  var i = startIndex, e = endIndex
  while i != e {
    yield &self[i]
    self.formIndex(after: &i)
  }
}
```

On the client side, it is clear how this could be used to
implement `for` loops; what is less clear is the right way to
allow generators to be used directly by code.  There are
interesting constraints on how the co-routine can be used
here because, as mentioned above, the entire co-routine
must logically execute within the scope of an access to the
base value.  If, as is common for generators, the generator
function actually returns some sort of generator object,
the compiler must ensure that that object does not escape
that enclosing access.  This is a significant source of
complexity.

クライアント側では、これが `for` ループを実装するために
どのように使えるのかは明らかだ。
ややはっきりしないのはコードから直接ジェネレータを使う正しい方法だ。
ここにはどのようにコルーチンが使えるかについての興味深い制約がある、
なぜなら、上述したように、
コルーチン全体で論理的には根本の値にアクセスできる
スコープの中で実行される必要があるからだ。
ジェネレータでも共通して、もし、
ジェネレータ関数が実際になんらかの種類のジェネレータオブジェクトを返したら、
コンパイラはアクセスを包んでそれがエスケープしないことを仮定せねばならない。
これは大きくソースを複雑にする。

### Generalized accessors
### 一般化されたアクセサ

Swift today provides very coarse tools for implementing
properties and subscripts: essentially, just `get` and `set`
methods.  These tools are inadequate for tasks where
performance is critical because they don't allow direct
access to values without copies.  The standard library
has access to a slightly broader set of tools which can
provide such direct access in limited cases, but they're
still quite weak, and we've been reluctant to expose them to
users for a variety of reasons.

Swiftは現在プロパティとサブスクリプトを実装する道具をとても荒く提供している。
本質的にただの `get` と `set` メソッドだ。
これらの道具はパフォーマンスが重要な課題には不十分だ、
なぜならコピーを作らない値への直接アクセスを許さないからだ。
標準ライブラリは限られた場合にそのような直接アクセスを提供する
すこし広い範囲の道具にアクセスするが、
それらはいまだとても弱く、
様々な目的のためにこれらをユーザーに使わせることには気乗りしていなかった。

Ownership offers us an opportunity to revisit this problem
because `get` doesn't work for collections of non-copyable
types because it returns a value, which must therefore be
owned.  The accessor really needs to be able to yield a
shared value instead of returning an owned one.  Again,
one reasonable approach for allowing this is to use a
special kind of co-routine.  Unlike a generator, this
co-routine would be required to yield exactly once.
And there is no need to design an explicit way for programmers
to invoke one because these would only be used in accessors.

オーナーシップはこの問題を再検討する機会をもたらす、
なぜなら `get` はコピー不可能な型のコレクションを取り扱うことができないからだ、
なぜならこれは返り値があり、これは結果として所有されねばならないからだ。
アクセサは所有してる値を返す代わりに
shared value を作り出すことができることが
本当に必要だ。
繰り返すが、コルーチンのような特別なものを使うことは
これを可能にする一つの合理的なアプローチだ。 

The idea is that, instead of defining `get` and `set`,
a storage declaration could define `read` and `modify`:

アイデアとしては、
`get` と `set` を定義する代わりに、
storage declaration を `read` と `modify` にしうる。

```
var x: String
var y: String
var first: String {
  read {
    if x < y { yield x }
    else { yield y }
  }
  modify {
    if x < y { yield &x }
    else { yield &y }
  }
}
```

A storage declaration must define either a `get` or a `read`
(or be a stored property), but not both.

storage declaration は `get` か `read` のどちらかを定義せねばならない
(もしくは stored property にする)、しかし両方は定義できない。

To be mutable, a storage declaration must also define either a
`set` or a `modify`.  It may also choose to define *both*, in
which case `set` will be used for assignments and `modify` will
be used for modifications.  This is useful for optimizing
certain complex computed properties because it allows modifications
to be done in-place without forcing simple assignments to first
read the old value; however, care must be taken to ensure that
the `modify` is consistent in behavior with the `get` and the
`set`.

ミュータブルにするため、 storage declarationは
`set` と `modify` のどちらかを定義せねばならない。
「両方」定義することを選んでもよく、
その場合には `set` は代入で使用され、
`modify` は modification に使用されるだろう。
これはいくらか複雑な computed property の最適化に便利だ、
なぜなら 単純な代入で最初に古い値を読むことなく
modification をその場で終えられるようにするからだ。
しかし、 `modify` が `get` と `set` の振る舞いと一貫性があるように
保証することに注意せねばならない。

### Intrinsic functions
### 組み込み関数

#### `move`
#### `move`

The Swift optimizer will generally try to move values around
instead of copying them, but it can be useful to force its hand.
For this reason, we propose the `move` function.  Conceptually,
`move` is simply a top-level function in the Swift standard
library:

Swiftのオプティマイザは一般に値をコピーする代わりにそれをムーブできないか試みるが、
それを手で強制できれば便利だろう。
この理由で、 `move` 関数を提案する。
概念的に、 `move` はSwift標準ライブラリの単純なトップレベル関数だ。

```
func move<T>(_ value: T) -> T {
  return value
}
```

However, it is blessed with some special semantics.  It cannot
be used indirectly.  The argument expression must be a reference
to some kind of local owning storage: either a `let`, a `var`,
or an `inout` binding.  A call to `move` is evaluated by
semantically moving the current value out of the argument
variable and returning it as the type of the expression,
leaving the variable uninitialized for the purposes of the
definitive-initialization analysis.  What happens with the
variable next depends on the kind of variable:

しかし、これはいくらかの特殊な意味付けをされている。
これを間接的に使うことはできない。
引数の式はローカルに所有するストレージをもつなにかへの参照でなければならない。
`let`, `var`, `inout` 束縛のいずれかだ。
`move` の呼び出しは意味論的には引数の変数から現在の値を取り出して、
式の型の値として返すように評価され、
残された変数は定義的初期化解析において未初期化として扱われる。
変数に次に何がおきるかは変数の種類に依存する。

- A `var` simply transitions back to being uninitialized.
  Uses of it are illegal until it is assigned a new value and
  thus reinitialized.

  `var` は単純に未初期化状態に戻る。
  新しい値が代入されてそれが最初期化されるまではそれの使用は違法となる。

- An `inout` binding is just like a `var`, except that it is
  illegal for it to go out of scope uninitialized.  That is,
  if a program moves out of an `inout` binding, the program
  must assign a new value to it before it can exit the scope
  in any way (including by throwing an error).  Note that the
  safety of leaving an `inout` temporarily uninitialized
  depends on the Law of Exclusivity.

  `inout` 束縛はちょうど `var` に似ているが、
  未初期化状態でスコープを出ることは違法となる点が違う。
  もしプログラムが `inout` 束縛から値をムーブするとき、
  プログラムはどうにかしてスコープから出られるようになるまえに、
  (errorをthrowすることも含む) 
  新しい値をそれに代入せねばならないということだ。
  `inout` が一時的に未初期化でなくなる事の安全性は
  排他則によることに注意せよ。

- A `let` cannot be reinitialized and so cannot be used
  again at all.

  `let` は再初期化することができないので再び使うことは全くできない。

This should be a straightforward addition to the existing
definitive-initialization analysis, which proves that local
variables are initialized before use.

これは既存のこれはローカル変数が使う前に初期化されていることを確かめている、
定義的初期化解析に素直に追加できる。

#### `copy`
#### `copy`

`copy` is a top-level function in the Swift standard library:

`copy` はSwift標準ライブラリのトップレベル関数だ

```
func copy<T>(_ value: T) -> T {
  return value
}
```

The argument must be a storage reference expression.  The
semantics are exactly as given in the above code: the argument
value is returned.  This is useful for several reasons:

引数は storage reference expression でなければならない。
意味論は上のコードで与えたまさにそれだ。
引数の値が返される。
これはいくつかの理由で便利だ。

- It suppresses syntactic special-casing.  For example, as
  discussed above, if a `shared` argument is a storage
  reference, that storage is normally accessed for the duration
  of the call.  The programmer can suppress this and force the
  copy to complete before the call by calling `copy` on the
  storage reference before.

  これは構文上の特殊な場合を抑制する。
  例えば既に議論したように、
  もし `shared` 引数が storage referenceなら、
  ストレージは通常呼び出しの間アクセスされる。
  プログラマはそれを抑制し、
  前もって storage reference に `copy` を呼び出して
  関数を呼び出す前にコピーを終わらせることを強制できる。

- It is necessary for types that have suppressed implicit
  copies.  See the section below on non-copyable types.

  暗黙のコピーが抑制された型にとって必要だ。
  コピー不可能な型についての続く章を見よ。

#### `endScope`
#### `endScope`

`endScope` is a top-level function in the Swift standard library:

`endScope` はSwift標準ライブラリのトップレベル関数だ。

```
func endScope<T>(_ value: T) -> () {}
```

The argument must be a reference to a local `let`, `var`, or
standalone (non-parameter, non-loop) `inout` or `shared`
declaration.  If it's a `let` or `var`, the variable is
immediately destroyed.  If it's an `inout` or `shared`,
the access immediately ends.

引数はローカルの `let`, `var` への参照か
独立な (引数やループではない) `inout` か `shared` 宣言でなければならない。
もし `let` か `var` なら、変数は即座に破棄される。
もし `inout` か `shared` なら、アクセスは即座に終了する。

The definitive-initialization analysis must prove that
the declaration is not used after this call.  It's an error
if the storage is a `var` that's been captured in an escaping
closure.

定義的初期化解析は宣言がこれの呼び出しの後で使われていない事を検証せねばならない。
もしストレージが `var` でエスケープするクロージャにキャプチャされていたらエラーだ。

This is useful for ending an access before control reaches the
end of a scope, as well as for micro-optimizing the destruction
of values.

これはアクセスをスコープの終わりに到達するまえに終わらせるためはもちろん、
値の破棄の小さな最適化にも便利だ。

`endScope` provides a guarantee that the given variable has
been destroyed, or the given access has ended, by the time
of the execution of the call.  It does not promise that these
things happen exactly at that point: the implementation is
still free to end them earlier.

`endScope` は
呼び出しの実行が終わる時に、
与えられた変数が破棄されることか、
与えられたアクセスが終了することの
保証を提供する。
これはそれらがまさにその時点で起きることを約束はしない。
実装はそれらをより早く終了させることがいまだできる。

### Lenses
### レンズ

Currently, all storage reference expressions in Swift are *concrete*:
every component is statically resolvable to a storage declaration.
There is some recurring interest in the community in allowing programs
to abstract over storage, so that you might say:

現在、Swiftの全ての storage reference expression は 「concrete」 だ。
全ての構成要素は静的に storage declaration へと解決できる。
コミュニティにはストレージを抽象化したプログラムを可能にする
いくつかの繰り返し起きる興味がある。
これによってこう言えるかもしれない。

```
let prop = Widget.weight
```

and then `prop` would be an abstract reference to the `weight`
property, and its type would be something like `(Widget) -> Double`.

そして `prop` は `weight` プロパティへの抽象的な参照となり、
これの型は `(Widget) -> Double` のようなものとなる。

This feature is relevant to the ownership model because an ordinary
function result must be an owned value: not shared, and not mutable.
This means lenses could only be used to abstract *reads*, not
*writes*, and could only be created for copyable properties.  It
also means code using lenses would involve more copies than the
equivalent code using concrete storage references.

この機能はオーナーシップモデルと関係がある、
なぜなら通常の関数の返り値は常に owned value でなければならないからだ。
shared ではないし mutable ではない。
これはレンズが抽象的な reads には使えるが writes には使えないことを意味する、
そしてコピー可能プロパティのみにだけ作成できる。
これはレンズを使ったコードは concrete な storage reference を使った
等価なコードより多くのコピーを引き起こすことを意味する。

Suppose that, instead of being simple functions, lenses were their
own type of value.  An application of a lens would be a storage
reference expression, but an *abstract* one which accessed
statically-unknown members.  This would require the language
implementation to be able to perform that sort of access
dynamically.  However, the problem of accessing an unknown
property is very much like the problem of accessing a known
property whose implementation is unknown; that is, the language
already has to do very similar things in order to implement
generics and resilience.

単純な関数であるかわりにレンズがそれらの値の型の own だったと想像せよ。
レンズの適用は storage reference expression となろう、
しかし 「抽象的な」それは静的にはどのメンバにアクセスするかわからない。
これは言語実装が動的なアクセスの一種をもたらすことを可能とする。
しかし、知らないプロパティへのアクセスの問題は
実装がわからない知っているプロパティへのアクセスの問題ととても良く似ている。
これは、言語は既にジェネリクスとレジリエンスの実装のためにとても似たものに取り組んだ。

Overall, such a feature would fit in very neatly with the
ownership model laid out here.

全体として、このような機能はここで示したオーナーシップモデルによって、
とても綺麗に適合するだろう。

## Non-copyable types
## コピー不可能な型

Non-copyable types are useful in a variety of expert situations.
For example, they can be used to efficiently express unique
ownership.  They are also interesting for expressing values
that have some sort of independent identity, such as atomic
types.  They can also be used as a formal mechanism for
encouraging code to work more efficiently with types that
might be expensive to copy, such as large struct types.  The
unifying theme is that we do not want to allow the type to
be copied implicitly.

コピー不可能な型は様々な専門的な場面で便利だ。
例えば、これはユニークオーナーシップを効率的に表現するのに使うことができる。
これはアトミック型のような独立した同一性のようなものをもつ値を表現するためにもまた興味深い。
これはコピーのコストが高いかもしれない型を効率的に取り扱う
コードを推奨する形式的な機構としても使うことができる。
統一されたテーマは暗黙にコピーされることを許したくないということだ。

The complexity of handling non-copyable types in Swift
comes from two main sources:

Swiftでコピー不可能な方を扱う複雑さは2つの主たる要因がある。

- The language must provide tools for moving values around
  and sharing them without forcing copies.  We've already
  proposed these tools in this document because they're
  equally important for optimizing the use of copyable types.

  言語は値をムーブする事と強制コピーされずに共有する事のための道具を提供せねばならない。
  既にこれらの道具をこの文書で提案した。
  なぜならこれらはコピー可能な型の最適化のために等しく重要だからだ。

- The generics system has to be able to express generics over
  non-copyable types without massively breaking source
  compatibility and forcing non-copyable types on everybody.

  ジェネリクスシステムはソース互換性を大きく壊すことなく、
  全てをコピー不可能な型に強制されずに、
  コピー不可能な型をジェネリックに表現できねばならない。

Otherwise, the feature itself is pretty small.  The compiler
implicitly emits moves instead of copies, just like we discussed
above for the `move` intrinsic, and then diagnoses anything
that that didn't work for.

それ意外では、機能それ自体はとても小さい。
コンパイラは
ちょうど `move` 組み込み命令で既に議論したのと同じように、
暗黙のコピーの代わりにムーブを吐き出し、
それで何か動かないことがないか診断する。

### `moveonly` contexts
### `moveonly` コンテキスト

The generics problem is real, though.  The most obvious way
to model copyability in Swift is to have a `Copyable`
protocol which types can conform to.  An unconstrained type
parameter `T` would then not be assumed to be copyable.
Unfortunately, this would be a disaster for both source
compatibility and usability, because almost all the existing
generic code written in Swift assumes copyability, and
we really don't want programmers to have to worry about
non-copyable types in their first introduction to generic
code.

しかし、ジェネリクスの問題は現実だ。
Swiftでコピー可能性を示すもっとも明らかな方法は 
`Copyable` プロトコルをつくり型がそれを満たすことだろう。
制約されない型 `T` はコピー可能ではないと仮定されるだろう。
残念ながら、これはソース互換性と使いやすさを崩壊させるだろう、
なぜならほとんど全ての既存のSwiftで書かれたジェネリックコードは
コピー可能性を仮定しているからだ。
それにプログラマがジェネリックコードを最初に導入する時に
コピー不可能な型について心配する必要があるようには本当にしたくない。

Furthermore, we don't want types to have to explicitly
declare conformance to `Copyable`.  That should be the
default.

さらに、`Copyable` を満たす宣言を明示的に型に宣言させるようにはしたくない。
これはデフォルトが良い。

The logical solution is to maintain the default assumption
that all types are copyable, and then allow select contexts
to turn that assumption off.  We will call these contexts
`moveonly` contexts.  All contexts lexically nested within
a `moveonly` context are also implicitly `moveonly`.

論理的な答えは全ての型がコピー可能であるというデフォルトの仮定を維持して、
選択したコンテキストではその仮定を無くせることだろう。
これを `moveonly` コンテキストと呼ぶ。
`moveonly` コンテキストにレキシカルにネストされた全てのコンテキストもまた
暗黙に `moveonly` である。

A type can be a `moveonly` context:

型は `moveonly` コンテキストにできる。

```
moveonly struct Array<Element> {
  // Element and Array<Element> are not assumed to be copyable here
  // Element と Array<Element> はここではコピー可能と仮定されない
}
```

This suppresses the `Copyable` assumption for the type
declared, its generic arguments (if any), and their
hierarchies of associated types.

これは宣言された型とそのジェネリック引数(もしあれば)、
そして associated type のヒエラルキー  
の `Copyable` な仮定を抑制しする。

An extension can be a `moveonly` context:

エクステンションは `moveonly` コンテキストにできる。

```
moveonly extension Array {
  // Element and Array<Element> are not assumed to be copyable here
  // Element と Array<Element> はここではコピー可能と仮定されない
}
```

A type can declare conditional copyability using a conditional
conformance:

型は conditional conformance を使って条件付けされたコピー可能性を宣言できる。

```
moveonly extension Array: Copyable where Element: Copyable {
  ...
}
```

Conformance to `Copyable`, conditional or not, is an
inherent property of a type and must be declared in the
same module that defines the type.  (Or possibly even the
same file.)

`Copyable` を満たすには、条件付きかどうかによらず、
型の固有のプロパティがその型の定義と同じモジュールで宣言されねばならない
(それか同じファイルかもしれない)

A non-`moveonly` extension of a type reintroduces the
copyability assumption for the type and its generic
arguments.  This is necessary in order to allow standard
library types to support non-copyable elements without
breaking compatibility with existing extensions.  If the
type doesn't declare any conformance to `Copyable`, giving
it a non-`moveonly` extension is an error.

型の non-`moveonly` なエクステンションは
その型とそれのジェネリック引数のコピー可能仮定を再導入する。
これは標準ライブラリの型がコピー不可能な型の要素を
既存のエクステンションと互換性を壊さずに対応するために必要となる。
もし型が `Copyable` を満たす宣言をされていなくて、
それに non-`moveonly` エクステンションを与えられていればエラーとなる。

A function can be a `moveonly` context:
関数は `moveonly` コンテキストにできる。

```
extension Array {
  moveonly func report<U>(_ u: U)
}
```

This suppresses the copyability assumption for any new
generic arguments and their hierarchies of associated types.

これは新しいジェネリック引数とそれの associated type のヒエラルキーについて
コピー可能仮定を抑制する。  

A lot of the details of `moveonly` contexts are still up
in the air.  It is likely that we will need substantial
implementation experience before we can really settle on
the right design here.

`moveonly` コンテキストの詳細の多くはいまだ検討中だ。
これの正しい設計を本当に確定させる前に多くの実装実験が必要だろう。

One possibility we're considering is that `moveonly`
contexts will also suppress the implicit copyability
assumption for values of copyable types.  This would
provide an important optimization tool for code that
needs to be very careful about copies.

考えている一つの可能性は
`moveonly` コンテキストはコピー可能な型の暗黙のコピー可能仮定を抑制することだ。
これはコピーについてとても注意する必要があるコードのための
重要な最適化の道具を提供するだろう。

### `deinit` for non-copyable types
### コピー不可能な型の `deinit` 

A value type declared `moveonly` which does not conform
to `Copyable` (even conditionally) may define a `deinit`
method.  `deinit` must be defined in the primary type
definition, not an extension.

`moveonly` と宣言された `Copyable` を満たさない値型は、
`deinit` メソッドを定義するかもしれない。
`deinit` は一級型定義として定義されねばならず、エクステンションにはできない。

`deinit` will be called to destroy the value when it is
no longer required.  This permits non-copyable types to be
used to express the unique ownership of resources.  For
example, here is a simple file-handle type that ensures
that the handle is closed when the value is destroyed:

`deinit` はもう必要とされない値が破棄されるときに呼ばれる。
これはコピー不可能な型がリソースのユニークオーナーシップを表現するのに使うことを許す。
例えば、これは単純なファイルハンドル型だで、
値が破棄されるときハンドルが閉じられることを保証する。

```
moveonly struct File {
  var descriptor: Int32

  init(filename: String) throws {
    descriptor = Darwin.open(filename, O_RDONLY)

    // Abnormally exiting 'init' at any point prevents deinit
    // from being called.
    //
    // どの点でも 'init' を異常脱出するのは deinit が呼ばれるのを防ぐ
    if descriptor == -1 { throw ... }
  }

  deinit {
    _ = Darwin.close(descriptor)
  }

  consuming func close() throws {
    if Darwin.fsync(descriptor) != 0 { throw ... }

    // This is a consuming function, so it has ownership of self.
    // It doesn't consume self in any other way, so it will
    // destroy it when it exits by calling deinit.  deinit
    // will then handle actually closing the descriptor.
    //
    // これは consuming 関数だから、selfのオーナーシップを取る。
    // 他の方法ではselfを消費しないので、
    // これはdeinitを呼んでこれを脱出しこれを破棄する。
    // deinit は実際にデスクリプタを閉じる。
  }
}
```

Swift is permitted to destroy a value (and thus call `deinit`)
at any time between its last use and its formal point of
destruction.  The exact definition of "use" for the purposes
of this definition is not yet fully decided.

Swiftはそれの最後の使用から破棄の形式的な点までの間のどこでも
値を破棄することが許されている。(そして `denit` を呼ぶ)
この定義の目的での正確な「使用」の定義はまだ完全に決定されていない。

If the value type is a `struct`, `self` can only be used
in `deinit` in order to refer to the stored properties
of the type.  The stored properties of `self` are treated
like local `let` constants for the purposes of the
definitive-initialization analysis; that is, they are owned
by the `deinit` and can be moved out of.

もし値型が `struct` なら、
`self` は `deinit` の中でその型の stored property を参照するためにしか使えない。
定義的初期化解析の目的では
`self` の stored propertyは ローカルの `let` 定数のように扱わる。
これは、それらが `deinit` によって保有され、そして取り除かれるからだ。

If the value type is an `enum`, `self` can only be used
in `deinit` as the operand of a `switch`.  Within this
`switch`, any associated values are used to initialize
the corresponding bindings, which take ownership of those
values.  Such a `switch` leaves `self` uninitialized.

もし値型が `enum` ならば、
`self` は `deinit` の中では `switch` のオペランドとしてのみしか使えない。
この `switch` の中で、
あらゆる associated value は対応するバインディングを初期化するのに使われ、
これらはその値のオーナーシップを取る。
この `switch` は `self` を未初期化にする。

### Explicitly-copyable types
### 明示的コピー可能型

Another idea in the area of non-copyable types that we're
exploring is the ability to declare that a type cannot
be implicitly copied.  For example, a very large struct
can formally be copied, but it might be an outsized
impact on performance if it is copied unnecessarily.
Such a type should conform to `Copyable`, and it should
be possible to request a copy with the `copy` function,
but the compiler should diagnose any implicit copies
the same way that it would diagnose copies of a
non-copyable type.

コピー不可能な方の領域の他のアイデアとして、
暗黙にコピーする事ができない型を宣言することを調べている。
例えば、とても大きい値型は形式的にはコピーできるが、
もしそれが不要にコピーされればパフォーマンスに特大な影響を与える。
そのような型は `Copyable` を満たし、
`copy` 関数によってコピーを要求する事が可能だが、
コンパイラはコピー不可能な型をコピーしているのを診断するのと同じように、
どんな暗黙のコピーも診断するべきだ。

## Implementation priorities
## 実装優先度

This document has laid out a large amount of work.
We can summarize it as follows:

この文書は多くの量の作業を示した。
それを以下に要約する。

- Enforcing the Law of Exclusivity:
  
  排他則の強制

  - Static enforcement

    静的強制

  - Dynamic enforcement

    動的強制

  - Optimization of dynamic enforcement

    動的強制の最適化

- New annotations and declarations:

  新しい修飾子と宣言

  - `shared` parameters

    `shared` 引数

  - `consuming` methods

    `consuming` メソッド

  - Local `shared` and `inout` declarations

    ローカルの `shared` と `inout` 宣言

- New intrinsics affecting DI:

  DIに影響する新しい組み込み機能

  - The `move` function and its DI implications

    `move` 関数とそのDI実装

  - The `endScope` function and its DI implications

    `endScope` 関数とそのDI実装

- Co-routine features:

  コルーチン機能

  - Generalized accessors

    一般化されたアクセサ

  - Generators

    ジェネレータ

- Non-copyable types

  コピーできない型

  - Further design work

    将来の設計作業

  - DI enforcement

    DIの強制

  - `moveonly` contexts

    `moveonly` コンテキスト

The single most important goal for the upcoming releases is
ABI stability.  The prioritization and analysis of these
features must center around their impact on the ABI.  With
that in mind, here are the primary ABI considerations:

もうすぐのリリースのための一つの最も重要な目標はABI安定性だ。
優先度とこれらの機能の分析はABIへの影響を中心にするべきだ。
この観点で、ここに一級のABIの考察がある。

The Law of Exclusivity affects the ABI because it
changes the guarantees made for parameters.  We must adopt
this rule before locking down on the ABI, or else we will
get stuck making conservative assumptions forever.
However, the details of how it is enforced do not affect
the ABI unless we choose to offload some of the work to the
runtime, which is not necessary and which can be changed
in future releases.  (As a technical note, the Law
of Exclusivity is likely to have a major impact on the
optimizer; but this is an ordinary project-scheduling
consideration, not an ABI-affecting one.)

排他則はABIに影響を与える、
なぜならこれは引数についての保証を変更するからだ。
ABIを固定するまえにこの規則を適用する必要があるし、
さもなくば永遠に保守的仮定を作るのから抜け出せなくなる。
しかし、これをどのように強制するかの詳細は
いくつかの作業を
必要がなく将来のリリースで変更できる
ランタイムにオフロードする事を選ぶかぎり
ABIに影響を与えない。
(技術的な注意として、
排他則はオプティマイザに主要な影響をあたえるだろう。
しかしこれは普通のプロジェクト計画の考慮で、
ABIに影響をあたえるものではない)

The standard library is likely to enthusiastically adopt
ownership annotations on parameters.  Those annotations will
affect the ABI of those library routines.  Library
developers will need time in order to do this adoption,
but more importantly, they will need some way to validate
that their annotations are useful.  Unfortunately, the best
way to do that validation is to implement non-copyable types,
which are otherwise very low on the priority list.

標準ライブラリは熱狂的にオーナーシップ修飾子を引数に与えるだろう。
これらの修飾子はそれらライブラリルーチンのABIに影響を与える。
ライブラリ開発者はこれを提供するために時間が必要で、
しかしもっと重要なのは、
これらの修飾子が便利なのかを検討する方法が必要な事だ。
残念ながら検討する最も良い方法はコピー不可能な型を実装することだが、
これは優先度リストでは他のものよりとても低い。

The generalized accessors work includes changing the standard
set of "most general" accessors for properties and subscripts
from `get`/`set`/`materializeForSet` to (essentially)
`read`/`set`/`modify`.  This affects the basic ABI of
all polymorphic property and subscript accesses, so it
needs to happen.  However, this ABI change can be done
without actually taking the step of allowing co-routine-style
accessors to be defined in Swift.  The important step is
just ensuring that the ABI we've settled on is good
enough for co-routines in the future.

一般化アクセサの作業は「もっとも一般的な」プロパティとサブスクリプトのアクセサの
標準セットを
`get` / `set` / `materializeForSet` から (本質的に) 
`read` / `set` / `modify` に変更することを含む。
これは全てのポリモーフィックなプロパティとサブスクリプトのアクセスの基本的なABIに
影響を与える、だからそれが起きる必要がある。
しかし、これはSwiftにコルーチンスタイルのアクセサを定義することを可能にする
段階を実際にはとることなしにこのABIを変更することができる。
重要な段階は確定させるABIが将来のコルーチンのために十分であることをただ保証することだ。

The generators work may involve changing the core collections
protocols.  That will certainly affect the ABI.  In contrast
with the generalized accessors, we will absolutely need
to implement generators in order to carry this out.

ジェネレータの作業はコアのコレクションプロトコルへの変更をもたらす。
これもいくらかABIに影響がある。
一般化されたアクセサと対象的に、
ジェネレータを実行できるように絶対に実装する必要がある。

Non-copyable types and algorithms only affect the ABI
inasmuch as they are adopted in the standard library.
If the library is going to extensively adopt them for
standard collections, that needs to happen before we
stabilize the ABI.

コピー不可能な型とアルゴリズムは標準ライブラリでそれが適用されるのと同じ程度だけ
ABIに影響を与える。
ライブラリが標準のコレクションにそれらを広く適用ならば、
ABIを安定させるまえにそれを起こす必要がある。

The new local declarations and intrinsics do not affect the ABI.
(As is often the case, the work with the fewest implications
is also some of the easiest.)

新しいローカル宣言と組み込み機能はABIに絵強を与えない。
(大抵の場合、影響の小さい作業は、いくらか簡単なものだ)

Adopting ownership and non-copyable types in the standard
library is likely to be a lot of work, but will be important
for the usability of non-copyable types.  It would be very
limiting if it was not possible to create an `Array`
of non-copyable types.

オーナーシップとコピー不可能な型を標準ライブラリに適用するのは
大きな作業となるだろう、
しかしコピー不可能な型の使いやすさのために重要だ。
もしコピー不可能な型の `Array` を作ることができなければこれはとても制限されるだろう。
