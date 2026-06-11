# 魔改造設計 X1 — ax base 結合による AI ネイティブ・ナレッジファブリック

> 本書は M1–M11 の Slack クローン仕様を「ax base（会社の中心となる RAG/コンテキスト基盤）の
> 一面」へと作り替える差分設計である。チャットを**永続ログ**から**ナレッジ・ハーベスタ兼
> 消費面**へ転換し、有料 AI「デキスギクン」へのクロスセル動線を構造に埋め込む。

---

## 0. 確定した前提（ユーザー回答に基づく）

1. **ax base** = 会社の中心となる**双方向の共有ナレッジ/RAG/コンテキスト基盤**。全製品の土台。
2. その上に **デキスギクン（有料 AI）**・**このチャット（無料・入口）**・**並行コーディング
   セッション**が「面」として乗る。3者は ax base を介して結合する。
3. **クロスセル**: チャットは無料で広く配り組織ナレッジを ax base に蓄積 →
   蓄積されたナレッジを武器にデキスギクン（有料 AI）へアップセル。**ナレッジ蓄積が
   乗り換え障壁（moat）**になる。

```
   デキスギクン(有料AI)      Chat(無料/入口/採取)      Coding Sessions(並行/エージェント)
        ▲  │                    ▲   │ harvest             ▲   │
        │  ▼ retrieve           │   ▼                     │   ▼
   ┌──────────────────────────  ax base  ──────────────────────────┐
   │  Knowledge Graph + Vector + Decision/QA Store + Provenance + ACL │  ← 会社の中心
   └─────────────────────────────────────────────────────────────────┘
```

---

## 1. 設計原則（不変条件・これを破ると製品が崩壊する）

- **AX-0 単一真実源**: 知識の正本は ax base。チャット/デキスギクン/コードは「面」であり
  知識を二重所有しない。各面は ax base の atom を参照・投影するだけ。
- **AX-1 ACL 忠実性（最重要）**: AI が user U に返す根拠は、**その瞬間に U が閲覧可能な
  ソースに由来する atom のみ**。ax base が全社横断で知識を集約しても、検索・回答は
  毎回 U の権限で再フィルタする。「U に見えないものを AI は決して漏らさない」。
- **AX-2 来歴必須**: すべての knowledge atom は出典（channel + ts + permalink + 由来 ACL）を
  持つ。出典のないナレッジは AI 回答に使わない（幻覚防止＋監査）。
- **AX-3 鮮度整合**: ソースの編集/削除/権限変更は atom へ伝播する（失効・再導出）。
  古い決定は supersede チェーンで明示。
- **AX-4 無摩擦採取**: ナレッジ化はユーザーの追加作業ゼロ。会話するだけで ax base が育つ。
- **AX-5 funnel 内蔵**: 価値の瞬間（AI が良い答えを出す瞬間）をチャット無料面に出し、
  完全版をデキスギクンへ誘導。ゲーティングは体験を壊さず「もっと見たい」を作る。

---

## 2. チャットの役割再定義（魔改造の中核思想）

| 従来 (M1–M11) | 魔改造後 |
|---|---|
| メッセージ＝永続ログの1行 | メッセージ＝**ナレッジ原石**。投稿のたびに ax base へ抽出投入 |
| 検索＝全文検索 (M8) | 検索＝**意味検索＋根拠付き回答**（ax base retrieval を裏に） |
| スレッド＝返信群 | スレッド＝**決定/Q&A の単位**。要約・決定抽出・FAQ 昇格の対象 |
| Canvas＝共同ドキュメント | Canvas＝**ナレッジの結晶化先**。会話 → 自動ドラフト → 正典 |
| bot＝外部連携 | デキスギクン＝**ax base を背負った一級参加者**（後述 §8） |
| 通知 (M5) | 通知＋**プロアクティブ知識介入**（重複作業・陳腐化・未文書化の検知） |

---

## 3. ナレッジ・ハーベスト・パイプライン（M1 ファンアウトの拡張）

M1 §3「コミット後の非同期ファンアウト」に **JOB_HARVEST** を追加する。既存の配信・通知・
インデックスは不変、その隣に採取系を生やす（チャット体験は1msも遅くしない）。

