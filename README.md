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

## 実装グレード仕様 (`slack-spec/impl/`)

上の 00–12 が「何があるか（機能仕様）」、こちらは「どう作るか（実装仕様）」。
DDL・擬似コード・状態機械・EBNF・フィールド単位契約・受け入れ基準 (Gherkin) で記述。

| # | ファイル | 内容 |
|---|---|---|
| M1 | [message-engine](slack-spec/impl/M1-message-engine.md) | 書き込みパス完全定義: SQL DDL、ts 採番アルゴリズム（2方式）、投稿トランザクション、冪等性、編集/削除の状態遷移、unfurl 後付け、スレッドのエッジケース、障害時挙動 |
| M2 | [mrkdwn-grammar](slack-spec/impl/M2-mrkdwn-grammar.md) | mrkdwn の EBNF、エスケープ規則、エンティティ文法、書式境界規則、オートリンク、絵文字解決アルゴリズム、rich_text 完全スキーマ、レンダリング規定、テストベクタ |
| M3 | [wire-protocol](slack-spec/impl/M3-wire-protocol.md) | boot 応答スキーマ、WS フレーム単位契約、接続状態機械、gap-fill 手順、楽観送信 outbox、typing/presence の数値規定、イベント適用アルゴリズム |
| M4 | [api-contracts](slack-spec/impl/M4-api-contracts.md) | コア API のフィールド単位契約: 全引数・バリデーション順序・全エラーコード・応答スキーマ、レート制限実装、Events API 配信契約 |
| M5 | [unread-notification-algorithms](slack-spec/impl/M5-unread-notification-algorithms.md) | 未読/メンション数の計算式、通知判定の決定的関数（decide）、配信ルーティング、プッシュペイロード、バッジ定義、DND 判定、受け入れ基準 |
| M6 | [ui-metrics](slack-spec/impl/M6-ui-metrics.md) | デザイントークン（色/タイポ）、レイアウト寸法、メッセージ行構造、集約規則、ホバーバー、コンポーザ、スクロール挙動、アニメーション、a11y |
| M7 | [blockkit-complete](slack-spec/impl/M7-blockkit-complete.md) | Block Kit 網羅: surface×block 配置マトリクス、全ブロック/要素/composition object のフィールド・上限値、インタラクションペイロード契約、バリデーション順序、フォールバック描画 |
| M8 | [search-grammar-ranking](slack-spec/impl/M8-search-grammar-ranking.md) | 検索クエリ EBNF、修飾子の値文法と結合規則（OR/AND/否定）、アナライザ/権限フィルタ、ランキング関数（BM25×鮮度×親和性×反応量）、検索 UI 状態機械 |
| M9 | [huddles-webrtc](slack-spec/impl/M9-huddles-webrtc.md) | Huddle: SFU 構成、ルーム状態機械（LINGER 60s）、シグナリングフレーム、参加シーケンス、simulcast/帯域降格、画面共有・共同描画、文字起こし、障害時挙動 |
| M10 | [canvas-crdt](slack-spec/impl/M10-canvas-crdt.md) | Canvas: 二層 CRDT（ブロック RGA + テキスト）、全 op 定義とマージ規則（LWW/tombstone）、同期プロトコル、スナップショット、権限、コメントのアンカー orphan 化、履歴復元 |
| M11 | [mobile-clients](slack-spec/impl/M11-mobile-clients.md) | iOS/Android: ライフサイクル別接続戦略、プッシュが同期経路になる設計、APNs/FCM 実装差、ナビゲーション、オフライン outbox、EMM、性能目標 |

## 設計上の不可侵原則（これを外すと「Slack ではない」）

1. **チャンネル中心** — 会話の第一級単位は「人」ではなく「チャンネル（トピック）」。
2. **`ts` がメッセージの一意キー兼ソートキー兼スレッド親参照** — チャンネル内で単調増加するマイクロ秒タイムスタンプ文字列。
3. **既読は per-channel の `last_read`** — メッセージ単位の既読フラグは持たない。
4. **スレッドは親メッセージ `ts` への返信** — 別エンティティではない。
5. **保存はテキスト＋構造（mrkdwn / Block Kit）** — WYSIWYG ではない。
6. **参加すれば過去ログ全体が読める** — メール的な「個人の受信箱」モデルではない。
7. **即時配信 + 楽観 UI + 起動スナップショット + 差分イベント** — 「Slack っぽさ」の正体。

> 注: 本書は公開情報・一般的なシステム設計知識に基づく外形的な再現仕様であり、Slack 社の内部実装そのものではありません。
