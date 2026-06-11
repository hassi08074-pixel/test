# 仕様検証レポート (VERIFICATION)

対象: `slack-spec/` 機能編 00–12 + 実装編 M1–M11（計24ファイル, 3,942行）
方法: 数値・規則・データフローをファイル横断で突き合わせ、相互参照の整合と内部論理の健全性を検査。
日付: 2026-06-11

---

## サマリ

| 重大度 | 件数 | 状態 |
|---|---|---|
| Critical（実装すると壊れる） | 3 | **修正済** (A,B,C) |
| Major（設計の穴・将来破綻） | 1 | **修正済** (D) |
| Minor（曖昧さ・二重計上リスク） | 1 | **修正済** (E) |
| 整合確認 OK（合格項目） | 7 領域 | 後述 |

> 第2パス（M7/M8/M9/M10 精査）の結果は本書末尾「第2パス精査」に追記（Major 1・Minor 3・難所 1）。

修正は本検証で該当ファイルに直接適用済み。以下は内容と根拠。

---

## A. [Critical/修正済] クライアントがスレッド返信を本流未読に二重計上

- **場所**: `M3 §5 on message()` の `is_countable(ev)` ↔ `M5 §2 unread_count_display`
- **不整合**: M3 の `is_countable` は **subtype だけ**を見る。通常のスレッド返信は
  `subtype=null`（→ countable=true）だが `thread_ts` を持つ。一方 M5 のサーバ式は
  `thread_root IS NULL OR thread_broadcast` で**スレッド返信を本流未読から除外**する。
  → クライアントが返信を受けるたびにチャンネル未読 +1 し、`channel_marked` が来るまで
  サーバ正本と食い違う（バッジがチラつく／過大表示）。
- **修正**: `on message()` で「`thread_ts` あり かつ 非 broadcast」を `is_countable`
  到達前に `return` し、本流カウンタに触れないよう変更。スレッド未読は
  `bump_thread_unread`（購読時のみ）へ分離。is_countable に注記を追加。

## B. [Critical/修正済] 未読クエリの述語が演算子優先順位で誤動作

- **場所**: `M5 §2 unread_count_display`
- **不整合**: `AND m.subtype IN (...) AND m.thread_root IS NULL OR m.subtype='thread_broadcast'`
  は括弧が無く、SQL の `AND > OR` 優先順位により
  `(... AND thread_root IS NULL) OR (subtype='thread_broadcast')` と解釈される。
  → `last_read` 以前・他チャンネル・削除済みを含む**全 thread_broadcast を無条件に計上**。
- **修正**: `AND (m.thread_root IS NULL OR m.subtype='thread_broadcast')` と括弧で閉じ、
  「括弧必須」コメントを付与。

## C. [Critical/修正済] ts 採番が各秒の先頭1件で未定義値を返す

- **場所**: `M1 §2 allocate_ts` 方式A
- **不整合**: `INSERT ... ON DUPLICATE KEY UPDATE next_seq=LAST_INSERT_ID(next_seq+1)`
  の **新規行 INSERT 経路では `LAST_INSERT_ID()` が設定されない**（このテーブルに
  AUTO_INCREMENT 列が無いため、直前の呼び出しのセッション値か 0 が返る）。
  → その秒で最初に投稿されたメッセージの `ts_seq` が不定。I-1（単調増加・一意）を破る恐れ。
- **修正**: `ROW_COUNT()` で分岐（`==1`=新規→seq=1 / `==2`=更新→`LAST_INSERT_ID()`）。
  境界も `seq <= 1000000`（→ 0始まり最大 999999）に整合。

## D. [Major/修正済] `@here` のメンション数が履歴再計算不能

- **場所**: `M5 §2 mention_count`（COUNT(mentions_user)） ↔ `M4 §4 conversations.mark`
  （「unread_count_display/mention_count はサーバ再計算値」）↔ `M3 §6 client.counts`
- **不整合**: `mentions_user` は `@here` を **「配信時に active だったか」** で判定する
  （M5 §規定で「配信時点で固定」と明記）。ところが mention_count は `last_read` 以降を
  対象に**後から SQL で COUNT** する設計。presence は揮発で過去値を保持しないため、
  boot/gap-fill/mark のいずれでも `@here` を含む過去メッセージの mention 数を再現できない。
  3経路で値がdrift する。
