# 実装仕様 M10 — Canvas の共同編集セマンティクス（CRDT/OT・権限・埋め込み）

> 目標: 複数人が同時編集してもマージ衝突がユーザーに見えず、
> オフライン編集も自然に合流する。参照設計として「ブロックリスト = RGA 系
> List CRDT、ブロック内テキスト = 同型のシーケンス CRDT」の二層を採用する。

---

## 1. ドキュメントモデル

```
Canvas
 └── Block[]                       ← 順序付きリスト（List CRDT）
      ├── id: BlockId = (replica_id, counter)   ← 不変・全域一意
      ├── type: paragraph | heading1..3 | bullet_list_item | ordered_list_item
      │        | checklist_item | quote | code | divider | image | file_embed
      │        | message_embed | user_mention_chip | channel_chip | table | section_break
      ├── content: Text CRDT（インライン: text/link/mention/emoji + style）
      ├── props: { indent:0..8, checked:bool, lang:string, table:{rows,cols,cells[][]} … }
      └── tombstone: bool
```

- インライン書式は M2 の rich_text スタイル（bold/italic/strike/code）と同一語彙。
- `checklist_item` は props に `{checked, assignee:U…?, due_date?}` を持つ。
- `message_embed` は permalink 参照（本文のコピーではなく live 参照。元削除で「削除済み」表示）。

---

## 2. 操作（ops）の定義

すべての編集は以下の op に正規化される。op は不変ログとして保存（イベントソーシング）:

> **不変条件 CR-0**: すべての op は `(lamport, replica)` を持つ。RGA の挿入順と
> 全 LWW タイブレークはこの対で決定する。これが無い op は収束しない。

```jsonc
// ブロック層（全 op に lamport,replica 必須）
{ "op":"block_insert", "id":BlockId, "after":BlockId|HEAD, "type":"paragraph",
  "lamport":n, "replica":r }                               // 挿入順 RGA のキーは id=(replica,counter)
{ "op":"block_delete", "id":BlockId, "lamport":n, "replica":r }   // delete-vs-move の順序判定に必要
{ "op":"block_move",   "id":BlockId, "after":BlockId|HEAD,
  "lamport":n, "replica":r }                               // ★LWW のため必須（旧版は欠落）
{ "op":"block_set",    "id":BlockId, "key":"type|indent|checked|…", "value":… ,
  "lamport":n, "replica":r }                               // LWW (lamport,replica) 比較
// テキスト層（ブロック内）
{ "op":"text_insert",  "block":BlockId, "id":CharId, "after":CharId|HEAD, "ch":"あ",
  "lamport":n, "replica":r }                               // CharId=(replica,counter)
{ "op":"text_delete",  "block":BlockId, "id":CharId, "lamport":n, "replica":r }
{ "op":"text_format",  "block":BlockId, "range":[CharId,CharId], "style":"bold", "value":true,
  "lamport":n, "replica":r }                               // ★replica 必須（旧版は欠落）。範囲は anchor の CharId 対
```

### マージ規則（決定的であること）
1. **挿入順序 (RGA)**: 同じ `after` に複数 replica が挿入 →
   `(lamport, replica_id)` の降順で直後に並べる。全 replica で同一結果。
2. **char-level delete vs insert**: tombstone 勝ち。**tombstone を anchor に持つ
   `text_insert` は可視**（消えた文字の直後にタイプした文字は見える）。anchor が
   tombstone でも新規 char 自体は tombstone ではない。
3. **block-level delete**: 削除ブロック配下の既存 op は破棄せず hidden 保持し、
   ブロックの undelete（block_set で復活）時に内容ごと戻す。← 旧規則2が char と
   block を混同していたため分離。
4. **set の競合**: 同一 `(id,key)` への並行 set は LWW。`(lamport, replica)` 大きい方が勝つ。
5. **text_format の競合**: **文字×style 単位の LWW**。重なり範囲の各 CharId について、
   同一 style への並行 format は `(lamport, replica)` 大きい方の `value` を採用
   （bold=true と bold=false が衝突しても決定的に1つに収束）。
