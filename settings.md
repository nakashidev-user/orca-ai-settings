Claude Code × Orca：main で指示 → PR のワークツリーで Claude に引き継ぐ

何が嬉しいか：指示はいつも main のセッションで出すだけでよく、PR ごとの作業はその PR 専用のワークツリーで別の Claude が進める。main のブランチは切り替わらない。
いつ発火するか：プロンプトに #12345（4桁以上）や .../pull/12345 が含まれたとき（UserPromptSubmit hook）。
何をするか：main の Claude が会話全体から PR ごとの引き継ぎ文を書き、orca-open-pr.sh でその PR のワークツリーに Claude を起動して渡す。main の Claude は、起動したパスと各 PR に渡した要点を報告して終わる。

引き継ぎ文に入るもの（main の Claude が PR ごとに書く）

• その PR でやること。会話全体からユーザーの意図を判断して書く（プロンプトの原文は貼らない）.
• その PR での担当範囲（何をして、何をしないか）。ほかの PR の作業は入れない.
• main で決まった前提・判断と、その根拠.
• 完了したときに報告する内容会話に出ていない推測は書きません。複数の PR に同時に指示しても、それぞれの Claude には自分の担当分だけが渡ります。.

挙動

• まだワークツリーが無い：orca worktree create で作り、PR の head ブランチに切り替えて origin に fast-forward する。作成時にできる最初の端末で Claude を起動する（タブは増えない）.
• すでにワークツリーで開いている：そのワークツリーに端末タブを1つ追加して Claude を起動する.
• gh で答えられる状態確認だけ：ワークツリーは開かず、main で答える.
• PR 番号が無い、または git リポジトリの外：何もしない起動した Claude は PR の head ブランチ上で作業します。修正はそのブランチにコミットして通常どおり push します（新しいブランチや PR は作りません）。.

注意点

1. 引き継ぎ文の中身は main の Claude の判断です。main の報告に出る「各 PR に渡した要点」で、ずれていないか確認してください。渡した引き継ぎ文は ~/.claude/state/orca-handoff/<日時>-pr<番号>/prompt.txt に残ります。.
2. 既存のワークツリーで起動すると、そのたびにタブが1つ増えます。使い終わったタブは閉じてください。.
3. 起動は bash のランチャー経由なので、ログインシェルが fish でも動きます。~/.claude/state/orca-handoff/ にはファイルが溜まり続けるので、ときどき掃除してください。.

導入方法
前提：gh、jq、orca CLI が使えること。対象リポジトリが Orca に登録済みであること。
下の「設定用プロンプト」をまるごと Claude Code に貼ると、ファイルの配置・hook の登録・CLAUDE.md への追記・検証まで進めてくれます。

———

設定用プロンプト

Claude Code に「main で指示 → PR のワークツリーで Claude に引き継ぐ」仕組みを設定してほしい。以下の手順どおりに進めて、最後に結果を報告して。

0. 前提の確認（足りなければ止まって私に聞く）

• gh・jq・orca CLI が PATH にあり、gh auth status が通ること.
• このリポジトリが Orca に登録済みであること（orca repo list の3列目にリポジトリのルートパスが出る）.
• 以下で作るファイルがすでにある場合は、上書きせずに差分を見せて確認を取ること.

1. ~/.claude/scripts/orca-open-pr.sh を作る（内容は一字一句このまま）

#!/bin/bash
# PR の head ブランチを Orca 管理のワークツリーで開き、そのパスを出力する。
# 既にどこかのワークツリーで開いていればそこを返す（新規作成しない）。
# --handoff <file> を渡すと、そのワークツリーで Claude を起動し、ファイルの内容を最初のプロンプトとして渡す。
#   新規ワークツリー: 作成時の最初の端末で起動する。既存ワークツリー: 端末タブを1つ追加して起動する。
# usage: orca-open-pr.sh <PR番号> [リポジトリ内のパス(default: cwd)] [--handoff <prompt-file>]
set -euo pipefail

