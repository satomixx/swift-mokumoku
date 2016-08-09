# Writing High-Performance Swift Code
- [最適化を可能にする](#最適化を可能にする)
- [モジュール全体の最適化](#モジュール全体の最適化)
- [Dynamic Dispatchの削減](#dynamic-dispatchの削減)
- [コンテナの型を効率的に使う](#コンテナの型を効率的に使う)
- [Unchecked Operation](#unchecked-operation)
- [Generics](#generics)
- [The cost of large Swift values](#the-cost-of-large-swift-values)
- [Unsafe code](#unsafe-code)
- [Protocols](#protocols)

## 最適化を可能にする
最適化を可能にすることは常に第一にやるべきことです。Swiftに3つの異なる最適化のレベルがあります。
- `-0none`: これは通常の開発を意味します。最小限の最適化をし、すべてのデバッグ情報を保持します。
- `-0`: これはほぼ本番用のコードを意味します。コンパイラは積極的な最適化をし型やコード量を劇的に変更します。デバッグ情報は放出され消えます。
- `-0unchecked`: これは特別な最適化モードで特定のライブラリやパフォーマンスが不安定になることを厭わないアプリケーションに用いられます。コンパイラは暗黙の型チェックと同様にオーバーフローチェックをすべて削除します。これは、メモリーの安全性やintegerのオーバーフローを検知できないかもしれない為、一般的に使ってもらいたいものではありません。唯一のユースケースは、integerのオーバーフローや型変換が充分に安全であるといえるほど、コードを注意深くレビューした場合です。

XcodeのUIでは、現在の最適化レベルは次の用に変更できます。

## モジュール全体の最適化
デフォルトでは、Swiftはそれぞれのファイルをここにコンパイルします。これにより、Xcodeに複数のファイルを並列にとても素早くコンパイルさせることが出来ます。しかし、夫々のファイルを別々にコンパイルすることで、あるコンパイラの最適化を妨げます。Swiftはプログラム全体を、まるで1つのファイルかの様にコンパイル出来、まるでSingle Comiplation Unitの様にプログラムを最適化できます。このモードは `swiftc` コマンドを `-whole-module-optimization` フラグと使って可能になります。このモードでコンパイルされたプログラムはコンパイルに時間がかかる様に思えますが、とてもも高速で動かせます。

このモードは、Xcodeの Build Setting で `Whole Module Optimization` を使うことで可能になります。

## Dynamic Dispatchの削減
- デフォルトでは、SwiftはObjective-Cのようにとても動的な言語です。Objective-Cと違うのは、Swwftはプログラマーに、必要に応じて動的性を消したり削減することでランタイムのパフォーマンスを改善する能力wを提供することです。このセクションでは、そのようなオペレーションに使うことが出来る言語設計のいくつかの例を見れいきます。

### Dynamic Dispatch
クラスはデフォルトでメソッドやプロパティにアクセスする Dynamic Dispatch を使用します。それ故次のコードスニペットでは、 `a.aProperty`, `a.doSomething()`, そして `a.doSomethingElse()` は Dynamic Dispatchを通じて呼びだされます。

```swift
class A {
  var aProperty: [Int]
  func doSomething() { ... }
  dynamic doSomethingElse() { ... }
}

class B : A {
  override var aProperty {
    get { ... }
    set{ ... }
  }

  override func doSomething() { ... }
}

func usingAnA(_ a: A) {
  a.doSomething()
  a.aProperty = ...
}
```

Swiftでは、Dynamic Dispatch は vtable[[注1]](#脚注) を通じた間接的な呼び出しをデフォルトに持ちます。もし一方が `dynamic` キーワード宣言を持てば、Swiftは Objective-C を通じて代わりに送られたメッセージの呼び出しが出来ます。どちらの場合にも、間接的に自信を呼び出すオーバーヘッドを追加することを多くのコンパイラの最適化[[注2]](#脚注)から妨げるため、直接的な関数のコールより遅くなります。パフォーマンスに致命的なコードでは、動的な振る舞いは制限を求められる様になります。

## アドバイス: 宣言がoverrideされる必要がないときは 'final' を使いましょう
`final` というキーワードは、クラスやメソッド、プロパティの宣言をoverrideされないという制限を加えます。これにより、コンパイラは、間接的な呼び出しの代わりに直接関数を呼び出すことができます。例えば、次の `C.array1`, `D.array1` は直接アクセスされます[[注3]](#脚注)。対して、 `D.array2` はvtableを通じて呼ばれます。

```swift
final class C {
  // No declarations in class 'C' can be overridden
  var array1: [Int]
  func doSomething() { ... }
}

class D {
  final var array1: [Int] // 'array1' cannot be overridden by a computed property.
  var array2: [Int] // 'array2' *can* be overridden by a computed property.
}

funv usingC(_ c: C) {
  c.array1[i] = ... // Can directly access C.array without going through dynamic dispatch.
  c.doSomething() = ... // Can directly call C.doSomething without going through virtual dispatch.
}

func usingD(_ d: D) {
  d.array1[i] = ... // Can directly access D.array1 without going through dymanic dispatch.
  d.array2[i] = ... // Will access D.array2 through dynamic dispatch.
}
```

## アドバイス: 宣言がfileの外からアクセスされる必要がなければ、'private' や 'fileprivate' を使いましょう
`private` や  `fileprivate` というキーワードを宣言に対して適用させた場合、それを宣言したfileの中でのみ見えるように制限されます。これにより、コンパイラは、潜在的にoverrideするすべての他の宣言を解析することが出来ます。それゆえ、そのような宣言が存在しない場合には、コンパイラは `final` キーワードを自動的に推察して、メソッドやフィールドへの直接的な呼び出しを順次削除出来る様になります。例えば次のような、 `e.doSomething()` と `f.myPrivateVar` は直接アクセスされることが出来、 `E` `F` が同じfile内でoverrideする宣言を持たないと推察されます。

```swift
private class E {
  func doSomething() { ... }
}

class F {
  fileprivate var myPrivateVar: Int
}

func usingE(_ e: E) {
  e.doSomething() // There is no sub class in the file that declares this class.
                  // The compiler can remove virtual calls to doSomething()
                  // and directly call A's doSomething method.
}

func usingF(_ f: F) {
  return f.myPrivateVar
}
```

# コンテナの型を効率的に使う
Swiftの標準ライブラリで提供される重要な機能に、ArrayやDictionaryのGenericコンテナがあります。このセクションでは、これらの型を効果的に使う方法を説明します。

## アドバイス: ArrayではValue Typesを使いましょう
Swiftでは、方は2つのカテゴリに分けられます。1つはValue Types(structs, enum, tuples) そしてもう一つは、Reference Types(classes)です。これらを区別するキーポイントは、Value TypesはNSArrayの中に含まれることは出来ないということです。それゆえ、Value Typesを使うときは、NSArrayに裏付けされたarrayの可能性を操作する必要があるArrayでのオーバーヘッドのほとんどを、Optimizerが削除できるのです。
更に、Reference Typeと較べて、Value TypesはReference Typesを再帰的に保有するなら、参照の数のみが必要になります。Reference TypesなしにValue Typesを使うことで、Arrayの中でトラフィックを保持した開放したりする追加的な処理をしなくてもよいのです。

```swift
// Don not use a class here.
struct PhonebookEntry {
  var name : String
  var number : [Int]
}

var a : [PhonebookEntry]
```

大きなValue Typesの使用とReference Typesの使用はトレードオフの関係にあることを心に止めてください。大きなValue Typesをコピーしたり移動させるオーバーヘッドは、そのブリッジングを削除することととオーバーヘッドを保持・解放することのコストをこえるでしょう。

## アドバイス: NSArray bridgingが不必要な際には、references typesとともにCOntiguousArrayを使いましょう

もしreference typesのarrayが必要で、そのarrayがNSArrauへのbridgingを必要としないなら、ArrayではなくContiguousArrayを使いましょう。

```swift
class C { ... }
var a: ContiguousArray<C> = [C(...), ..., C(...)]
```

## アドバイス: object-reassignmentの代わりに、配置済みのmutationを使いましょう
すべてのSwiftの標準ライブラリコンテナは、COW(copy-on-write)を正確なコピーの代わりに用いるvalue typesです。多くの場合、これによりコンパイラはdeep copyをする代わりにコンテナを保つことで不必要なコピーを省略できます。これは、もし、コンテナのreference count数が1より大きく、コンテナがmutatedなら、下層コンテナをただコピーすることでなされます。例えば、次の様に、 `d` が  `c` にアサインされるときにコピーは起きませんが、 `2` がたされることとで `d` はstructural mutationを行うとき、 `d` はコピーされ `2` は `d` にたされます。

```swift
var c: [Int] = [ ... ]
var d = c // No copu will occur here.
d.append(2) // A copy *does* occur here.
```

もしユーザーが注意深くなければ、時々COWは予期しないコピーをします。例えば、関数のobject-reassignmentを通じたmutationを試みます。Swiftでは、すべてのパラメーターは `+1` でわたされます。パラメーターはcallsiteの前に保持され、そして呼びだされたものが終了するときに解放されます。之は、次の様に関数を書くことを意味します。

```swift
func append_one(_ a: [Int]) -> [int] {
  a.append(1)
  return a
}

var a = [1, 2, 3]
a = append_one(a)
```

`a` は `a` のバージョンによらずコピーされ、assignmentにより `append_one` の後に使われることはない。これは、 `inout` パラメーターの使用をすることで避けられます。

```swift
func append_one_in_place(a: input [Int]) {
  a.append(1)
}

var a = [1, 2, 3]
append_one_in_place(&a)
```
# Unchecked Operation
Swiftは、標準の計算を行う際は、overflowのチェックをすることで、integer overflowのバグを排除します。これらのチェックは、ハイパフォーマンスを求めるなら適切ではありません。

## アドバイス: overflowが起きないと証明出来るときは、unchecked integer arithmeticを使いましょう

パフォーマンスを最重要とするコードでは、安全だと言える場合にはoverflowのチェックを省略します。

```swift
a: [Int]
b: [Int]
c: [Int]

// Precondition: for all a[i], b[i]: [i] + b[i] does not overflow!
for i in 0 ... n {
  c[i] = a[i] &+ b[i]
}
```

# Generics
SwiftはGeneric Typesを使うことでとても強力な抽象化メカニズムを提供します。Swiftコンパイラは どの `T` に対して働くことができる `MySwiftFunc<T>` 具体的なコードのブロックを放出します。生成されたコードはfunction pointersのテーブルと追加パラメータとしての `T` を持つboxを持ちます。 `MySwiftFunc<Int>` と `MySwiftFunc<String>` の振る舞いの違いは、異なるfunction pointersテーブルをパスすることや、boxにより提供される抽象化の規模で算出されます。以下Genericの例です。

```swift
class MySwiftFunc<T> { ... }

MySwiftFunc<Int> X // Will emit code that works with Int...
MySwiftFunc<String> Y // ... as well as String
```

最適化が可能なとき、Swiftコンパイラはそのようなコードの起動を見て、起動時に使われる non-generic type の様な具体性を突き止めようとします。もし、generic functionの定義がoptimizerによって見ることが出来、concrete typeが知られたら、Swiftコンパイラは特定の型に特別化されたgeneric functionのバージョンを放出します。このプロセスは *specialization* と呼ばれ、genericsに関連づいたオーバーヘッドを削除することが出来ます。

```swift
class MyStack<T> {
  func push(_ element: T) { ... }
  func pop() -> T { ... }
}

func myAlgorithm(_ a: [T], length: Int) { ... }

// The compiler can specialize code of MyStack<Int>
var stackOfInts: MyStack<Int>
// Use stack of Int.
for i in ... {
  stack.push(...)
  stack.pop(...)
}

var arrayOfInts: [Int]
// The compiler can emit a specialized version of 'myAlgorithmn' targeted for 
// [Int] types.
myAlgorithm(arrayOfInts, arrayOfInts.length)
```

## アドバイス: Genericの宣言は、使用されているのと同じモジュールに置くこと
Optimizerは、generic declarationの定義が現在のモジュール内で可視的である場合にはspecializationをすることが出来ます。これは、もし宣言がgenericの起動ファイルと同じfileにある場合で、 `-whole-module-optimization` フラグが使われている場合に発生します。標準ライブラリは特別です。標準ライブラリでの定義はすべてのモジュールに可視的で、specializationが可能です。

## アドバイス:  Genericをspecializeするコンパイラに @_specialize を用いること
もし呼ぶ場所と呼ばれた関数が同じモジュールにあるなら、コンパイラはただただ自動的にgenericコードをspecializeします。しかし、プログラマは @_specialize 属性のフォーム内でコンパイラにヒントを与えることが出来ます。例えば[これをみてください。](https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst#id6)

この属性はコンパイラに特定の具体的な型リストをspecializeするようにコンパイラに指示します。コンパイラは型チェックを行いgeneric functionから特定の変数に送ります。次の例では、@_specialize 属性を注入したことで、コードが10倍早くなります。

```swift
/// ---------------
/// Framework.swift

public protocol Pingable { func ping() -> Self }
public protocol Playable { func play() }

extension Int : Pingable {
    public func ping() -> Int { return self + 1 }
}

public class Game<T : Pingable> : Playable {
    var t : T

    public init (_ v : T) {t = v}

    @_specialize(Int)
    public func play() {
      for _ in 0...100_000_000 { t = t.ping() }
    }
}

/// -----------------
/// Application.swift

Game(10).play
```


# The cost of large Swift values
Swiftでは、値はデータのユニークなコピーを保持します。値が特別な状態を保証するような、value-typesを使う幾つかのアドバンテージがあります。値をコピーするとき（assignmentの影響やinitialization、引数の引き渡し）、プログラムは新しい値のコピーを作ります。大きな値の場合は、これらのコピーには時間がかかり、プログラムのパフォーマンスにダメージを与えます。

"value"ノードを使ったTreeを定義した以下の例を考えてみてください。Treeノードはprotocolを使う他のノードを持ちます。CGでは、シーンは、値として表現できる異なる権限や変異形から構成されたりします。なのでこの例は現実的だったりします。

```swift
protocol P {}
struct Node : P {
    var left, right : P?
}

struct Tree {
    var node : P?
      init() { ... }
}
```

Treeがコピーされた時、Tree全体がコピーされます。わたしたちのTreeの場合は、たくさんのmalloc/allocを呼び出すことが必要なとても高価なオペレーションであり、膨大なReference Countオーバーヘッドになります。

しかし、もし値が値が持つセマンティクスと同じくらい長くメモリーにコピーされるなら、気にする必要ありません。

## アドバイス: 大きな値にはcopy-on-writeを使うこと
大きな値をコピーするコストを排除するには、copy-on-writeを採用しましょう。copy-on-writeの最も簡単な実装はArrayの様に、存在するcopy-on-write構造を構成することです。Swift arraysは常に値ですが、arrayの中身は、copy-on-writeのtraitの為、arrayが引数として渡される度にコピーされるわけではありません。

Treeの例で、array内で、ラッピングによりTreeの中身をコピーするコストを排除します。この単純な変更は、Treeデータ構造のパフォーマンスに多大な影響を持ち、O(1)へのTreeのサイズにより、O(n)から落ちた引数としてのarrayを渡すコストにも大きな影響を持ちます。

```swift
struct Tree : P {
  var note : [P?]
  init() {
    node = [thing]
  }
}
```

COWセマンティクスのArrayを使うことのディスアドバンテージは明確に2つあります。ひとつ目は、Arrayが、value wrapperの混んてキスので意味を成さない "count"や"append"の様なメソッドをむき出しにすることです。これらのメソッドはReference wrapperを扱いにくくします。使わないAPIを隠す構造体のwrapperを作成することでこの問題に働きかけることができ、Optimizerはこのオーバーヘッドを削除しますが、wrapperは次の問題を解決しません。2つめの問題は、Arrayはprogram safetyを求めるコードを持ち、Objective-Cと相互作用します。Swiftは、indexされたアクセスがarray boundsとともに落ちるかチェックます。それはarray storageが拡張が必要なら値を保存するときに行われます。これらのランタイムチェックは物事を遅くします。

Arrayを使う代わりにvalue wrapperとしてArrayに変わるcopy-on-write構造を実装することがあります。

```swift
final class Ref<T> {
  var val: T
  init(_ v: T) {val = v}
}

struct Box<T> {
  var ref : Ref<T>
  init(_x : T) { ref = Ref(x) }

  var value: T {
    get { return ref.value }
    set {
      if !isUniquelyReferenceNonObjC(&Ref) {
        ref = Ref(newValu)
        return
      }
      ref.val = newValue
    }
  }
}
```

この `Box` はArrayに置き換えられます。

# Unsafe code
Swiftのクラスは常にReferenceをカウントされます。Swiftのコンパイラはオブジェクトがアクセスされる度にReference Countを増加させます。例えば、クラスを使って実装されたLkinked listをスキャンする問題を考えてみてください。リストをスキャンすることは、あるノードから `elem = elem.next` へリファレンスを移動されることでなされます。私たちがReferenceを移動させる度に、Swiftは `next` オブジェクトへのReference Countを増加させ、以前のオブジェクトへの Reference Countを減少させます。これらのReference Count Operationはとても高価なものであり、Swiftクラスを使う際には、避けることができません。

```swift
final class Node {
  var next: Node?
  var data: Int
}
```

## アドバイス: Reference Countのオーバーヘッドを避ける為に Unmanaged Referenceを用いること
`Unmanaged<T>`._withUnsafeGuaranteeRef` はPublic APIではなく、将来的になくなることに留意してください。それ故に、将来的に変更しないコードには使わないでください。

Performance-criticalなコードでは、unmanaged referencesを使うことが選択肢になります。 `Unmanaged<T>` の構造は開発者に特定のReferenceへの自動的なReference Countを不能にすることを許可します。

これをするとき、 インスタンスを行きた状態をキープする `Unmanaged` を使用する間は `Unmanaged` 構成のインスタンスによるインスタンスへの別のReferenceが存在していることを確認する必要があります。（詳細は[こちら Unmanaged.swift](https://github.com/apple/swift/blob/master/stdlib/public/core/Unmanaged.swift)）

```swift
// The call to ``withExtendedLifetime(Head)`` makes sure that the lifetime of
// Head is guaranteed to extend over the region of code that uses Unmanaged
// references. Because there exists a reference to Head for the duration
// of the scope and we don't modify the list of ``Node``s there also exist a
// reference through the chain of ``Head.next``, ``Head.next.next``, ...
// instances.

withExtendedLifetime(Head) {

  // Create an Unmanaged reference.
  var Ref : Unmanaged<Node> = Unmanaged.passUnretained(Head)

  // Use the unmanaged reference in a call/variable access. The use of
  // _withUnsafeGuaranteedRef allows the compiler to remove the ultimate
  // retain/release across the call/access.

  while let Next = Ref._withUnsafeGuaranteedRef { $0.next } {
    ...
    Ref = Unmanaged.passUnretained(Next)
  }
}
```

# Protocols
## アドバイス: プロトコルは、class-protocolsのようなclassesによってのみ満たされると注意しましょう

Swiftはclassのprotocol adoptionにのみ制限されます。class-onlyのprotocolのマークのアドバンテージは、classだけがprotocolを満たすことを知ることを基礎に持つプログラムを、コンパイラは最適化出来ます。例えば、ARC memory management systemは、クラスにまつわることと認識していれば、簡単に保持（objectの参照数を増加）できます。この認識がないと、コンパイラは、structがprotocolを満たすかもしれないと踏みかねず、小さくなくとてもコストが掛かる構造を保持、解放する必要がでてきます。

もし、classのprotocolの採用制限を知っていれば、cloass-only protocolとしてのmark protocolsはruntimeでよりハイパフォーマンスになります。

```swift
protocol Pingable : class { func ping() -> Int }
```



# 脚注
- 1. vtable: 仮想メソッドテーブルもしくは `vtable` は型特定テーブルで型メソッドのアドレスを持つインスタンスに関連されます。Dynamic Dispatchはまずオブジェクトからテーブルを検索することで進行し、次にテーブルのメソッドを検索します。
- 2. これはコンパイラがどの関数を呼び出すかしらないことによります。
- 3. i.e. クラスのフィールドの直接的なロードや、関数の直接的な呼び出し