```
post_message COMMIT 後:
  publish(...)            # M1: WS 配信（不変）
  enqueue(JOB_NOTIFY)     # M5: 通知（不変）
  enqueue(JOB_INDEX)      # M8: 全文索引（不変）
  enqueue(JOB_HARVEST, msg)   # ★追加: ナレッジ採取
```

### 3.1 HARVEST ステージ（段階パイプライン）
```
JOB_HARVEST(msg):
  1. gate:        is_knowledge_bearing(msg)?  # 雑談/絵文字のみ/bot ノイズを除外
                  → false なら provenance だけ残して終了（後で再評価可能に）
  2. enrich(LLM): 下記 enrichment スキーマを生成（バッチ・低優先・安価モデル）
  3. atomize:     enrichment から knowledge_atom を生成/更新（決定・Q&A・事実・用語）
  4. resolve_acl: atom の閲覧 ACL を確定（§7。ソース会話の ACL を継承）
  5. embed:       atom 本文をベクトル化（多言語埋め込み）
  6. graph_link:  entity/edge を knowledge graph に upsert
  7. ingest:      ax base へ upsert（§6 の契約）。冪等キー = atom_id + source_rev
```

### 3.2 enrichment スキーマ（メッセージ1件あたりの抽出物）
```jsonc
{
  "source": { "team":"T…","channel":"C…","ts":"…","thread_ts":"…?","author":"U…" },
  "lang": "ja",
  "intent": "question|answer|decision|commitment|announcement|status|chitchat|task",
  "is_knowledge_bearing": true,
  "salience": 0.0..1.0,                 // ナレッジ価値スコア（低いものは atom 化しない）
  "entities": [ { "text":"課金API", "type":"system|project|person|term|metric|doc",
                  "canonical_id":"ent_…?", "confidence":0.0..1.0 } ],
  "relations": [ { "subject":"ent_A","predicate":"depends_on","object":"ent_B" } ],
  "qa": { "is_question": false,
          "answers_ts": ["…"],          // この投稿が回答している質問の ts
          "resolved": true },
  "decision": { "is_decision": true, "statement":"課金APIはv2へ移行する",
                "rationale":"…","owners":["U…"],"supersedes":"dec_…?","scope":"#billing" },
  "commitments": [ { "who":"U…","what":"移行設計を書く","due":"2026-06-20" } ],
  "sensitivity": "normal|pii|secret",   // §12 ガバナンス入力
  "redactions": [ {span,reason} ]
}
```

### 3.3 is_knowledge_bearing / salience の規定
- 安価な分類器で前段ゲート（LLM コスト削減）。`chitchat`・絵文字のみ・`+1`・
  システムメッセージ subtype は salience≈0 → atom 化しない（ただし会話文脈としては保持）。
- 後から閾値変更・モデル改善で**再ハーベスト**できるよう、生メッセージ→enrichment の
  バージョンを記録（`harvest_rev`）。

---

## 4. データモデル魔改造（ax base 側／チャット側の新テーブル）

> shard 規則は M1 を継承（team_id）。ax base 本体は別サービスだが、ここでは論理スキーマを示す。

