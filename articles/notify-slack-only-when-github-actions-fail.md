---
title: "GitHub Actions が失敗したときだけ Slack に通知する"
emoji: "🔔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "githubactions", "slack"]
published: true
---

## 結論

こちらのツールをSlackに追加してください。

https://endid.app/

## GitHub Actions の結果を Slack に通知する方法

GitHub Actions の結果を Slack に通知する方法はいくつかあります。

- **Webhooks で Slack に通知する** （以下参考記事）

https://zenn.dev/wakkunn/articles/c6978bd2b5ca55

- **Slack App の GitHub から通知する** （以下参考記事）

https://zenn.dev/k_saito7/articles/notification-github-actions-workflow-to-slack

ただしそれぞれ以下の点で問題があります。

- **Webhooks で Slack に通知する** の場合

  - Webhooks の設定が面倒
  - Workflow に job を追記する必要がある

- **Slack App に GitHub から通知する** の場合
  - 「失敗したときだけ」通知する機能が未実装

Slack App の GitHub アプリは Workflow の通知自体はできるため惜しいです。
多くの人が「失敗したときだけ」のフィルターを熱望しているようですが、まだこの Issue は Close されていません。

https://github.com/integrations/slack/issues/1563

## Endid for GitHub が良い

[Endid for GitHub](https://endid.app/) といういい感じのツールを見つけました。
https://endid.app/

### 導入方法

1. Webページの Add to Slack から slack app directory 移動し、お好みのチャンネルにアプリを追加します。
   ![alt](https://raw.githubusercontent.com/kou72/zenn/main/image/notify-slack-only-when-github-actions-fail-1.png)
1. Slackに追加されたEndidからGitHub Connectionのボタンを選択して、GitHubアカウントと連携します。
1. Workflow notificationsを設定します。以下がおすすめです。
   - **Channel to receive notifications** : 通知先のチャンネル
   - **Notification on Success** : When State Changesd
   - **Notification on Failure** : Always
     ![alt](https://raw.githubusercontent.com/kou72/zenn/main/image/notify-slack-only-when-github-actions-fail-2.png)

これで **「失敗したときだけ」** Slackに通知できます（簡単！）
さらに `Notification on Success : When State Changesd` にしているので、 **「失敗→成功の時」** も通知されます（気が利く！）

通知のフォーマットはこんな感じのようです。
カスタマイズは難しそうですが、十分ですね。

![alt](https://raw.githubusercontent.com/kou72/zenn/main/image/notify-slack-only-when-github-actions-fail-3.png)

注意として無料プランだと Public Repository のみの対応となっています。

## まとめ

ちょうど欲しかったツールを見つけたので共有しました。