6. **move の競合**: 同一ブロックへの並行 move は `(lamport, replica)` の LWW。
   move は「delete+insert」ではなく専用 op（重複出現を防ぐ）。
   **⚠ 既知の難所（CR-1）**: 相互参照する move（A を B の後ろへ／同時に B を A の後ろへ）は
   単純な "after"+LWW ではサイクルを生み線形順序が壊れ得る。本実装は
   **Kleppmann の move-tree 方式**（各 move に親子関係＋並行 move の祖先循環検出で
   後着 move を無効化）を採用し、または server-assigned fractional index を
   move のたびに LWW 更新する方式に退避する。pure list-RGA だけでは不十分。
7. **checked の競合**: LWW（最後の操作者が勝ち。トグルのカウンタ加算ではない）。

---

## 3. 同期プロトコル

```
クライアント ⇄ サーバ: 既存 WS 上の canvas_* フレーム
上り: { "type":"canvas_ops", "canvas":"Cv…", "base_seq":1041, "ops":[…] }
下り: { "type":"canvas_ops", "canvas":"Cv…", "seq":1042, "ops":[…], "author":"U…" }
     { "type":"canvas_presence", "canvas":"Cv…",
       "peers":[{"user":"U…","cursor":{block,offset},"selection":{from,to},"color_idx":0..8}] }
```

- サーバは op ログに**全順序 seq** を採番して全購読者へ再配布（自分の op も seq 付きで返る）。
- クライアントは「楽観適用 → サーバ seq 順で並べ直し」。CRDT なので並べ直しても収束。
- スナップショット: 1000 op ごと、またはアイドル 30 秒で materialized document を保存。
  新規参加者は スナップショット + 以降の ops を受信（op 全再生はしない）。
- オフライン編集: ローカルに op を蓄積、再接続で一括送信。CRDT 規則によりマージ。
- presence: カーソル/選択は 100ms デバウンスの揮発配信。peer 色は参加順 9 色循環。

---

## 4. 権限モデル

```
canvas_access:
  channel canvas   : 既定 = チャンネルメンバー全員が編集可
  standalone canvas: owner + 明示共有 {user|channel|team} × {edit|view}
評価順: 明示 deny は無し。max(個人付与, チャンネル経由, チーム既定) を適用
ゲスト: single-channel guest は当該チャンネルの canvas のみ・編集可否は付与に従う
外部 (Connect): 既定 view。edit は明示付与
```
- view-only ユーザーには presence を表示しない（閲覧者数のみ「👁 3」表示）。
- セクションロック: ブロック範囲に `locked_by_role:admin` を設定可能（block_set、編集 op を拒否）。

## 5. Canvas 内コメント
- 任意のテキスト範囲 or ブロックに対しコメントスレッドを作成。
- 実体は **メッセージ基盤を再利用**: 隠しチャンネル相当の comment thread
  （M1 のスレッド仕様準拠）。アンカー = (BlockId, CharId range)。
- アンカー文字列が削除された場合: コメントは「元のテキスト: "…"」付きで orphan 化し
  右マージンからドキュメント末尾の「解決済み」リストへ移動。
- 解決 (resolve) / 再オープン。コメント追加で対象者・購読者に M5 経由の通知。

## 6. 履歴・復元
- バージョン履歴: スナップショット単位で時系列表示（編集者アバター付き）。
- 差分表示: ブロック単位の added/removed/changed をハイライト。
- 復元 = 「過去スナップショットとの差分を打ち消す ops を新規発行」
  （履歴を巻き戻すのではなく前進で戻す。並行編集と矛盾しない）。

## 7. エクスポート/検索
- 検索インデックス: ブロックを連結したプレーンテキストを M8 のインデックスへ
  （doc 単位 + 見出しアンカー）。`is:canvas` 修飾子相当はファイル種別 `has:canvas`。
- エクスポート: Markdown / PDF。message_embed は permalink 行に展開。

## 8. 受け入れ基準
```gherkin
Scenario: 並行挿入の収束
  Given AとBがオフラインで同一位置にそれぞれ "X" と "Y" を挿入した
  When  両者が再接続する
  Then  全クライアントの表示順が一致する（lamport/replica 順）

Scenario: チェックの競合
  Given AとBがほぼ同時に同じチェックボックスを操作した（Aがon、Bがoff）
  Then  lamport が大きい方の状態に全員が収束し、トグルが「2回反転」しない

Scenario: コメントのアンカー消失
  Given "重要" という語にコメントが付いている
  When  他者が "重要" を含む文を削除する
  Then  コメントは消えず、引用付きで解決済みリストに移る
```