```sql
-- 知識アトム（ax base の最小単位。1 decision / 1 QA / 1 fact / 1 term）
CREATE TABLE knowledge_atom (
  atom_id        BINARY(16) PRIMARY KEY,
  team_id        BIGINT UNSIGNED,
  kind           ENUM('fact','decision','qa','term','procedure','owner_map'),
  title          VARCHAR(280),
  body           MEDIUMTEXT,              -- 正規化された知識本文（LLM 整形）
  status         ENUM('active','superseded','stale','retracted'),
  superseded_by  BINARY(16) NULL,
  salience       FLOAT,
  acl_ref        BIGINT UNSIGNED,         -- §7 acl_set への参照（実効 ACL の評価対象）
  harvest_rev    INT,                     -- 再ハーベスト世代
  created_at     BIGINT, updated_at BIGINT
);

-- 来歴（atom ←→ ソースメッセージ。多対多。AX-2）
CREATE TABLE atom_provenance (
  atom_id      BINARY(16),
  team_id      BIGINT UNSIGNED,
  channel_key  BIGINT UNSIGNED,
  ts_sec INT UNSIGNED, ts_seq MEDIUMINT UNSIGNED,
  permalink    VARCHAR(512),
  contribution ENUM('primary','supporting'),
  source_acl_ref BIGINT UNSIGNED,         -- そのソース時点の ACL（intersection 計算に使用）
  PRIMARY KEY (atom_id, channel_key, ts_sec, ts_seq)
);

-- ベクトル（ax base のベクトルストアに対応。ここでは参照のみ）
CREATE TABLE atom_embedding (
  atom_id BINARY(16) PRIMARY KEY,
  model   VARCHAR(64), dim INT,
  vector  BLOB,            -- 実体は専用 ANN インデックス(HNSW)へ
  updated_at BIGINT
);

-- ナレッジグラフ
CREATE TABLE kg_entity (
  entity_id BINARY(16) PRIMARY KEY, team_id BIGINT UNSIGNED,
  type ENUM('system','project','person','term','metric','doc'),
  canonical_name VARCHAR(280), aliases JSON, owner_user_key BIGINT UNSIGNED NULL
);
CREATE TABLE kg_edge (
  team_id BIGINT UNSIGNED, subject BINARY(16), predicate VARCHAR(64), object BINARY(16),
  weight FLOAT, atom_id BINARY(16),       -- この関係の根拠 atom
  PRIMARY KEY (team_id, subject, predicate, object)
);

-- 決定台帳 / Q&A 台帳（atom の特化ビュー。検索・UI 用に非正規化）
CREATE TABLE decision_record (
  atom_id BINARY(16) PRIMARY KEY, statement TEXT, rationale TEXT,
  owners JSON, scope VARCHAR(120), decided_at BIGINT, supersedes BINARY(16) NULL
);
CREATE TABLE qa_pair (
  atom_id BINARY(16) PRIMARY KEY, question TEXT, answer TEXT,
  asker_user_key BIGINT UNSIGNED, resolver_user_key BIGINT UNSIGNED,
  confidence FLOAT, canonical BOOL        -- FAQ 昇格済みか
);

-- 採取ジョブの監査
CREATE TABLE harvest_log (
  team_id BIGINT UNSIGNED, channel_key BIGINT UNSIGNED,
  ts_sec INT UNSIGNED, ts_seq MEDIUMINT UNSIGNED,
  harvest_rev INT, model VARCHAR(64), salience FLOAT, atom_ids JSON,
  status ENUM('ok','skipped','error'), at BIGINT,
  PRIMARY KEY (team_id, channel_key, ts_sec, ts_seq, harvest_rev)
);
```

---

## 5. ax base 結合契約（密接かつ完全な結合）

チャット ⇄ ax base は **イベント取り込み（write）** と **検索（read）** の2契約で双方向結合する。
両系とも ax base が単一真実源（AX-0）。

### 5.1 Ingestion（チャット → ax base、write）
```
POST axbase/v1/ingest                    # 冪等。key = atom_id + harvest_rev
{ "op":"upsert|retract|supersede",
  "atom": { …knowledge_atom… },
  "provenance":[ … ], "embedding_request":true,
  "acl": { "acl_ref":…, "principals_snapshot":[…]?},   // §7
  "graph": { "entities":[…], "edges":[…] } }
→ { "ok":true, "atom_id":…, "indexed":true }
```
- ソース編集（M1 §5）/ 削除（M1 §6）/ 権限変更は **CDC（change data capture）**で
  `op:retract|supersede|acl_update` を発火し ax base に伝播（AX-3）。
- 取り込みは非同期・at-least-once。冪等キーで重複吸収。

### 5.2 Retrieval（チャット/デキスギクン → ax base、read）
```
POST axbase/v1/retrieve                   # ★ACL 忠実（§7）。全クエリに as_user 必須
{ "as_user":"U…", "team":"T…",
  "query":"課金APIの移行先は？",
  "mode":"hybrid",                        // vector + bm25 + graph
  "filters": { "kind":["decision","qa"], "freshness":"active" },
  "top_k":12, "need_citations":true }
→ { "atoms":[ { atom, score, citations:[{permalink,ts,channel}], acl_ok:true } ],
    "answer_grounding":[…] }              // デキスギクンが回答合成に使う根拠束
```
- **`as_user` は省略不可**。ax base は as_user の閲覧可能 principal 集合で必ず後段フィルタ。
- mode=hybrid: ベクトル近傍 → BM25 マージ → グラフ拡張（関連 entity 経由）→ ACL フィルタ →
  リランク（M8 のランキング思想を atom レベルに移植：意味類似 × 鮮度 × 親和性 × 権威性）。

