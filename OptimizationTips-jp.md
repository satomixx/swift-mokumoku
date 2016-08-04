# Writing High-Performance Swift Code
- [最適化を可能にする](#最適化を可能にする)
- [モジュール全体の最適化](#モジュール全体の最適化)
- [Dynamic Dispatchの削減]()

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


# 脚注
- 1. vtable: 仮想メソッドテーブルもしくは `vtable` は型特定テーブルで型メソッドのアドレスを持つインスタンスに関連されます。Dynamic Dispatchはまずオブジェクトからテーブルを検索することで進行し、次にテーブルのメソッドを検索します。
- 2. これはコンパイラがどの関数を呼び出すかしらないことによります。
- 3. i.e. クラスのフィールドの直接的なロードや、関数の直接的な呼び出し
