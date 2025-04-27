+++
title = "uv workspacesでスッキリ作るPythonモノレポ"
date = "2025-04-27"

[taxonomies]
categories = ["Posts"]
tags = ["entry", "python", "uv"]
+++

# この記事で言いたいこと

uvはいいぞ。Pythonでモノレポするならuv workspacesを使おう！

# イントロダクション

Python界でデファクトになりつつある新興パッケージ／プロジェクト管理ツールの[uv](https://docs.astral.sh/uv/)。

uvには，[workspaces](https://docs.astral.sh/uv/concepts/projects/workspaces/)という，マルチパッケージをサポートするための機能がある。[Cargoにある同名の機能](https://doc.rust-lang.org/cargo/reference/workspaces.html)からインスパイアされたもので，Rustを使っている人には馴染み深いコンセプトだろう。

これまで，Pythonでモノレポ構成を作るための，これといって決め手となるソリューションはなかったように思う。Bazelは小規模プロジェクトで使うには正直とっつきづらく，Poetryでも頑張ればできると思うのだけれど，Poetryそのものはマルチパッケージ構成をサポートしていないためdependency groupを駆使するなどしなければならず，一筋縄ではいかない。

uvのworkspacesならRust(Cargo)のようにスッキリとモノレポ管理ができるのでは，と思って試したところ，かなりいい感じにできそうだったので，少し詳細に手順と構成をまとめておく。

# 要件（実現したいこと）

ひとくちにモノレポといっても，その定義は開発チームが実現したいことによってさまざま。この記事で私が実現したい主要な要件は以下の通り。

- 1つのリポジトリで複数のPythonパッケージ（つまり`pyproject.toml`）を管理したい
- Pythonバージョンや，`ruff`, `pyright`などの開発ツールキットはプロジェクトルートの`pyproject.toml`で統一したい
- 各サブパッケージで個別に依存ライブラリを管理したい
- サブパッケージ間で依存関係を持たせたい

# サンプルコード

workspacesの説明をするためのtoy projectを用意した。

コードは [mocobeta/uv-workspaces-eggdishes](https://github.com/mocobeta/uv-workspaces-eggdishes) に置いてある。

## 機能とパッケージ構成

サンプルコードの機能と構成はこんな感じ。

- 卵料理のレシピを教えてくれるCLIツール
- 拡張性のため，複数の卵料理のレシピと，CLIアプリケーションを個別パッケージで管理する
  - コアライブラリパッケージ(`eggdishes-core`)にベースクラス`EggDish`とそのデフォルト実装`FreshEgg`を配置
  - いくつかの拡張ライブラリパッケージ(`eggdishes-*`)に`EggDish`のサブクラス（例: `BoiledEgg`）を配置して，コアライブラリとは独立して管理する
  - アプリケーションパッケージ(`eggdishes-main`)にCLIのコードを配置
- アプリケーションパッケージから，コアライブラリとその拡張ライブラリのクラスを呼べる

toy projectなのでかなり簡略化しているけれど，現実のプロジェクトでもよくある構成を想定している。

# Getting Started

早速リポジトリを作っていく。最初にコアライブラリとアプリケーションの2つのサブパッケージを用意して，workspacesの基本的な使い方を確認してから，そのあとで拡張パッケージを追加していく。

## プロジェクトの初期化

ルートプロジェクト（ルートパッケージ）を作って，その下にサブパッケージを作る。
```bash
$ mkdir eggdishes
$ cd eggdishes
$ uv init --bare  # ルートパッケージの初期化. bareで作っておく.
$ uv init --package eggdishes-core --lib  # ライブラリパッケージをworkspacesのメンバーとして追加
$ uv init --package eggdishes-main --app  # アプリケーションパッケージをworkspacesのメンバーとして追加
```

コマンドから想像できる通り，`init`する時に与える`--package`オプションが，workspacesを構成する「パッケージ」を指定するオプションで，パッケージを指定して操作する時は必ずこのオプションを与える。

この時点で，ディレクトリ構成はこうなっている。（uvバージョンは0.5.29）

```bash
$ tree
.
├── eggdishes-core
│   ├── pyproject.toml
│   ├── README.md
│   └── src
│       └── eggdishes_core
│           ├── __init__.py
│           └── py.typed
├── eggdishes-main
│   ├── pyproject.toml
│   ├── README.md
│   └── src
│       └── eggdishes_main
│           └── __init__.py
├── pyproject.toml
└── uv.lock
```

ルートディレクトリ直下の`pyproject.toml`, `eggdishes-core/pyproject.toml`, `egggdishes-main/pyproject.toml`の3つのパッケージ設定ファイルができている。

## サブパッケージに依存関係を追加する

**3rd partyの依存を追加**

`eggdishes-main`はCLIアプリケーションの想定なので，[click](https://click.palletsprojects.com/en/stable/)を依存に追加する。

```bash
# eggdishes-mainパッケージにclickへの依存を追加
$ uv add --package eggdishes-main click
```

**パッケージ間の依存関係の追加**

`eggdishes-main`から`eggdishes-core`のコードを呼べるようにするため，パッケージ間の依存関係を追加する。そのためのコマンドはないので`eggdishes-main/pyproject.toml`を直接編集する。

```bash
$ vi eggdishes-main/pyproject.toml
[project]
...
dependencies = ["click>=8.1.8", "eggdishes-core"]  # dependencyにeggdishes-coreを追加
...

[tool.uv.sources]
eggdishes-core = { workspace = true }  # eggdishes-coreがworkspacesで管理されるメンバーであることを示す
```

この段階で，3つの`pyproject.toml`はこんな感じになる（一部不要なプロパティを削除したり，編集している）。

```bash
# ルートパッケージの設定
$ cat ./pyproject.toml 
[project]
name = "eggdishes"
version = "0.1.0"
description = "Egg Recipe CLI"
requires-python = ">=3.13"
dependencies = []

[tool.uv.workspace]
members = ["eggdishes-core", "eggdishes-main"]  # workspaceのメンバー定義

# eggdishes-coreパッケージの設定
$ cat eggdishes-core/pyproject.toml 
[project]
name = "eggdishes-core"
version = "0.1.0"
description = "core module for eggdishes"
readme = "README.md"
dependencies = []

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

# eggdishes-mainパッケージの設定
$ cat eggdishes-main/pyproject.toml 
[project]
name = "eggdishes-main"
version = "0.1.0"
description = "application module for eggdishes"
readme = "README.md"
dependencies = ["click>=8.1.8", "eggdishes-core"]

[project.scripts]
eggdishes = "eggdishes_main:main"  # CLIコマンド eggdishes を定義

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

大事なことは，各パッケージはそれぞれでdependency graphを管理できるということ。

ただし，プロジェクト全体では1つのlockfileしか持たないため，全体として整合している必要があり，パッケージ間でコンフリクトする依存関係は書けない。

> In a workspace, each package defines its own pyproject.toml, but the workspace shares a single lockfile, ensuring that the workspace operates with a consistent set of dependencies.

[https://docs.astral.sh/uv/concepts/projects/workspaces/](https://docs.astral.sh/uv/concepts/projects/workspaces/)

この制約があることで，個人的にはモノレポが満たしていてほしい性質が担保される（依存関係がコンフリクトしているモノレポは管理がつらいし，そもそもモノレポ管理の必要性が疑わしい気がする）。

なお，ここではルートパッケージでPythonバージョンを`>=3.13`で統一しているが，`required-python`はサブパッケージごとに定義できるため，パッケージごとに異なるPythonバージョンを指定することもできる。モノレポ内で，違うPythonバージョンはあまり使いたくないけれど，やむにやまれぬ事情でどうしてもパッケージごとに異なるPythonバージョンを指定したい時はあるかもしれない。

## dependency graphの確認

ここまでで，まだアプリケーションコードは1行も書いていないけれど基本となるプロジェクト構成が整ったので，`tree`, `sync` コマンドでdependency graphを確認しておく。

```bash
$ uv tree
Resolved 5 packages in 1ms
eggdishes v0.1.0
eggdishes-main v0.1.0
├── click v8.1.8
└── eggdishes-core v0.1.0
```

想定通りの依存グラフができている。

`sync`コマンドの動作も確認しておく。`sync`は，手元のPython環境を`pyproject.toml`と一致させるコマンドだが，ここでは3つのパッケージがあるため，`--package`オプションを指定することで環境の切り替えができる。
```bash
# eggdishesパッケージ(root)とsync
$ uv sync
$ uv pip freeze
# 依存ライブラリなし

# eggdishes-coreパッケージとsync
$ uv sync --package eggdishes-core
$ uv pip freeze
-e file:///home/moco/repo/eggdishes/eggdishes-core

# eggdishes-mainパッケージとsync
$ uv sync --package eggdishes-main
$ uv pip freeze
click==8.1.8
-e file:///home/moco/repo/eggdishes/eggdishes-core
-e file:///home/moco/repo/eggdishes/eggdishes-main
```

`tree`コマンドの結果と整合している。

## アプリのコードを書く

まずはコアライブラリのコードを書いていく。

```python
# eggdishes-core/src/eggdishes_core/lib.py
from abc import ABC, abstractmethod

class EggDish(ABC):
    name: str
    
    @abstractmethod
    def recipe(self) -> str:
        """Return the recipe for the egg dish."""
        pass

# eggdishes-core/src/eggdishes_core/fresh_egg.py
from .lib import EggDish

class FreshEgg(EggDish):
    name = "採れたて卵"

    def recipe(self):
        return "ご飯に新鮮な卵とかつお節をのせて、醤油をかけて召し上がれ！"
```

コードの内容自体はまったく重要ではないのだけど，`EggDish`という抽象クラスと，それを拡張した`FreshEgg`というデフォルト実装クラスがある。`recipe()`メソッドの実装によって振る舞いが変わる。

CLIアプリのほうはこんな感じで。

```python
# eggdishes-main/src/eggdishes_main/__init__.py 
from .cli import cli

# エントリーポイントとなるmainメソッド
def main() -> None:
    cli()

# eggdishes-main/src/eggdishes_main/cli.py
import click
from eggdishes_core.fresh_egg import FreshEgg

@click.group()
def cli():
    pass

# recipe コマンドで，FreshEgg.recipe()の内容を表示する
@cli.command(
    "recipe",
    help="Show the recipe for an egg dish."
)
def show_recipe():
    """Show the recipe for a fresh egg dish."""
    egg_dish = FreshEgg()
    click.echo(f"Recipe for {egg_dish.name}:")
    click.echo(egg_dish.recipe())
```

## 1stバージョンが完成

ここまでで，1stバージョンのアプリが動作するようになる。

`uv run eggdishes recipe`でテスト実行する。CLIコマンド`eggdishes`は`eggdishes-main`で定義しているので，`sync`で`eggdishes-main`とsyncしておくのを忘れないように。

```bash
$ uv sync --package eggdishes-main
$ uv run eggdishes recipe
Recipe for 採れたて卵:
ご飯に新鮮な卵とかつお節をのせて、醤油をかけて召し上がれ！
```

なおここまでのスナップショットを[0.1.0](https://github.com/mocobeta/uv-workspaces-eggdishes/tree/0.1.0)としてタグを打っておいたので，全体が見たい場合はこのタグのソースツリーを参照してほしい。

# プロジェクトを拡張する

ここまででworkspacesの基本の説明が終わったので，ここからは，いろんな卵料理のレシピを追加するための拡張ライブラリを書いていく。

## サブパッケージを追加する

スクランブルエッグ(`eggdishes-scrambled`)，目玉焼き(`eggdishes-sunnysideup`)，ポーチドエッグ(`eggdishes-poached`)，ゆで卵(`eggdishes-boiled`)の4つの拡張パッケージを追加する。

```
$ uv init --package eggdishes-scrambled --lib
$ uv init --package eggdishes-sunnysideup --lib
$ uv init --package eggdishes-poached --lib
$ uv init --package eggdishes-boiled --lib

$ tree -L2
.
├── eggdishes-boiled
│   ├── pyproject.toml
│   ├── README.md
│   └── src
├── eggdishes-core
│   ├── pyproject.toml
│   ├── README.md
│   └── src
├── eggdishes-main
│   ├── pyproject.toml
│   ├── README.md
│   └── src
├── eggdishes-poached
│   ├── pyproject.toml
│   ├── README.md
│   └── src
├── eggdishes-scrambled
│   ├── pyproject.toml
│   ├── README.md
│   └── src
├── eggdishes-sunnysideup
│   ├── pyproject.toml
│   ├── README.md
│   └── src
├── pyproject.toml
├── README.md
└── uv.lock

# workspacesのメンバーに，新規作成したパッケージが追加されている
$ cat ./pyproject.toml
...
[tool.uv.workspace]
members = [
    "eggdishes-core",
    "eggdishes-main",
    "eggdishes-scrambled",
    "eggdishes-sunnysideup",
    "eggdishes-poached",
    "eggdishes-boiled",
]
```

## サブパッケージの依存関係を更新

`eggdishes-boiled`は`eggdishes-core`に依存するので，依存関係を追加する。

```bash
$ vi eggdishes-boiled/pyproject.toml 
[project]
...
dependencies = ["eggdishes-core"]

[tool.uv.sources]
eggdishes-core = { workspace = true }
```

その他の拡張パッケージも同様。

また，`eggdishes-main`はすべての拡張パッケージを呼び出すため，`eggdishes-main/pyproject.toml`に以下を追加。

```bash
$ vi eggdishes-main/pyproject.toml
[project]
...
dependencies = [
    "click>=8.1.8",
    "eggdishes-core",
    "eggdishes-boiled",
    "eggdishes-poached",
    "eggdishes-scrambled",
    "eggdishes-sunnysideup",
]

[tool.uv.sources]
eggdishes-core = { workspace = true }
eggdishes-boiled = { workspace = true }
eggdishes-poached = { workspace = true }
eggdishes-scrambled = { workspace = true }
eggdishes-sunnysideup = { workspace = true }
```

## 拡張ライブラリパッケージのコード例

拡張ライブラリに追加するファイルは1つだけで，`EggDish`のサブクラスを生やす。

```python
# eggdishes-boiled/src/eggdishes_boiled/boiled_egg.py
from eggdishes_core.lib import EggDish

class BoiledEgg(EggDish):
    name = "Boiled Egg"

    def recipe(self):
        return """
1. 卵を常温に戻す
    冷蔵庫から出したばかりの卵はヒビが入りやすいので、10〜20分ほど室温に置きます。
2. 鍋に卵を並べる
    卵同士がぶつからないように注意しながら鍋に並べます。
3. 水を入れる
    卵がしっかり浸るくらいまで水を注ぎます。
4. 火にかける
    中火で加熱し、沸騰させます。
5. 茹で時間を調整する
    沸騰したらタイマーをセット！お好みの仕上がりに合わせて茹で時間を変えます。
    - 半熟（とろとろ）：6〜7分
    - 半熟（ねっとり）：8〜9分
    - 固ゆで（しっかり）：10〜12分
6. 冷水にとる
    茹で上がったらすぐに冷水（または氷水）に移して冷やします。殻がむきやすくなります。
完成！
"""
```

のような感じで，他の拡張パッケージについても，同様に`EggDish`の実装クラスをそれぞれ生やしておく。

CLIアプリケーションのほうは，追加した拡張をインポートして，`recipe`コマンドの引数に応じて挙動が切り替えられるようにする。

```python
# eggdishes-main/src/eggdishes_main/cli.py
import click
from eggdishes_core.fresh_egg import FreshEgg
from eggdishes_boiled.boiled_egg import BoiledEgg
from eggdishes_poached.poached_egg import PoachedEgg
from eggdishes_scrambled.scrambled_egg import ScrambledEgg
from eggdishes_sunnysideup.sunnysideup_egg import SunnysideUpEgg

@click.group()
def cli():
    pass

@cli.command(
    "recipe",
    help="Show the recipe for an egg dish."
)
@click.argument(
    "dish",
    type=click.Choice(["boiled", "poached", "scrambled", "sunnysideup", "fresh"]),
    default="fresh",
)
def show_recipe(dish: str):
    """Show the recipe for a fresh egg dish."""
    egg_dish = None
    if dish == "boiled":
        egg_dish = BoiledEgg()
    elif dish == "poached":
        egg_dish = PoachedEgg()
    elif dish == "scrambled":
        egg_dish = ScrambledEgg()
    elif dish == "sunnysideup":
        egg_dish = SunnysideUpEgg()
    else:
        egg_dish = FreshEgg()
    click.echo(f"Recipe for {egg_dish.name}:")
    click.echo(egg_dish.recipe())
```

## 2ndバージョンが完成

2ndバージョンのコードは[0.2.0](https://github.com/mocobeta/uv-workspaces-eggdishes/tree/0.2.0)としてタグを打っているので，全体感はこのタグのソースツリーを参照してほしい。

`recipe`コマンドの引数に応じて挙動が変わることを確認する。

```bash
$ uv sync --package eggdishes-main

# 引数に boiled を指定して recipe コマンドを実行
$ uv run eggdishes recipe boiled  
Recipe for Boiled Egg:

1. 卵を常温に戻す
    冷蔵庫から出したばかりの卵はヒビが入りやすいので、10〜20分ほど室温に置きます。
2. 鍋に卵を並べる
    卵同士がぶつからないように注意しながら鍋に並べます。
3. 水を入れる
    卵がしっかり浸るくらいまで水を注ぎます。
4. 火にかける
    中火で加熱し、沸騰させます。
5. 茹で時間を調整する
    沸騰したらタイマーをセット！お好みの仕上がりに合わせて茹で時間を変えます。
    - 半熟（とろとろ）：6〜7分
    - 半熟（ねっとり）：8〜9分
    - 固ゆで（しっかり）：10〜12分
6. 冷水にとる
    茹で上がったらすぐに冷水（または氷水）に移して冷やします。殻がむきやすくなります。
完成！

# 引数に scrambled を指定して recipe コマンドを実行
$ uv run eggdishes recipe scrambled
Recipe for Scrambled Egg:

1. 卵をよく溶く
    ボウルに卵を割り入れ、牛乳（または生クリーム）と一緒によく混ぜます。白身と黄身がしっかりなじむように。
2. フライパンにバターを溶かす
    中火より少し弱い火で、バターをゆっくり溶かします。
3. 卵液を流し入れる
    バターが溶けたら卵液を一気に流し入れます。
4. やさしくかき混ぜる
    菜箸やゴムベラで、フライパンの外側から中心に向かってゆっくり混ぜます。
    ※焦らず、火が強すぎないよう注意！
5. 半熟状態で火を止める
    卵が半熟でとろっとしてきたら火を止め、余熱で仕上げます。ふわふわ派はここで止めるのがポイント！
6. 塩・こしょうで味付け
    最後に軽く塩・こしょうをふる
完成！
```


# その他の話題

その他，雑多なトピックを補足しておく。

## workspacesの標準ディレクトリ構成

プロジェクト内にサブパッケージをどう配置するかについての縛りはないので，自由に配置できる。ただし，workspacesの公式ドキュメンテーションでは，`<project-root>/packages/`以下にサブパッケージを配置する例が書かれているので，この記事のようにルートディレクトリ直下にパッケージを置かずに，`pakcages/eggdishes-core`のように一段下げて配置するのが標準か推奨構成と思われる。サンプルコードを書いたあとで気づいた。。。

## dev dependency

`ruff`や`pyright`のような開発ツールは，ルートのdev dependency groupに追加すればOK。

```bash
$ uv add pyright --dev
$ uv add ruff --dev

$ cat ./pyproject.toml
...
[dependency-groups]
dev = [
    "pyright>=1.1.400",
    "ruff>=0.11.7",
]

$ uv run pyright
0 errors, 0 warnings, 0 informations

$ uv run ruff check
All checks passed!
```

## final dependency graph

ここまでのステップをすべて実行した最終的な依存グラフはこうなる。スッキリ！

```bash
$ uv tree
Resolved 13 packages in 5ms
eggdishes v0.2.0
├── pyright v1.1.400 (group: dev)
│   ├── nodeenv v1.9.1
│   └── typing-extensions v4.13.2
└── ruff v0.11.7 (group: dev)
eggdishes-main v0.2.0
├── click v8.1.8
├── eggdishes-boiled v0.2.0
│   └── eggdishes-core v0.2.0
├── eggdishes-core v0.2.0
├── eggdishes-poached v0.2.0
│   └── eggdishes-core v0.2.0
├── eggdishes-scrambled v0.2.0
│   └── eggdishes-core v0.2.0
└── eggdishes-sunnysideup v0.2.0
    └── eggdishes-core v0.2.0
```

## パッケージビルドと配布

PyPIなどに公開する場合は，シングルパッケージの時と同じく`build`と`publish`で。

```bash
# --all-pakcagesオプションを指定してbuildすると，workspaces内のすべてのパッケージが一気にビルドされる
$ uv build --all-packages
...
Successfully built dist/eggdishes_boiled-0.2.0.tar.gz
Successfully built dist/eggdishes_boiled-0.2.0-py2.py3-none-any.whl
Successfully built dist/eggdishes_core-0.2.0.tar.gz
Successfully built dist/eggdishes_core-0.2.0-py2.py3-none-any.whl
Successfully built dist/eggdishes_main-0.2.0.tar.gz
Successfully built dist/eggdishes_main-0.2.0-py2.py3-none-any.whl
Successfully built dist/eggdishes_poached-0.2.0.tar.gz
Successfully built dist/eggdishes_poached-0.2.0-py2.py3-none-any.whl
Successfully built dist/eggdishes_scrambled-0.2.0.tar.gz
Successfully built dist/eggdishes_scrambled-0.2.0-py2.py3-none-any.whl
Successfully built dist/eggdishes_sunnysideup-0.2.0.tar.gz
Successfully built dist/eggdishes_sunnysideup-0.2.0-py2.py3-none-any.whl

# --packageオプションを指定して，パッケージごとのビルドも可
$ uv build --package eggdishes-core
Building source distribution...
Building wheel from source distribution...
Successfully built dist/eggdishes_core-0.2.0.tar.gz
Successfully built dist/eggdishes_core-0.2.0-py2.py3-none-any.whl
```

```bash
# 配布 (PyPI等のパッケージインデックスへ公開)
$ uv publish --index <INDEX> --token <TOKEN> dist/*
```

# 所感

少し丁寧にuv workspacesの機能と動作を試してみて，個人的に理想に近いモノレポ構成が実現できそうな感触をもった。プロジェクトが一定の規模に成長すると，モノレポとして一定のコントロールを効かせながら，かつ実装と依存関係をサブ機能（サブパッケージ）ごとに分けて分割管理したいケースがよくある。今後使う機会が増えそう。
