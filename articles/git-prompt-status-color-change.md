---
title: "git-promptのステータスでプロンプトの色を変える"
emoji: "🌈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git", "prompt", "zsh"]
published: true
---

# gitのステータスでプロンプトの色を変えたい！

↓こんな感じにしたい。(zshを前提とします)

![](https://raw.githubusercontent.com/kou72/zenn/main/image/git-prompt-status-color-change1.png)

# git-promptの設定をする

まずはgit-promptを設定します。

**git-prompt.shをインストール**

```bash
mkdir ~/.zsh
curl -o ~/.zsh/git-prompt.sh https://raw.githubusercontent.com/git/git/master/contrib/completion/git-prompt.sh
```

**zshrc設定**

`.zshrc`

```sh
source ~/.zsh/git-prompt.sh
```

# git-promptの必要なオプションを確認する

git-promptのオプションを確認。

| 設定項目                   | 設定値 | 説明                                                                                                                                |
| -------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------------------------- |
| GIT_PS1_SHOWDIRTYSTATE     | true   | addされていない変更を「\*」で示す<br>commitされていない変更を「+」で示す                                                            |
| GIT_PS1_SHOWUNTRACKEDFILES | true   | addされていない新規ファイルの存在を「%」で示す                                                                                      |
| GIT_PS1_SHOWSTASHSTATE     | true   | stashがある場合は「$」で示す                                                                                                        |
| GIT_PS1_SHOWUPSTREAM       | auto   | upstream と同期している場合は「＝」で示す<br>upstream より進んでいる場合は「＞」で示す<br>upstream より遅れている場合は「＜」で示す |

必要な項目を設定します。
私の場合stashのステータスは不要なので`unset`しておきます。

```sh
# addされていない変更を「*」commitされていない変更を「+」で示す
GIT_PS1_SHOWDIRTYSTATE=true
# addされていない新規ファイルの存在を「%」で示す
GIT_PS1_SHOWUNTRACKEDFILES=true
# stashがある場合は「$」で示す
unset GIT_PS1_SHOWSTASHSTATE
# upstreamと同期「=」進んでいる「>」遅れている「<」で示す
GIT_PS1_SHOWUPSTREAM=auto

setopt PROMPT_SUBST
```

# プロンプトの設定をする

色分けするプロンプトを作成します。

`__git_ps1 "%s"`でgitのステータスを取り出せるので、これで場合分けする`git_color()`を作成します。

```sh
function git_color() {
  local git_info="$(__git_ps1 "%s")"
  if [[ $git_info == *"%"* ]] || [[ $git_info == *"*"* ]]; then
    echo '%F{red}'
  elif [[ $git_info == *"+"* ]]; then
    echo '%F{green}'
  else
    echo '%F{cyan}'
  fi
}
```

色を変えたい部分を`$(git_color)`と`%f`で囲みます。

```sh
PS1='%~ $(git_color)$(__git_ps1 "%s")%f\$ '
```

# 好みの設定を追加

あとは好みの設定を追加していきます。
参考までに私の設定を載せておきます。

- カレントディレクトリの色を紫に変更
  - `%F{magenta}%~%f`
- gitのステータスを黄色の[]で囲む
  - `%F{yellow}$(__git_ps1 "[")%f$(git_color)$(__git_ps1 "%s")%f%F{yellow}$(__git_ps1 "]")%f`
  - ※git initされてないディレクトでは表示したくなかったので、少し不格好ですがこうしています。
- プロンプトを2行にする
- 直前のプロンプトと一行空ける

```sh
PS1='
%F{magenta}%~%f %F{yellow}$(__git_ps1 "[")%f$(git_color)$(__git_ps1 "%s")%f%F{yellow}$(__git_ps1 "]")%f
\$ '
```

これで冒頭の画像のようなプロンプトが作成できました！

# zshrc全体

`.zshrc`

```sh
# prompt setting

## curl -o ~/.zsh/git-prompt.sh https://raw.githubusercontent.com/git/git/master/contrib/completion/git-prompt.sh
source ~/.zsh/git-prompt.sh

# addされていない変更を「*」commitされていない変更を「+」で示す
GIT_PS1_SHOWDIRTYSTATE=true
# addされていない新規ファイルの存在を「%」で示す
GIT_PS1_SHOWUNTRACKEDFILES=true
# stashがある場合は「$」で示す
unset GIT_PS1_SHOWSTASHSTATE
# upstreamと同期「=」進んでいる「>」遅れている「<」で示す
GIT_PS1_SHOWUPSTREAM=auto

setopt PROMPT_SUBST

function git_color() {
  local git_info="$(__git_ps1 "%s")"
  if [[ $git_info == *"%"* ]] || [[ $git_info == *"*"* ]]; then
    echo '%F{red}'
  elif [[ $git_info == *"+"* ]]; then
    echo '%F{green}'
  else
    echo '%F{cyan}'
  fi
}

PS1='
%F{magenta}%~%f %F{yellow}$(__git_ps1 "[")%f$(git_color)$(__git_ps1 "%s")%f%F{yellow}$(__git_ps1 "]")%f
\$ '

```
