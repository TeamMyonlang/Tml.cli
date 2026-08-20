# Tml.cli

Myon で CLI ツールを書くときに欲しくなるものをまとめた小さなライブラリ。
引数パース・ANSI カラー出力・プログレスバー / スピナーを提供する。

## Install

```sh
myon pkg install https://github.com/TeamMyonlang/Tml.cli
```

## Modules

このパッケージは `cli` namespace 配下に、それぞれ独立した submodule として
3つの機能を提供する。必要なものだけを import できる。

| module | 内容 |
| --- | --- |
| `cli.args` | コマンドライン引数パーサー |
| `cli.color` | ANSI カラー / 装飾出力 |
| `cli.progress` | プログレスバー・スピナー |

## `cli.args`

`myon.argv()` の生配列を解析し、`--flag` / `--key=value` / `--key value` /
`-f`（短縮フラグ）/ positional 引数 に振り分ける。

```myon
system myon.useversion=1
module myon.stdio
module cli.args as args

parsed = args.parse(myon.argv())

myon.if args.has(parsed, "verbose") then {
    myon.print("verbose mode")
}

name = args.get_str(parsed, "name", "world")
count = args.get_int(parsed, "count", 1)
force = args.get_bool(parsed, "force")

pos = args.positionals(parsed)
first = args.positional_at(parsed, 0, "")
```

`--key value` の形式は、次のトークンが別のオプション（`-` で始まる）
でない場合にのみ値として消費される。それ以外（次が無い／次もオプション）は
値なしフラグとして `"true"` が入る。

| 関数 | シグネチャ | 説明 |
| --- | --- | --- |
| `parse` | `(argv: myon.array(str)) ret ParsedArgs` | argv を解析する |
| `has` | `(p: ParsedArgs, key: str) ret bool` | key が指定されたか |
| `get_str` | `(p: ParsedArgs, key: str, default: str) ret str` | 文字列値を取得 |
| `get_int` | `(p: ParsedArgs, key: str, default: int) ret int` | int としてパース |
| `get_bool` | `(p: ParsedArgs, key: str) ret bool` | `"true"/"1"/"yes"` を true とみなす |
| `positionals` | `(p: ParsedArgs) ret myon.array(str)` | positional 引数一覧 |
| `positional_at` | `(p: ParsedArgs, index: int, default: str) ret str` | index 番目の positional |

## `cli.color`

ANSI エスケープシーケンスで文字列を装飾する。前景色8色・明るい前景色8色・
背景色8色・装飾（bold/dim/italic/underline）・256色（`fg256`/`bg256`）・
セマンティックヘルパー（`success`/`error_text`/`warn`/`info`）を持つ。

```myon
system myon.useversion=1
module myon.stdio
module cli.color as color

myon.print(color.red("error!"))
myon.print(color.bold(color.green("done")))
myon.print(color.success("OK"))
myon.print(color.fg256(208, "orange-ish"))
```

パイプ出力やログ保存などで ANSI を無効化したい場合は `strip` で除去できる。

```myon
plain = color.strip(color.red("colored"))
```

`myon.is_tty()`（`module myon.stdio`）と組み合わせて、TTY でないときだけ
`strip` を通す、という使い方を推奨する。

## `cli.progress`

`myon.write("\r...")` + `myon.flush()` による1行上書き方式のプログレスバーと
スピナー。TTY でない場合（パイプ・リダイレクト）は `\r` 上書きをせず、
割合(%)が変化したときだけ改行付きで出力するため、ログファイルが荒れない。

```myon
system myon.useversion=1
module myon.stdio
module cli.progress as progress

bar = progress.new_labeled(100, "download")
myon.for i myon.in range(0, 101) {
    progress.update(bar, i)
    // 実際の処理...
}
progress.finish(bar)
```

不定長タスク向けのスピナー:

```myon
module myon.time

sp = progress.new_spinner("working")
myon.for i myon.in range(0, 20) {
    progress.spin(sp)
    myon.time.sleep_ms(100)
}
progress.spinner_finish(sp, "done!")
```

| 関数 | シグネチャ | 説明 |
| --- | --- | --- |
| `new` | `(total: int) ret Bar` | total を100%としたバーを作成 |
| `new_labeled` | `(total: int, label: str) ret Bar` | ラベル付きで作成 |
| `update` | `(bar: Bar, current: int) ret void` | 現在値を反映して描画 |
| `finish` | `(bar: Bar) ret void` | 100%で確定し改行する |
| `new_spinner` | `(label: str) ret Spinner` | スピナーを作成 |
| `spin` | `(sp: Spinner) ret void` | 次フレームへ進めて描画（TTYのみ） |
| `spinner_finish` | `(sp: Spinner, message: str) ret void` | 終了メッセージを確定 |

## Security

このパッケージは Myon の `myon.stdio` / `myon.string` の範囲のみを使用し、
file I/O・network・FFI は一切行わない。