- **修正**: `message_mentions` materialization テーブルを M1 §3（投稿 Tx 内）と M5 §2.1 に追加。
  配信時に `mentions_user=true` の受信者を `reason` 付きで展開保存し、`mention_count` は
  この表の COUNT に変更。`@here` は確定した active 集合のみ行を持つ＝後から増えない
  （規定どおり）。これで boot・gap-fill・mark が同一テーブルから決定的に一致する。
  副次効果として mention_count が索引付きクエリ化され高速。

## E. [Minor/修正済] アプリバッジで MPIM を二重計上し得る

- **場所**: `M5 §5 app_badge`
- **不整合**: `Σ_channels mention_count + Σ_DMs/MPIMs unread`。`mention_count` は DM/MPIM 内で
  「全件 mention 扱い」(§2 の `is_im OR is_mpim` 節)。`Σ_channels` が im/mpim を含むと、
  同じ DM/MPIM が両項で加算され**約2倍**になる。
- **修正**: 第1項の集合を `channels(¬im ∧ ¬mpim)` に限定し、「二重計上禁止」を明記。

---

## 合格した整合確認（横断チェック OK）

| 領域 | 確認内容 | 判定 |
|---|---|---|
| `trigger_id` 有効期間 | 09 / M4§11 / M7§4.1 すべて「3秒・1回限り」 | ✅ 一致 |
| `response_url` | 09 / M7 すべて「30分・5回・message起点」 | ✅ 一致 |
| メッセージ text 上限 | 02 / M1§1 / M4§1 すべて 40,000 | ✅ 一致 |
| ファイルアップロード | 08 / 12 / M4§10 すべて getUploadURLExternal→直PUT→complete の3段、旧 files.upload 非推奨 | ✅ 一致 |
| blocks 上限 | 09 / 02 / M1§1 / M4§1 / M7§0 すべて message=50 | ✅ 一致（M7 が modal/home=100 を追加定義、矛盾なし） |
| 編集ウィンドウ | 01(`-1=無制限`) / 03(`0=不可`) / M1§5(`window==-1 OR …<=window*60`) | ✅ 論理一致（0 で常に不可、-1 で常に可） |
| reaction 上限 | M1(ユニーク~50) と M4(`too_many_emoji>50` + `too_many_reactions`=1ユーザー23) | ✅ 別軸の2制限で矛盾なし |
| countable subtype 表 | M3§5 と M5§2 が同一集合を参照（A 修正後は thread 条件も一致） | ✅ 修正後一致 |
| reply_broadcast 前提 | M1§7 / M4§1 ともに `thread_ts` 必須・実体1行表示2箇所 | ✅ 一致 |
| 楽観送信 de-dup | M1(client_msg_id 冪等) / M3§5,§7(client_msg_id 照合) | ✅ 一致 |
| presence_sub 上限 | M3§4(最大500) / M3§9(上限500) | ✅ 一致 |

---

## 残存する設計判断（バグではないが要明示の選択）

1. **`conversations.history` は ts 降順**（M4§2）だが**クライアント store は昇順**（M3§5）。
   レイヤ間で並びが逆 → クライアント受信時に reverse が必要。矛盾ではないが実装者向けに
   M4 へ「クライアントは反転して merge」の一文を足すと親切（未適用・任意）。
2. **`channels.last_read`**（01 のスキーマにカラムとして併記、注釈で「実体は membership 側」）。
   正規化上は `channel_members.last_read` が唯一の真実。01 の二重掲載は注釈付きなので可だが、
   実装時は channels 側カラムを作らないこと（レポートとして明記）。
3. **キーワード一致が mention_count に算入**（M5§2 `keyword_hit`）。Slack 実機もキーワードは
   赤バッジ相当だが、組織によっては「ハイライトのみ・バッジ非加算」を望む場合がある。
   現仕様は加算で固定 — 設定化するなら `keywords_count_as_mention` pref を追加する余地あり。

---

## 第2パス精査（M7 Block Kit / M8 ランキング / M9 / M10 CRDT）

| 重大度 | 件数 | 状態 |
|---|---|---|
| Major | 1 (G) | 修正済 |
| Minor | 3 (F,H,+clarity) | 修正済 |
| 既知の難所として明示 | 1 (CR-1) | 注記追加 |

