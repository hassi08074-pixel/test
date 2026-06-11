# Slack 完全再現仕様書 — 00. 製品概要・設計思想・用語

> 本仕様群は、Slack を「細部まで同一の製品」として再実装するための分析・言語化ドキュメントである。
> 各ファイルはドメイン単位で独立しているが、用語と ID 体系は全ファイルで共通とする。

---

## 0.1 製品の本質定義

Slack は **「永続的・チャンネルベースのチームメッセージング基盤」** であり、本質は以下4要素の合成である。

1. **永続ログ (Persistent Log)** — すべての会話はチャンネル/DM に時系列で永続保存され、後から検索・参照できる。メールのように「個人の受信箱」に閉じず、組織の共有資産になる。
2. **リアルタイム配信 (Real-time Fan-out)** — 投稿は即座に全購読クライアントへ WebSocket でプッシュされる。体感遅延 100ms 未満を目標。
3. **拡張プラットフォーム (Platform)** — Bot・スラッシュコマンド・Incoming/Outgoing Webhook・Block Kit UI・Workflow・Events API により、外部システムを会話の中に統合できる。
4. **組織階層 (Org Hierarchy)** — User → Channel → Workspace → Enterprise Grid (Org) の入れ子構造で、SMB から数十万人規模まで同一モデルでスケールする。

### 設計上の不可侵原則（これを外すと「Slack ではない」）
- **チャンネル中心**: 会話の第一級単位は「人」ではなく「チャンネル（トピック）」。DM は二級。
- **既読は per-channel の last_read タイムスタンプ**で管理し、メッセージ単位の既読フラグは持たない（DM の既読インジケータを除く）。
- **メッセージの一意キーは `ts`（チャンネル内で単調増加するマイクロ秒タイムスタンプ文字列）**。これがメッセージ ID 兼ソートキー兼スレッド親参照キーを兼ねる。
- **スレッドは「親メッセージ ts への返信」**であり、別エンティティではない。
- **絵文字リアクションとカスタム絵文字**が一級市民。
- **WYSIWYG ではなく構造化 (Block Kit / mrkdwn)**。表示はリッチでも、保存はテキスト＋構造。

---

## 0.2 全体アーキテクチャ（再現対象の俯瞰）

```
┌─────────────────────────────────────────────────────────────┐
│ Clients: Web(SPA) / Desktop(Electron) / iOS / Android        │
└───────────────┬──────────────────────────┬──────────────────┘
                │ HTTPS (Web API, REST-ish) │ WSS (RTM / flannel)
                ▼                            ▼
┌──────────────────────────┐   ┌──────────────────────────────┐
│ API Edge (api.slack.com) │   │ Real-time Gateway            │
│  - auth/token            │   │  - WebSocket fan-out          │
│  - rate limit            │   │  - presence                   │
│  - method dispatch       │   │  - "flannel" edge cache       │
└────────────┬─────────────┘   └───────────────┬──────────────┘
             │                                  │
             ▼                                  ▼
┌─────────────────────────────────────────────────────────────┐
│ Core Services                                                │
│  Message Service / Channel Service / User Service /          │
│  Search (indexer) / Notification (push/email) /              │
│  Files / Platform (apps,bots,webhooks) / Workflow / Calls    │
└────────────┬───────────────────────────────┬────────────────┘
             ▼                                ▼
┌──────────────────────┐          ┌───────────────────────────┐
│ Datastore            │          │ Async/Queue               │
│  - Sharded MySQL     │          │  - Job queue (push,email, │
│  - per-team sharding │          │    search index, webhooks)│
│  - Vitess-style      │          │  - Pub/Sub for fan-out    │
│  Object store (files)│          └───────────────────────────┘
│  Search index (Solr/ES)                                       │
│  Cache (memcache/redis)                                       │
└──────────────────────┘
```

### 再現時の技術選択（推奨スタック例）
- **DB**: シャーディングされた MySQL（team_id を shard key）。Slack 実機は Vitess を使用。
- **リアルタイム**: WebSocket ゲートウェイ（Go/Elixir 等）。チャンネル単位の pub/sub（Redis Pub/Sub または NATS）でファンアウト。
- **検索**: Elasticsearch / Solr。メッセージを非同期インデックス。
- **オブジェクトストア**: S3 互換（ファイル本体）。
- **ジョブキュー**: Kafka / SQS + ワーカー（通知・メール・Webhook・インデックス）。
- **キャッシュ**: Memcached（ブートデータ）、Redis（presence, unread counts）。
- **フロント**: SPA（React 相当）。デスクトップは Electron。

---

## 0.3 ID 体系（厳密に踏襲すべき）

すべてのオブジェクトは**型プレフィックス + Base34 風英数字**の不透明 ID を持つ。再現時もこの規約を守ると外部互換性が高い。

| 型 | プレフィックス | 例 | 備考 |
|---|---|---|---|
| Team(Workspace) | `T` | `T024BE7LD` | |
| Enterprise(Org) | `E` | `E012ABC3456` | Grid のみ |
| User | `U` / `W` | `U023BECGF` | `W` は Enterprise グローバルユーザー |
| Bot user | `B` | `B0KL5G2DT` | |
| Public/Private Channel | `C` | `C024BE91L` | |
| DM (1:1) | `D` | `D024BE91L` | |
| Group DM (MPIM) | `G` | `G024BE91L` | multi-party IM |
| Message | `ts` 文字列 | `1633036800.000100` | ID ではなくタイムスタンプ |
| File | `F` | `F024BE7LH` | |
| App | `A` | `A0F7YS25R` | |
| Workflow | `Wf` 内部 | — | |
| Usergroup(@here的なメンション可能グループ) | `S` | `S0614TZR7` | |
| Reminder | `Rm` | — | |
| Call | `R` | `R0E69JAID` | |

