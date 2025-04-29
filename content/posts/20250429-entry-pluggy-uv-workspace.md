+++
title = "uv workspacesとpluggyで作る，プラッガブルなPythonエコシステム"
date = "2025-04-29"

[taxonomies]
categories = ["Long Posts"]
tags = ["in-depth", "python", "pluggy"]
+++

[uv workspacesでスッキリ作るPythonモノレポ](https://blog.mocobeta.dev/posts/20250427-entry-monorepo-with-uv-workspaces/)では，uv workspacesを使ったモノレポ構成について書いた。この記事はその続編で，モノレポに[pluggy](https://pluggy.readthedocs.io/en/latest/)を組み合わせて，プラッガブルなPythonエコシステムを作っていく。

# この記事で実現したいこと

uv workspacesのcommon use caseとして，「プラグインシステム」が挙げられている。

> A library with a plugin system, where each plugin is a separate workspace package with a dependency on the root.

[https://docs.astral.sh/uv/concepts/projects/workspaces/#when-not-to-use-workspaces](https://docs.astral.sh/uv/concepts/projects/workspaces/#when-not-to-use-workspaces)

## プラグインシステムとpluggy

前回の記事では，コアライブラリ(`eggdishes-core`)と拡張ライブラリ(`eggdishes-*`)を別パッケージに分離することで，インタフェースと実装部分を分けて開発することができるようになった。ただし，アプリケーションであるCLIツール(`eggdishes-main`)のほうは，コアライブラリ（インタフェース）に加えてすべての拡張ライブラリ（実装クラス）の詳細を知らないといけない。拡張を追加するたびにアプリケーション側を変更しないといけないのは，拡張性という観点では自由度が低い。

[pluggy](https://pluggy.readthedocs.io/en/latest/)は，pytestで使われているプラグイン管理ツールで，インタフェースとその拡張ないし実装（プラグイン）を仲介してくれる。拡張を使うアプリケーション（ホストプログラム）は，インタフェースとプラグインマネージャー（後述）だけ知っていれば良い。アプリケーション側を変更せずとも，拡張ライブラリをホストプログラムと同じ環境に`pip install`するだけで呼べるようになる。

この記事では，前回作ったモノレポにpluggyを導入して，CLIアプリケーションから拡張ライブラリへの依存を剥がし，ドロップイン方式で拡張ライブラリが実行できるようにしていく。

# toy projectのサンプルコード

[https://github.com/mocobeta/uv-workspaces-eggdishes](https://github.com/mocobeta/uv-workspaces-eggdishes)

# pluggy入門

pluggyの使い方は少し込み入っていて，公式リファレンスを読んでもとっつきづらいため，実際に自分のプロジェクトに導入しながら手を動かすほうが理解がすすむ。

ということで早速やっていく。

## pluggyのインストール

`EggDish`インタフェースの定義があるコアライブラリにpluggyをインストールする（他のパッケージは，コアライブラリ経由でpluggyの機能が使えるため，インストール不要）。

```bash
$ uv add --package eggdishes-core pluggy
```

## スペック定義と実装マーカーの公開

プラグインのスペック（仕様）と，その実装を示すマーカーをコアライブラリの`hookspecs.py`（ファイル名はなんでも良い）に定義する。

```python
# eggdishes-core/src/eggdishes_core/hookspecs.py
import pluggy

# プラグイン仕様を示すマーカー
hookspec = pluggy.HookspecMarker("eggdishes")
# プラグイン実装を示すマーカー
hookimpl = pluggy.HookimplMarker("eggdishes")

# プラグインが実装するべきメソッドインタフェース
@hookspec
def register_eggdish(recipes):
    """Register egg dish recipe."""
    pass
```

`__init__.py` で実装マーカーを公開しておくと，プラグイン側から使いやすい。

```python
# eggdishes-core/src/eggdishes_core/__init__.py
from .hookspecs import hookimpl
```

## プラグインマネージャーを書く

プラグインのスペックと実装をつなぐPluginManagerのコードを，同じくコアライブラリに追加する。pluggyを使う上で，たぶんここが一番ややこしいので，細かくコメントをつけてみた。

```python
# eggdishes-core/src/eggdishes_core/plugins.py
from .lib import EggDish
from . import hookspecs
from . import fresh_egg

import pluggy

# EggDish のレジストリ
# プラグイン名と，対応する実装クラスのファクトリメソッドを管理する
__recipes = {}

def __get_plugin_manager():
    # プラグインマネージャーを取得
    pm = pluggy.PluginManager("eggdishes")
    # hookspec(プラグインのインターフェース)を登録
    pm.add_hookspecs(hookspecs)
    # 環境内のプラグインをロード
    # このentry point名をプラグイン側のpyproject.tomlのentry pointと一致させることで，プラグインとして認識される
    pm.load_setuptools_entrypoints("eggdishes")  
    # fresh_eggの実装はeggdishes_core自身で定義されているので，明示的に登録する
    pm.register(fresh_egg)
    return pm


def register_eggdishes():
    """Register available recipes to the registry."""
    pm = __get_plugin_manager()
    # 環境内のすべてのプラグインフックを呼び出す
    for recipe in pm.hook.register_eggdish(recipes=__recipes):
        if recipe:
            __recipes[recipe.name] = recipe


def get_available_recipes():
    """Return a list of available egg dish recipes."""
    return __recipes.keys()


def get_recipe(name) -> EggDish | None:
    """Return the recipe for the given egg dish name."""
    recipe = __recipes.get(name)
    if recipe:
        return recipe()
    return None
```

## プラグインを書く

コアライブラリ内で定義したデフォルト実装`FreshEgg`をプラグインとして登録する。

```python
# eggdishes-core/src/eggdishes_core/fresh_egg.py
from .lib import EggDish
import eggdishes_core

class FreshEgg(EggDish):
    name = "採れたて卵"

    def recipe(self):
        return "ご飯に新鮮な卵とかつお節をのせて、醤油をかけて召し上がれ！"


# プラグイン実装
@eggdishes_core.hookimpl
def register_eggdish(recipes):
    def create():
        return FreshEgg()
    recipes["fresh"] = create
```

ここまででpluggyの使い方の説明がだいたい終わったのだけれど，おわかりいただけただろうか...。初見でぱっとわかるかというと正直厳しいと思う（私はわかったと言えるまでに数時間はかかった）が，スペック定義，プラグインマネージャー，プラグイン実装のコードを突き合わせて追いかけると，頭が整理できてくると思う。

## アプリケーション（ホストプログラム）の変更

CLIツール(`eggdishes-main`)のほうにも手を入れて，拡張クラスへの依存をすべて剥がし，コアライブラリが提供するプラグインシステムを使うように変更する。また，環境内で利用可能なプラグインをリストする`list`コマンドを追加する。

```bash
$ cat eggdishes-main/pyproject.toml 
[project]
...
dependencies = ["click>=8.1.8", "eggdishes-core"]
```

```python
# eggdishes-main/src/eggdishes_main/cli.py
import click
from eggdishes_core.plugins import register_eggdishes, get_available_recipes, get_recipe

@click.group()
def cli():
    pass

@cli.command(
    "list",
    help="List available egg dish recipes."
)
def list_recipes():
    register_eggdishes()
    available_recipes = get_available_recipes()
    click.echo("Available egg dish recipes:")
    for recipe in available_recipes:
        click.echo(f"- {recipe}")


@cli.command(
    "recipe",
    help="Show the recipe for an egg dish."
)
@click.argument(
    "dish",
    type=str,
    default="fresh",
)
def show_recipe(dish: str):
    register_eggdishes()
    recipe = get_recipe(dish)
    if recipe:
        click.echo(f"Recipe for {recipe.name}:")
        click.echo(recipe.recipe())
    else:
        click.echo(f"No recipe found for {dish}.")
```

前の記事での完成形だったバージョン`0.2.0` の [`eggdishes-main/pyproject.toml`](https://github.com/mocobeta/uv-workspaces-eggdishes/blob/0.2.0/eggdishes-main/pyproject.toml), [`eggdishes-main/src/eggdishes_main/cli.py`](https://github.com/mocobeta/uv-workspaces-eggdishes/blob/0.2.0/eggdishes-main/src/eggdishes_main/cli.py) と比較すると違いがわかりやすいと思う。

dependency graphも確認しておく。

```bash
$ uv tree
Resolved 14 packages in 68ms
eggdishes v0.4.0
├── pyright v1.1.400 (group: dev)
│   ├── nodeenv v1.9.1
│   └── typing-extensions v4.13.2
└── ruff v0.11.7 (group: dev)
eggdishes-boiled v0.4.0
└── eggdishes-core v0.4.0
    └── pluggy v1.5.0
eggdishes-main v0.4.0
├── click v8.1.8
└── eggdishes-core v0.4.0 (*)
eggdishes-poached v0.4.0
└── eggdishes-core v0.4.0 (*)
eggdishes-scrambled v0.4.0
└── eggdishes-core v0.4.0 (*)
eggdishes-sunnysideup v0.4.0
└── eggdishes-core v0.4.0 (*)
```

## 動作確認

ここまでで，動作を一度確認する。プラグインはコアに含まれる`FreshEgg`しか登録されていない。

```bash
$ uv sync --package eggdishes-main

$ uv run eggdishes list
Available egg dish recipes:
- fresh

$ uv run eggdishes recipe fresh
Recipe for 採れたて卵:
ご飯に新鮮な卵とかつお節をのせて、醤油をかけて召し上がれ！

$ uv run eggdishes recipe spam
No recipe found for spam.
```

# pluggy実践編（プラグイン追加）

ここからは，拡張ライブラリ（`eggdishes-*`）を全部プラグイン化していく。デフォルト実装`FreshEgg`をプラグイン化したのと同じように，拡張コード内に`hookimpl`マーカーをつけた実装を追加するだけ。たとえば`BoiledEgg`の場合：

```python
from eggdishes_core.lib import EggDish
import eggdishes_core


class BoiledEgg(EggDish):
    name = "Boiled Egg"

    def recipe(self):
        # snip

@eggdishes_core.hookimpl
def register_eggdish(recipes):
    def create():
        return BoiledEgg()
    recipes["boiled"] = create
```

また，このパッケージがプラグインであることをプラグインマネージャーに通知するために，`pyproject.toml`に以下2行を追加する。

```bash
$ cat eggdishes-boiled/pyproject.toml 
...

# プラグインのentry pointを登録
[project.entry-points."eggdishes"]
"boiled" = "eggdishes_boiled.boiled_egg"
```

プラグインパッケージ側の変更は以上。

## 動作確認（プラグイン全部入り版）

すべての拡張ライブラリパッケージを同じ要領でプラグイン化したら，パッケージをビルドして動作確認する。

```bash
$ uv build --all-packages
```

ドロップインでプラグインが動作するかは実際に`pip install`してみないと確認できないので，適当な環境を作ってwheelをインストールする。

```bash
$ mkdir -p ~/temp/eggdishes-test
$ cd ~/temp/eggdishes-test
$ python -m venv .venv
$ . .venv/bin/activate
$ pip install <path-to-eggdishes-repo>/dist/*.whl
```

こんな感じで，インストールしたプラグインが全て認識されるはず。

```bash
$ eggdishes list
Available egg dish recipes:
- fresh
- sunnysideup
- poached
- boiled
- scrambled

$ eggdishes recipe sunnysideup
Recipe for Sunny Side Up Egg:

1. フライパンを温める
    中火でフライパンを軽く温め、油かバターを入れて広げます。
2. 卵をそっと割り入れる
    黄身が割れないように優しく！できれば小さい器に一度割ってからフライパンへ流し込むと安心。
3. 弱火にして水を加える
    卵の白身が広がったら、すぐに火を弱火にしてフライパンの端から水を加えます。
4. すぐふたをする
    加えた水が蒸気になって卵を包み込みます。ふたをして、約1〜2分蒸し焼きに。
5. 様子を見ながら火を止める
    白身が固まって、黄身がまだぷるんぷるんしていたらOK！
    早めに火を止めて余熱で少しだけ火を通すと、絶妙な半熟に仕上がります。
6. 塩・こしょうで味付けする
完成！
```

ここまでのコードは[0.4.0](https://github.com/mocobeta/uv-workspaces-eggdishes/tree/0.4.0)としてタグを打っているので，全体感はこのタグのソースツリーを参照してほしい。

# まとめ

uv workspacesとpluggyを使うと，

1. 関連する複数パッケージをモノレポで管理する
2. 粗結合・プラッガブルなアプリケーションフレームワークを作る

ことができた。

プラグインのコードはもちろん，モノレポ内で管理されていなくても良い。「フレームワークとデフォルトプラグインをコア開発者が提供して，他の開発者が自作プラグインを作って公開できる」というようなエコシステムまで視野に入れると，このスキームの威力が発揮される。