---

## 6. ACL 忠実リトリーバル（最難関・最重要 = AX-1）

> 「全社のナレッジを集約」と「各人には見える範囲しか返さない」を両立させる中核。
> ここを誤ると private チャンネル/DM の情報が AI 経由で漏れ、製品が即死する。

### 6.1 ACL の継承と合成
```
atom の実効 ACL = ∩(contributing sources の ACL)        # 最も制限の強いソースに合わせる
  例: #public の事実 + #exec-private の補足 から合成した atom
      → exec-private のメンバーしか見られない（intersection）
分割戦略（推奨）: ACL が異なるソースを混ぜて1 atom にしない。
  異 ACL のソースは別 atom として保持し、retrieval 時に as_user が見られる atom だけ合成。
  （intersection で価値が痩せるより、ACL 同質な atom に割って保持する方が回答が豊か）
```

### 6.2 検索時アルゴリズム（二段ガード）
```
function retrieve(as_user, query, k):
  cand = ann_search(query, k*8) ⊕ bm25(query, k*8) ⊕ graph_expand(query)
  visible = []
  for a in cand:
     # ガード1: atom の acl_ref を「今の」会話 ACL で再評価（メンバーシップは変わる）
     if not can_read(as_user, a.acl_ref): continue
     # ガード2: provenance の各ソースが「今も」as_user に見えるか個別検証
     if not all(can_read_source(as_user, p) for p in primary_provenance(a)): continue
     visible.append(a)
  return rerank(visible)[:k]
```
- **二重検証**（atom ACL ＋ ソース現況）にする理由: チャンネルから kick された直後・
  private 化された直後でも、キャッシュされた atom から漏らさないため。
- `can_read` は M1/M4 のチャンネルメンバーシップ判定を**そのまま再利用**（権限の真実源を
  二重に持たない＝バグの温床を断つ）。
- DM/MPIM 由来の atom は参加者のみ。Slack Connect 外部メンバー由来は組織境界も考慮。

### 6.3 回答合成時の最終ガード
- デキスギクンが回答を作る直前にも、引用する atom 集合を **もう一度 as_user で検証**
  （retrieval と生成の間に権限が変わるレース対策）。
- 引用ゼロになったら「根拠が見つからない/権限により表示できない」と返す（AX-2）。

### 6.4 受け入れ基準（このセクションは特に厳格に）
```gherkin
Scenario: private 由来知識の遮断
  Given #exec-private の決定から atom が作られている
  And   Bob は #exec-private のメンバーでない
  When  Bob がデキスギクンに「その決定は？」と聞く
  Then  回答にその atom は一切使われず、引用も出ない

Scenario: kick 後の即時遮断
  Given Carol が #billing の atom を過去に閲覧でき、回答に使われていた
  When  Carol が #billing から kick される
  Then  直後の同一質問で当該 atom は二段ガードで除外される
```

---

## 7. provenance・鮮度・失効（AX-3）

- **編集**: ソース編集 → 該当 atom を `harvest_rev+1` で再導出（差分が小さければ skip）。
- **削除**: ソース削除 → provenance から除去。primary が全消失した atom は `retract`。
- **権限変更**: チャンネル private 化・メンバー変更 → `acl_update` を ax base へ。
- **決定の陳腐化**: 新しい decision が古い decision を `supersedes` → 旧は `superseded`。
  retrieval は既定で `active` のみ。デキスギクンは「この決定は2026-06-01に更新済み」と注記。
- **stale 検出**: 参照 entity の最終更新からの経過・矛盾する新 atom の出現で `stale` 化、
  プロアクティブ通知（§8）の起点に。

---

## 8. デキスギクンのチャット内サーフェス（無料面に出す価値の瞬間）

デキスギクンは bot ではなく **ax base を背負った一級参加者**。チャット上の現れ方：