> **`ts` の規則**: `"<unix秒>.<6桁マイクロ秒連番>"`。同一チャンネル内で**厳密に単調増加・一意**。ソートはこの文字列の数値比較で行う。メッセージの編集や削除でも `ts` は不変。スレッド返信は親の `ts` を `thread_ts` として持つ。

---

## 0.4 階層モデル

```
Enterprise Grid (Org, E...)         ← 任意。大企業のみ
   └── Workspace / Team (T...)      ← 課金・管理境界。複数可
          ├── User membership       ← User は複数 Workspace に所属可
          ├── Channel (C...)        ← public / private
          │     └── Message (ts)    ← thread を内包
          ├── DM (D...) / MPIM (G...)
          ├── Usergroup (S...)
          ├── Custom Emoji
          ├── App installation
          └── Settings / Roles
```

- **Workspace（旧称 Team）**: 一般ユーザーが体感する「Slack の単位」。URL は `myteam.slack.com`。
- **Enterprise Grid**: 複数 Workspace を束ねる組織レイヤー。共有チャンネル、統一管理、グローバルユーザー (`W...`) を提供。
- **Slack Connect**: 異なる組織間でチャンネル/DM を共有する機構（後述）。

---

## 0.5 主要ユースケース（受け入れ基準の起点）

1. ユーザーがワークスペースに参加 → チャンネル一覧が左サイドバーに出る。
2. チャンネルを開く → 過去ログが時系列で読み込まれ、未読位置に「new messages」境界線。
3. 投稿 → 全メンバーのクライアントに即時表示。投稿者には楽観的 UI（先に表示）。
4. メンション/キーワード → 通知（デスクトップ/モバイルプッシュ/バッジ/メール）。
5. スレッド返信 → 親に返信数・参加者アバター表示、スレッドビューで展開。
6. リアクション、ピン留め、ブックマーク（保存）、編集、削除。
7. ファイル添付、コードスニペット、Canvas。
8. 検索（全文・修飾子付き）。
9. Huddle（軽量音声）/ 通話。
10. アプリ連携（GitHub 通知が channel に流れる、`/poll` 等のコマンド）。

---

## 0.6 用語集（全ファイル共通）

| 用語 | 定義 |
|---|---|
| Workspace / Team | 課金・管理・メンバーシップの境界 |
| Channel | トピック単位の永続会話。public(`C`)/private(`C` but is_private) |
| DM / IM | 1:1 ダイレクトメッセージ (`D`) |
| MPIM / Group DM | 多人数 DM (`G`)。最大9人 |
| Message (`ts`) | 1投稿。テキスト+blocks+attachments+files |
| Thread | 親メッセージへの返信群。`thread_ts` で連結 |
| Reaction | メッセージへの絵文字リアクション |
| Mention | `<@U...>` / `<#C...>` / `<!here>` / `<!channel>` / `<!subteam^S...>` |
| Presence | active / away のオンライン状態 |
| DND | Do Not Disturb（通知抑制時間帯） |
| Star → Saved/Later | 後で見る用ブックマーク（旧 Star、現 Saved/Later） |
| Pin | チャンネルに固定された重要メッセージ |
| Block Kit | 構造化 UI コンポーネント仕様 |
| mrkdwn | Slack 独自 Markdown 方言 |
| Huddle | 軽量な音声/画面共有セッション |
| Canvas | チャンネル/DM に紐づくドキュメント面 |
| Clip | 音声/動画の短い録画メッセージ |
| Slack Connect | 組織間チャンネル/DM 共有 |
| Enterprise Grid | 複数 Workspace を束ねる Org レイヤー |
| Socket Mode | Public URL 不要でイベントを受ける WS 接続方式 |

---

## 0.7 仕様ファイル索引

| ファイル | 内容 |
|---|---|
| `00-overview.md` | 本書。全体像・原則・ID・用語 |
| `01-domain-model.md` | データモデル/DBスキーマ全定義 |
| `02-realtime-messaging.md` | WebSocket・配信・presence・整合性 |
| `03-messaging-features.md` | 投稿/編集/削除/書式/リアクション/ピン/保存 |
| `04-channels-dms-threads.md` | チャンネル/DM/MPIM/スレッド/既読 |
| `05-search.md` | 全文検索・修飾子・ランキング |
| `06-notifications-presence.md` | 通知/プッシュ/メール/DND/ステータス |
| `07-huddles-calls-canvas-clips.md` | 音声通話/Huddle/Canvas/Clip |
| `08-files.md` | アップロード/プレビュー/権限 |
| `09-platform-apps.md` | Bot/コマンド/Webhook/BlockKit/Workflow |
| `10-admin-enterprise-security.md` | 権限/Grid/SSO/SCIM/コンプライアンス |
| `11-clients-ui-ux.md` | レイアウト/ショートカット/テーマ/オンボーディング |
| `12-apis.md` | Web API/Events API/Socket Mode/レート制限 |