handoff=""
args=()
while [ $# -gt 0 ]; do
  case "$1" in
    --handoff) handoff="${2:?--handoff にはファイルが必要}"; shift ;;
    *) args+=("$1") ;;
  esac
  shift
done
pr="${args[0]:?usage: orca-open-pr.sh <PR番号> [repo-path] [--handoff <prompt-file>]}"
repo_dir="${args[1]:-$PWD}"
[ -z "$handoff" ] || [ -s "$handoff" ] || { echo "handoff ファイルが空か存在しない: $handoff" >&2; exit 1; }
main_root=$(cd "$(git -C "$repo_dir" rev-parse --git-common-dir)/.." && pwd)

head=$(cd "$repo_dir" && gh pr view "$pr" --json headRefName -q .headRefName)
git -C "$repo_dir" fetch -q origin "$head"

path=$(git -C "$repo_dir" worktree list --porcelain \
  | awk -v b="refs/heads/$head" '/^worktree /{p=substr($0,10)} $0=="branch "b{print p}')
first_terminal=""

if [ -z "$path" ]; then
  repo_id=$(orca repo list | awk -v r="$main_root" '$3==r{print $1}')
  [ -n "$repo_id" ] || { echo "Orca にリポジトリ未登録: $main_root" >&2; exit 1; }

  created=$(orca worktree create --repo "id:$repo_id" --name "pr-$pr" --base-branch "origin/$head" \
    --setup skip --no-parent --comment "PR #$pr ($head)" --json)
  path=$(jq -r '.. | objects | select(has("git")) | .git.path' <<<"$created" | head -1)
  [ -n "$path" ] || { echo "orca worktree create の出力からパスを取得できない" >&2; exit 1; }
  first_terminal=$(jq -r '[.. | objects | .startupTerminal?.handle? // empty] | first // empty' <<<"$created")
  [ -n "$first_terminal" ] || first_terminal=$(orca terminal list --worktree "path:$path" --json \
    | jq -r '.result.terminals[0].handle // empty')

  orca_branch=$(git -C "$path" branch --show-current)
  git -C "$path" switch -q "$head"
  git -C "$path" merge -q --ff-only "origin/$head" \
    || echo "警告: ローカルの $head が origin と分岐している（fast-forward 不可）" >&2
  # Orca が作った作業用ブランチは PR のブランチと同じコミットを指すだけなので外す（未マージなら -d が拒否して残る）
  [ "$orca_branch" = "$head" ] || git -C "$path" branch -q -d "$orca_branch" 2>/dev/null || true

  title=$(cd "$repo_dir" && gh pr view "$pr" --json title -q .title)
  orca worktree set --worktree "path:$path" --display-name "PR #$pr $title" --comment "PR #$pr ($head)" --json >/dev/null 2>&1 || true
fi

if [ -n "$handoff" ]; then
  # 端末のログインシェル（fish 等）に依存しないよう、起動は bash のランチャー経由にする
  dir="$HOME/.claude/state/orca-handoff/$(date +%Y%m%d-%H%M%S)-pr$pr"
  mkdir -p "$dir"
  cp "$handoff" "$dir/prompt.txt"
  printf '#!/bin/bash\nexec claude "$(cat %q)"\n' "$dir/prompt.txt" > "$dir/launch.sh"
  launch="bash $(printf %q "$dir/launch.sh")"
  if [ -n "$first_terminal" ]; then
    orca terminal send --terminal "$first_terminal" --text "$launch" --enter --json >/dev/null
    echo "Claude を起動（最初の端末）: PR #$pr (prompt: $dir/prompt.txt)" >&2
  else
    orca terminal create --worktree "path:$path" --title "Claude PR #$pr" --command "$launch" --json >/dev/null
    echo "Claude を起動（タブ追加）: PR #$pr (prompt: $dir/prompt.txt)" >&2
  fi
fi

echo "$path"