| サーフェス | 動作 | 無料/有料 |
|---|---|---|
| **アンビエント回答** | チャンネルの質問(intent=question)を検知し、ax base 根拠付きで回答候補を ephemeral 提示（「この回答を投稿/編集」） | **無料はプレビュー**（要約1–2文＋引用1件）/ 全文・追問はデキスギクン有料 |
| **キャッチアップ** | 「未読をまとめて」→ スレッド/チャンネル要約＋決定/ToDo 抽出 | 無料は1日1回/有料は無制限＋全社横断 |
| **根拠付き検索** | M8 検索バーを「質問で聞ける」化。答え＋出典カード | 無料は同一チャンネル/有料は ax base 全域 |
| **起案** | 「この議論から PRD/ADR/Canvas を起こして」→ 下書き生成（§4 の decision/qa を素材に） | 有料 |
| **プロアクティブ介入** | 重複質問の検知（「同じ質問が#devで既出、回答はこちら」）、陳腐化決定の警告、未文書化の促し | 無料で“匂わせ”、深掘りは有料 |
| **@デキスギクン メンション** | 通常の参加者として会話に呼べる。M3 の WS/通知に乗る | 基本無料/重い agent 実行は有料 |

- アンビエント回答は **M5 の通知パイプラインに `ai_suggestion` チャネルを追加**して配信
  （ephemeral・本人のみ。鳴らさない＝節度を守る）。
- すべての AI 出力に**出典カード**（permalink ジャンプ）を必須化（AX-2、信頼の核）。

---

## 9. 並行コーディングセッションとの双方向結合（MCP ブリッジ）

ax base を共有基盤に、チャットの議論とエージェントの実装を1つのループに閉じる。

```
Chat 議論 ──harvest──▶ ax base ──MCP read──▶ Coding Session(Claude Code 並行セッション)
   ▲                                              │ 実装/PR/ADR
   │ surface（結果通知・要約）                      ▼
   └──────── ax base ◀──write back（決定・設計・PR根拠を atom 化）──────────┘
```

### 9.1 ax base を MCP サーバとして公開（read 面）
- コーディングセッションは MCP 経由で ax base を引く：
  `axbase.search_decisions`, `axbase.glossary(term)`, `axbase.who_owns(system)`,
  `axbase.context_pack(task)`（タスク関連の決定・用語・制約・関係者を1束で返す）。
- **as_user = セッション実行者**で ACL 忠実（§6 をそのまま適用）。エージェントも越権しない。

### 9.2 write back（コード → ax base → チャット）
- PR 説明・ADR・設計判断・コミットメッセージの「なぜ」を **decision/fact atom** として
  ax base へ ingest（§5.1）。→ チャットに「#billing: 移行PRがマージ、決定の根拠はこちら」と
  デキスギクンが要約投稿（surface）。
- これで「会話で決まったこと → 実装 → 結果と理由がまた組織知に還る」循環が完成（AX-0/AX-4）。

### 9.3 チャットからエージェント起動（任意・有料の上澄み）
- スレッドのアクション「これを実装させる」→ context_pack を添えて並行コーディングセッションを
  起票（trigger）。進捗・完了をスレッドに返す。**最強のクロスセル動線**（無料の会話が
  有料のエージェント実行を呼ぶ）。

---

## 10. クロスセル機構（無料チャット → 有料デキスギクン）

### 10.1 funnel の構造
```
獲得: チャット無料（広く配布）→ 会話するほど ax base にナレッジ蓄積（AX-4）
活性: アンビエント回答プレビューで「AI が自社の文脈で答える」価値を体験
転換: プレビュー → 全文/追問/全社横断/起案/エージェント実行 で有料ゲート
定着: ナレッジ蓄積量＝乗り換え障壁。解約すると“会社の脳”を失う構造
```

### 10.2 ゲーティング設計（体験を壊さない）
- **見せて、止める**: 答えの存在と価値（要約＋1引用）は無料で見せ、**全文と深掘りで課金**。
  「答えはあるのに読めない」フラストレーションが転換を駆動。完全ブラックボックスにしない。
- **シート単位の AI 活性化**: チャットは全員無料、デキスギクンは活性シートのみ有料。
  チームの一部から始められる。
- **使用量メータ**: 無料は「AI 回答 N回/日・同一チャンネル限定」。超過・横断で有料。

### 10.3 ナレッジ・カバレッジ・ダッシュボード（営業＆定着の武器）
```
coverage_score(team) = f( 採取済 atom 数, 決定/QA の網羅, entity グラフ密度,
                          回答可能質問率（過去質問のうち ax base で答えられる割合） )
表示: 「御社のナレッジは 62% 捕捉済み。デキスギクン有効化で回答可能質問が 3.1倍」
→ 管理者向けに ROI を可視化し、有料転換の根拠にする。
```

