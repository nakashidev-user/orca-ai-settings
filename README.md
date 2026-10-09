# orca-ai-settings

Claude Code × Orca の設定をまとめたリポジトリ。「main のセッションで指示 → PR ごとのワークツリーで別の Claude に引き継ぐ」仕組みを扱う。

## ファイル

| ファイル | 内容 |
|---|---|
| `settings.md` | 仕組みの説明（何が嬉しいか・発火条件・挙動・注意点・導入方法）と設定用プロンプト |
| `setup-prompt.md` | `settings.md` から設定用プロンプトだけを抜き出したもの。Claude Code にそのまま貼る |

## 使い方

前提: `gh`・`jq`・`orca` CLI が使え、対象リポジトリが Orca に登録済みであること。

1. `setup-prompt.md` の中身を Claude Code に貼る。
2. スクリプト・hook の配置、`~/.claude/settings.json` への登録、`~/.claude/CLAUDE.md` への追記、検証まで Claude が進める。