### F. [Minor/修正済] M7 `option.value` の上限が実 API と不一致
- M7 §1.3 が `value` 上限を 150 と記載。Slack 実 API は **75**。バリデータが本物より
  緩く、本家が弾く payload を通してしまう。→ 75 に訂正。
- 併せて section の `expand` フィールドが実 API に存在しない可能性が高いため
  「本クローンの拡張」と明示（互換厳守なら削除する判断を委ねる）。
- なお header=150 / section text=3000 / button text=75・value=2000・url=3000 /
  confirm 100/300/30/30 / actions 25 / context 10 / checkboxes・radio 10 /
  overflow 2–5 / placeholder 150 は実 API と一致を確認（合格）。

### G. [Major/修正済] M10 の op が LWW タイブレークキーを欠く（CRDT 非収束）
- マージ規則 3〜6 は `(lamport, replica)` の LWW を前提にするが、`block_move` と
  `text_format` の op 定義に **lamport/replica が無かった**。equal-lamport の並行
  move / 並行 format で勝者が非決定 → レプリカ間で表示が分岐し得る。
- 修正: 不変条件 **CR-0「全 op は (lamport, replica) を持つ」** を新設し、
  block_insert/delete/move・text_insert/delete/format すべてにキーを付与。
  text_format の競合解決規則（文字×style 単位 LWW）を新規追記。

### Clarity. [Minor/修正済] M10 規則2 が char 削除と block 削除を混同
- 「tombstone への text_insert は非表示」は char レベルでは誤り（削除直後にタイプした
  文字は**見えなければならない**）。char-level（規則2: insert は可視）と block-level
  （規則3: 配下を hidden 保持し undelete で復活）に分離。

### H. [Minor/修正済] M8 修飾子のみクエリでスコアが全件同点に縮退
- 乗算スコアは BM25 を含むため、自由語ゼロのクエリ（`in:#x has:pdf` 等）で全件 0 点 →
  実質ランダム（ts タイブレークのみ）。→ 自由語ゼロ時は recency×affinity を実効スコアに
  する timestamp フォールバックを明文化。

### CR-1. [既知の難所/注記] M10 の相互参照 move はサイクルを生む
- list-RGA に "after"+LWW で move を載せると、相互 move でサイクル化し線形順序が壊れる
  典型問題。Kleppmann の move-tree（並行 move の祖先循環検出で後着を無効化）採用を
  規則6に明記。「pure list-RGA では不十分」と警告。

### 第2パスで合格（OK）
- M7 のブロック/要素上限値は `option.value` を除き実 API と一致。
- M8 recency は単調減少・下限 0.05（約130日で床）、affinity 各加点は意図的スタック、
  engagement は cap 付き単調 — 単調性に破綻なし。
- M9 Huddle: ルーム状態機械（LINGER 60s）・参加シーケンス・障害遷移に論理矛盾なし
  （`addTransceiver video sendonly×2` は simulcast 層数の表記ゆれのみ＝無害）。

---

## 検証の結論（2パス合計）

総計: **Critical 3 / Major 2 / Minor 4 / 既知の難所 1**。修正必須 9 件すべて適用済み、
難所 1 件は方式を明記。

- **第1パス（コア整合）**: Critical 3件は「そのまま実装するとデータ不整合（未読・ts）を
  生む」実害級。@here の Major は materialization 追加で boot/gap-fill/mark の三経路整合を回復。
- **第2パス（Block Kit・ランキング・CRDT）**: M10 の CRDT が **LWW キー欠落で非収束**だった
  のが最大の発見（Major G）。Block Kit は `option.value` の1値を除き実 API と一致。
  M8 のランキングは単調性に破綻なし、修飾子のみクエリの縮退のみ補正。
- 横断する固定値（3秒・30分・40000・3段アップロード・各種ブロック上限）は全ファイルで一貫し、
  仕様群の内部参照規律は健全と判断する。
- 未検証で残る領域: M11 モバイルのプッシュ取り消しレースの網羅、M9 の再ネゴシエーション
  シーケンスの SDP 詳細、各 Gherkin の機械実行（実コード化での裏取り）。
