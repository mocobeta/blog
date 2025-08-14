+++
title = "Rustのinterior mutabilityとCellとRefCell - Programming Rust, 2nd Editionから"
date = "2025-08-14"

[taxonomies]
categories = ["Short Posts"]
tags = ["til", "rust"]
+++

[Programming Rust, 2nd Edition](https://www.oreilly.com/library/view/programming-rust-2nd/9781492052586/) という本(*1)を少しずつ読んでいます。総ページ数は700ページ超えと，紙の本だと軽く鈍器みがありますが，Rustの基礎がプレーンな英語で書かれていて読みやすい。言うなれば，公式の[The Rust Book](https://doc.rust-lang.org/book/)を10倍に噛み砕いて，初学者向けに構成を練り直したという感じ。普段Rustはそこそこ書いてるけれど，雰囲気で使っているなあという人が少し腰を据えて学び直すにも良さそうです。

*1) 邦訳も出ています: [プログラミングRust 第2版](https://www.oreilly.co.jp/books/9784873119786/)


本記事は，その中で9章のInterior Mutabilityについてのメモです。

- Interior mutability（内部可変性）とは，全体はimmutable（不変）データでありつつ，その一部をmutable（可変）にすること。
- Rustの標準ライブラリでは，`Cell<T>`と`RefCell<T>`の2つの型をinterior mutabilityのために提供している。
  - ただし，`Cell`と`RefCell`はスレッドセーフでないため，マルチスレッドプログラムでは使用できない。

というのがinterior mutabilityと`Cell`と`RefCell`のおそらくよくある大まかな定義（Rust Bookでもだいたいこんな感じで説明されていると思う）で，定義だけだとまあよくわからない。わかるようでピンときていなかったinterior mutabilityが，この本の説明で「あああ，なるほど完全理解」と思えたのが収穫でした。

以下は，interior mutabilityを実例で理解するために自分で書いたコードです。本のサンプルコードとは別物です。

```rust
use std::cell::{Cell, RefCell};

// 運転免許証を表す構造体
// フィールドとして番号，生年月日，改定番号，氏名，住所をもつ。
// `DriverLicense` は基本的に不変でありながら，`revision`, `name`, `address` の各フィールドは内部可変性を持つ。
#[derive(Debug, Clone, PartialEq)]
struct DriverLicense {
    number: String,
    birthday: String,

    // `u32`のように，`Copy`を実装しているシンプルな型は，`Cell<T>`でラップする。
    // `Cell<T>`はラップした値の所有権を持つ。
    revision: Cell<u32>,

    // `String`や構造体のように，複雑な型は`RefCell<T>`でラップする。
    // `RefCell<T>`は`Box<T>`や`Rc<T>`といったスマートポインタの一種。
    name: RefCell<String>,
    address: RefCell<Address>,
}

/// 住所を表す構造体
#[derive(Debug, Clone, PartialEq)]
struct Address {
    zip: String,
    prefecture: String,
    city: String,
}
```

```rust
    // DriverLicenseのインスタンスを生成
    // `license`は（基本的に）不変なので，`mut` でないことに注意。
    let license = DriverLicense {
        number: "12345".to_string(),
        birthday: "1990-08-01".to_string(),
        revision: Cell::new(1),
        name: RefCell::new("山田　さくら".to_string()),
        address: RefCell::new(Address {
            zip: "123-4567".to_string(),
            prefecture: "東京都".to_string(),
            city: "立川市".to_string(),
        }),
    };
```

`revision`を変更する

```rust
    // Cell::set() で値を変更
    license.revision.set(license.revision.get() + 1);
    println!("Revision: {}", license.revision.get());
```

`name` を変更する

```rust
    // RefCell::replace() で値を置換
    license.name.replace("田中　さくら".to_string());
    println!("Name: {}", license.name.borrow());
```

`address` を変更する

```rust
    // RefCell::borrow_mut() で可変参照を取得して値を変更
    let mut addr = license.address.borrow_mut();
    addr.zip = "765-4321".to_string();
    addr.prefecture = "神奈川県".to_string();
    addr.city = "横浜市".to_string();

    // panic! since `addr` is already borrowed mutably
    println!("Address: {:?}", license.address.borrow());
```

このコードはぱっと見，動きそうですが，最後の行でpanicします。borrow ruleに違反しているため。

`RefCell::borrow()`や`RefCell::borrow_mut()`のborrow ruleは，コンパイル時ではなく実行時にチェックされるため，うっかりpanicするコードを書いてしまうことがあります。

ということで，`RefCell`を使う場合は`try_borrow()`や`try_borrow_mut()`を使って，panicを起こす代わりにエラーを処理するほうが良さそうです。

```rust
    if let Ok(mut addr) = license.address.try_borrow_mut() {
        addr.zip = "765-4321".to_string();
        addr.prefecture = "神奈川県".to_string();
        addr.city = "横浜市".to_string();
    }

    if let Ok(addr) = license.address.try_borrow() {
        assert_eq!(
            *addr,
            Address {
                zip: "765-4321".to_string(),
                prefecture: "神奈川県".to_string(),
                city: "横浜市".to_string()
            }
        );
    }
```
