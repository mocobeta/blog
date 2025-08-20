+++
title = "RustのTraitとTrait objectとGenerics - Programming Rust, 2nd Editionから"
date = "2025-08-20"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "rust"]
+++

[Programming Rust, 2nd Edition](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) の11章"Traits and Generics"からのTIL。

## Trait objectとGenericsの違い

Trait objectとGenerics（型パラメータ）は，どちらもTraitを活用したポリモーフィズムを提供しますが，使いどころは異なります。

例として，麻雀の面子を表すTraitと，その実装を考えます。ちなみに，この例のように，RustのidentifierにはJavaやPythonと同じくマルチバイト文字も使えます。メインコードでは原則非推奨としたいですが，英語で説明しがたい特殊な状況を再現するためのテストメソッド名などに便利そう。

```rust
trait 面子 {}

// コーツ
struct 刻子 {}
impl 面子 for 刻子 {}

// カンツ
struct 槓子 {}
impl 面子 for 槓子 {}

// シュンツ
struct 顺子 {}
impl 面子 for 顺子 {}

// トイツ
struct 対子 {}
impl 面子 for 対子 {}

// 残りは略
```

手牌は面子の集合からなりますが，ここでTrait objectを使って，こう書けます。`dyn 面子` がTrait objectです。Trait objectは，コンパイル時にはどの型が入るか決まっておらずサイズが不定なので，`Box`でラップして使います。

```rust
struct 手牌 {
    面子: Vec<Box<dyn 面子>>,
}
```

ここで，Generics（型パラメータ）でこう書くのは間違い，というよりは，シンタックス上は間違いではないけれど，麻雀の手牌を表現できません。なぜなら，`T`はコンパイル時に具体的な型が決まるため，`Vec<T>`はすべて同じタイプの面子になってしまうから。

```rust
struct 手牌<T: 面子> {
    面子: Vec<T>,
}
```

つまり，

- Trait object (`dyn Trait`) は動的に，実行時に型が決まる
  - 実行するべきメソッドをルックアップするための実行時オーバーヘッドがある
  - バイナリサイズは小さくなる（具体的な型ごとのコードは生成されないため）
  - 実行時の柔軟性がある
- Generics (`<T: Trait>`) は静的に，コンパイル時に型が決まる
  - 実行時オーバーヘッドがない
  - バイナリサイズは大きくなりがち（具体的な型ごとにコードが生成されるため）
  - 実行時の柔軟性はない

という違いです。わかってしまえば当たり前，というような話ですが，わかっていなかったのでメモ。

Trait objectが，Javaのインタフェースの概念に近しく，オブジェクト指向に慣れていれば親しみやすい一方で，Generics（型パラメータ）でしか使えないTraitの機能もあります。

## Associated functions

Generics，つまり型パラメータでないと使えないTraitの機能の代表的な例が associated functionsです。その特徴を箇条書きにすると

- Traitも構造体と同じく，associated functionsを定義できる
- *T::associated_function()* という形で，型パラメータで呼び出せる

Java 8でインタフェースがstaticメソッドを持てるようになりましたが，そのようなイメージで合っていると思います（多分）。

associated functionsと同じように，Traitで定義して，型パラメータと組み合わせて使える機能として associated types と associated consts があります。

## Trait bounds

その他，Genericsでしか使えない機能に，型パラメータの境界を指定する trait bounds があります。

これは書ける。

```rust
let x: T: Debug + Hash + Eq;
```

こうは書けない。

```rust
// let x: &dyn Debug + Hash + Eq;
```

## Generic traitsとデフォルト型パラメータ

このあたりから，複雑さが増して認知負荷が高くなってくるため，そういうのもあるのね，というくらいのメモ。

Rustの数値演算子は，Generic trait，つまり型パラメータを定義に含むtraitでこのように定義されています。たとえば掛け算 [Mul](https://doc.rust-lang.org/std/ops/trait.Mul.html) (`*`) の例。

```rust
pub trait Mul<RHS = Self> {
    type Output;

    fn mul(self, rhs: RHS) -> Self::Output;
}
```

`rhs`の型パラメータ`RHS`はRight Hand Sideの略とのことです。

- `mul` の戻り値の型は associated type の `Output`
- self, つまりLeft Hand Sideの型と`rhs`の型とOutputの型は，すべて異なっていても良い
- `rhs`のデフォルトの型はSelf
