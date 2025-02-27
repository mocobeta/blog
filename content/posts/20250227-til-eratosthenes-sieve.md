+++
title = "素数判定法エラトステネスの篩の最適化"
date = "2025-02-27"

[taxonomies]
categories = ["Posts"]
tags = ["til", "algorithm"]
+++

最近，数学とプログラミングの練習のために，[Project Euler](https://projecteuler.net/archives)をRustで解いている。正答したあとに見られる解説が丁寧なので，とりあえず思いつくナイーブな方法で解いあと，解説を読んで最適解を実装してみる，という感じでゆるゆると進めている。

初級レベルの問題で素数判定を使う問題が出てくるのだけど，よく知られていて実装が簡易な素数判定アルゴリズム[エラトステネスの篩](https://ja.wikipedia.org/wiki/%E3%82%A8%E3%83%A9%E3%83%88%E3%82%B9%E3%83%86%E3%83%8D%E3%82%B9%E3%81%AE%E7%AF%A9)の最適化の解説を見つけて，面白かったので書き残しておく。

基本的なアイディアは，「2以外の偶数は素数になりえないから，奇数だけの配列を持つ」というもので，つまり配列の`i`番目の要素は，`2*i+1`を意味する。配列の要素数がオリジナルのアルゴリズムの半分で済み，大きな数を扱う時にメモリ使用量とキャッシュミスが減る，というメリットがある。

アイディア自体は直感的ですぐ理解できるのだけど，これを実装に落とし込むにあたってはインデックスの計算がトリッキーで，紙に書いて理解して実装するのに私は2時間くらいかかった（こういうのがぱぱっと実装できてしまう人が，プログラミングが得意な人なんだろう）。

テスト済みのRust実装は以下。素数列を生成したかったので，Iteratorとして作った。

コードを見ても何をしているのかわかりづらいと思うので，興味がある人はProject Eulerの[Problem 10](https://projecteuler.net/problem=10)を解いて解説を読んでほしい（解説自体の配布は禁止されているので注意）。


```rust
pub struct EratosthenesSieve {
    limit: u64,
    sieve: Vec<bool>,
    ptr: usize,
}

impl EratosthenesSieve {
    pub fn new(limit: u64) -> Self {
        let sieve_size = (limit as usize) / 2;  // 配列の要素数は，limitの半分
        let mut sieve = EratosthenesSieve {
            limit,
            sieve: vec![true; sieve_size],
            ptr: 0,
        };
        sieve.initialize();
        sieve
    }

    // 配列のi番目の要素について，2*i+1が素数でなければfalseに更新する関数
    fn initialize(&mut self) {
        let stop = (((self.limit as f64).sqrt()) / 2.0 + 1.0) as usize;
        for i in 1..stop {
            if self.sieve[i] == true {
                for j in (2 * i * (i + 1)..self.sieve.len()).step_by(2 * i + 1) {
                    self.sieve[j] = false;
                }
            }
        }
    }
}

impl Iterator for EratosthenesSieve {
    type Item = u64;

    fn next(&mut self) -> Option<Self::Item> {
        if self.ptr == 0 {
            self.ptr += 1;
            Some(2)  // 最初の素数は1ではなく2なので，ちょっとハックを入れる
        } else if self.ptr >= self.sieve.len() {
            None
        } else {
            while self.ptr < self.sieve.len() && self.sieve[self.ptr] == false {
                self.ptr += 1;
            }
            if self.ptr < self.sieve.len() {
                let next_prime = (2 * self.ptr + 1) as u64;
                self.ptr += 1;
                Some(next_prime)
            } else {
                None
            }
        }
    }
}
```