2. ~/.claude/hooks/pr-worktree-nudge.sh を作る（内容は一字一句このまま）

#!/bin/bash
# プロンプトに PR 番号 / PR URL があれば、Orca のワークツリーで Claude に引き継ぐ手順をコンテキストに差し込む。

input=$(cat)
prompt=$(printf '%s' "$input" | jq -r '.prompt // empty')
cwd=$(printf '%s' "$input" | jq -r '.cwd // empty')

prs=$(printf '%s' "$prompt" | grep -oE '(#[0-9]{4,}|/pull/[0-9]+)' | grep -oE '[0-9]+' | sort -u | tr '\n' ' ')
[ -n "$prs" ] || exit 0
git -C "${cwd:-.}" rev-parse --git-dir >/dev/null 2>&1 || exit 0

branch=$(git -C "${cwd:-.}" branch --show-current 2>/dev/null)
ctx="PR 参照を検知: ${prs}(現在の作業ディレクトリ ${cwd} のブランチ: ${branch:-detached})。
この PR のコード変更・レビュー・調査など、ワークツリーでの作業が必要なら、このセッションでは作業せず、PR ごとに引き継ぎ文を書いて起動する。
1. 引き継ぎ文をファイルに書く（PR ごとに1つ）。このセッションの会話から、その PR に関係する部分だけを選ぶ:
      - この PR でやること: 会話全体からユーザーの意図を判断して書く（プロンプトの原文は貼らない）
      - この PR での担当範囲: 何をして、何をしないか（他の PR に属す作業は含めない）
      - main で決まった前提・判断と、その根拠
      - 完了時の報告内容
   引き継ぎ文だけで作業が始められるようにし、会話にない推測は書かない。
2. \`bash ~/.claude/scripts/orca-open-pr.sh <PR番号> --handoff <引き継ぎ文ファイル>\` を実行する。新規ワークツリーなら最初の端末、既存ならタブを1つ追加して Claude が起動する。
3. このセッションは、起動したワークツリーのパスと、各 PR に渡した要点を報告して終える。
gh で答えられる状態確認だけなら、ワークツリーは開かない。"

jq -n --arg c "$ctx" '{hookSpecificOutput:{hookEventName:"UserPromptSubmit",additionalContext:$c}}'

3. hook を登録する
~/.claude/settings.json の hooks.UserPromptSubmit 配列に、次の要素を1つ追加する。既存の hooks や他の設定は消さない。同じ command がすでにあれば追加しない。

{ "hooks": [{ "type": "command", "command": "bash ~/.claude/hooks/pr-worktree-nudge.sh", "timeout": 10 }] }

4. 運用方針を ~/.claude/CLAUDE.md に追記する（既存の内容は変えない）

## 別 PR の作業は Orca のワークツリーで行う
指示は常に main のセッションで出す。PR のワークツリーでの作業は main では行わず、PR ごとに引き継ぎ文を書いて `bash ~/.claude/scripts/orca-open-pr.sh <PR番号> --handoff <引き継ぎ文ファイル>` で起動した Claude に任せる。
起動した Claude は PR の head ブランチ上で作業し、修正はそのブランチにコミットして通常 push する。新しいブランチや PR は作らない。
gh で答えられる状態確認だけなら、ワークツリーは開かない。

5. 検証して報告する

• bash -n で2つのスクリプトの構文を確認する.
• hook を擬似入力で動かし、PR 番号があれば additionalContext が出て、無ければ何も出ないことを確認するprintf '%s' '{"prompt":"#12345 を見て","cwd":"'"$PWD"'"}' | bash ~/.claude/hooks/pr-worktree-nudge.sh.
• 実際にワークツリーを作る起動テストは、PR 番号を私に確認してから行う（ワークツリーが1つ増えるため）.
• 報告には、作ったファイル・変更した設定・検証結果・やらなかったことを書く。settings.json の hook は、Claude Code を再起動するか /hooks を開いてから有効になることも伝える.
