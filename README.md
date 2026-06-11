# Slack 完全再現仕様書 (Slack Full-Clone Specification)

Slack を「細部まで同一の製品」として再実装できるよう、ドメインごとに分析・言語化した仕様書群です。
各ファイルは独立して読めますが、ID 体系・用語は `slack-spec/00-overview.md` で全ファイル共通に定義しています。

## 目次

| # | ファイル | 内容 |
|---|---|---|
| 00 | [overview](slack-spec/00-overview.md) | 製品の本質定義・設計原則・全体アーキテクチャ・ID 体系・階層モデル・用語集 |
| 01 | [domain-model](slack-spec/01-domain-model.md) | データモデル / DB スキーマ全定義・シャーディング・整合性ルール |
| 02 | [realtime-messaging](slack-spec/02-realtime-messaging.md) | WebSocket 配信・boot・楽観 UI・presence・既読同期・整合性 |
| 03 | [messaging-features](slack-spec/03-messaging-features.md) | 投稿/編集/削除/mrkdwn/メンション/リアクション/ピン/保存/unfurl |
| 04 | [channels-dms-threads](slack-spec/04-channels-dms-threads.md) | チャンネル/DM/MPIM/スレッド/既読/サイドバー/Slack Connect |
| 05 | [search](slack-spec/05-search.md) | 全文検索・修飾子・ランキング・クイックスイッチャー |
| 06 | [notifications-presence](slack-spec/06-notifications-presence.md) | 通知判定パイプライン/プッシュ/メール/DND/カスタムステータス |
| 07 | [huddles-calls-canvas-clips](slack-spec/07-huddles-calls-canvas-clips.md) | Huddle/通話(WebRTC)/Canvas(共同編集)/Clip |
| 08 | [files](slack-spec/08-files.md) | アップロード3段方式/プレビュー/権限/保持 |
| 09 | [platform-apps](slack-spec/09-platform-apps.md) | OAuth/Bot/Webhook/Slash/Events/Block Kit/Workflow |
| 10 | [admin-enterprise-security](slack-spec/10-admin-enterprise-security.md) | ロール/Grid/SSO/SCIM/Retention/eDiscovery/DLP/EKM |
| 11 | [clients-ui-ux](slack-spec/11-clients-ui-ux.md) | レイアウト/テーマ/ショートカット/オンボーディング/i18n |
| 12 | [apis](slack-spec/12-apis.md) | Web API/Events API/Socket Mode/レート制限/署名検証 |

## 設計上の不可侵原則（これを外すと「Slack ではない」）

1. **チャンネル中心** — 会話の第一級単位は「人」ではなく「チャンネル（トピック）」。
2. **`ts` がメッセージの一意キー兼ソートキー兼スレッド親参照** — チャンネル内で単調増加するマイクロ秒タイムスタンプ文字列。
3. **既読は per-channel の `last_read`** — メッセージ単位の既読フラグは持たない。
4. **スレッドは親メッセージ `ts` への返信** — 別エンティティではない。
5. **保存はテキスト＋構造（mrkdwn / Block Kit）** — WYSIWYG ではない。
6. **参加すれば過去ログ全体が読める** — メール的な「個人の受信箱」モデルではない。
7. **即時配信 + 楽観 UI + 起動スナップショット + 差分イベント** — 「Slack っぽさ」の正体。

> 注: 本書は公開情報・一般的なシステム設計知識に基づく外形的な再現仕様であり、Slack 社の内部実装そのものではありません。
