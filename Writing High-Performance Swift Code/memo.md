https://github.com/apple/swift/blob/master/docs/OptimizationTips.rst

# 学んだこと
## Optimizerについて
- 最適化し過ぎはよくない。開発用には `-0none`, 本番用には `-0` を使えばいいと思う

## Dynamic Dispatchの削減
- Dynamic Dispatchは動作を遅くする
- 宣言がoverrideされる必要がないときは `final` を使う
- file外からのアクセスがないなら、 `private` や `fileprivate` を使う

## 型を適切に使う
- Arrayでは Value Types(strucs, enum, tuples)を使うといい
- 大きな値のValue Typesは動作を重くするので、Reference Typesを使ったほうがいい

大きなValue Typesの使用とReference Typesの使用はトレードオフの関係にあることを

# 参考
- [[Swift]動的ディスパッチを減らすことでパフォーマンスを改善](http://blog.andgenie.jp/articles/843)