### 10.4 共有アイデンティティ/課金
- チャットとデキスギクンは**同一テナント・同一 SSO/SCIM（M10 admin）**。
- 課金は ax base テナント単位。チャット＝シート無料 or 低額、デキスギクン＝AI 活性シート＋
  使用量。1クリックでチャット管理画面から AI を有効化（フリクションレス）。

---

## 11. ガバナンス・プライバシー・信頼（ハーベストの社会的許可）

- **AX-1 の徹底**（§6）に加え:
- **opt-out**: チャンネル/DM 単位で「ナレッジ採取しない」フラグ。DM は既定 opt-out 推奨。
- **sensitivity 検出**: PII/秘密を enrichment で検出 → atom 化抑制 or マスキング、retrieval 除外。
- **保持整合**: M10 の retention/eDiscovery と atom を連動（ソース削除＝atom 失効）。
- **AI 監査ログ**: デキスギクンが「誰の質問に・どの atom を・どの権限で」使ったか全記録。
  ユーザーは「この回答の根拠を見る」で透明化。
- **学習の境界**: テナントのナレッジを他テナントの回答に使わない（テナント分離）。
  基盤モデルの学習に顧客データを使わない既定。
- **規制**: GDPR の忘れられる権利＝ソース削除で atom も消える設計で担保。

---

## 12. 計測（KPI）

| 層 | 指標 |
|---|---|
| 採取 | atom 生成率、salience 分布、決定/QA 捕捉数、coverage_score |
| 価値 | 回答可能質問率、回答の引用クリック率、回答採用率（投稿された割合） |
| funnel | プレビュー表示→課金ゲート到達→転換率、AI 活性シート率 |
| 定着 | ナレッジ蓄積量 × 解約率の逆相関、デキスギクン WAU、エージェント起票数 |
| 信頼 | ACL 違反 0 件（最重要・常時監視）、根拠なし回答率 ≈ 0 |

---

## 13. フェーズドロールアウト

1. **P0 採取基盤**: JOB_HARVEST + provenance + ACL 継承（回答機能なしでも ax base は育つ）。
   ※ 採取は回答より先。蓄積が moat なので**1日でも早く溜め始める**。
2. **P1 根拠付き検索**: M8 検索を ax base retrieval（§5.2）に接続。ACL 忠実（§6）を完成。
3. **P2 アンビエント回答（無料プレビュー）＋ゲーティング**: funnel 起動。
4. **P3 デキスギクン有料**: キャッチアップ/起案/全社横断/追問。
5. **P4 コーディングセッション双方向（MCP）**: write-back ループとエージェント起票。
6. **P5 プロアクティブ介入・カバレッジ営業ダッシュボード**。

---

## 14. 魔改造が触る既存仕様（差分マップ）

| 既存 | 変更 |
|---|---|
| M1 §3 ファンアウト | `JOB_HARVEST` 追加。編集/削除に CDC で ax base 伝播 |
| M1 §5/§6 編集削除 | atom の再導出/失効をトリガ |
| M3 WS | `ai_suggestion` / `digest_ready` イベント型を追加 |
| M5 通知 | `ai_suggestion` 配信チャネル（ephemeral・非鳴動）を decide() に追加 |
| M5 §2.1 mention materialization | **ACL 評価ロジックを ax base retrieval が再利用**（真実源共有） |
| M8 検索 | 全文検索の上に「質問→根拠付き回答」を重畳。ランキング思想を atom に移植 |
| M7 Block Kit | 出典カード/回答プレビュー/「投稿する」アクションの新ブロックパターン |
| M10 Canvas | 会話→自動 Canvas 起案（atom を素材に）。Canvas も atom ソースに |
| M10 admin | テナント＝ax base 課金単位。AI 活性シート、opt-out、AI 監査ログ |

---

## 15. 一行サマリ

**チャットは無料で配って「会話するだけで会社の脳（ax base）が育つ」装置にし、育った脳を
ACL 忠実に引ける有料 AI（デキスギクン）と並行コーディングセッションへ双方向接続する。
ナレッジ蓄積そのものが乗り換え障壁になり、クロスセルが構造的に成立する。**
