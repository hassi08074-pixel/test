# 実装仕様 M8 — 検索クエリ文法 (EBNF)・正規化・ランキング関数

## 1. クエリ文法 (EBNF)

```ebnf
query        = term { WS term } ;                  (* 暗黙 AND *)
term         = [ "-" ] ( modifier | phrase | word ) ;
phrase       = '"' { char - '"' } '"' ;            (* 完全一致・語順固定 *)
word         = { char - WS - '"' } ;
modifier     = mod_name ":" mod_value ;
mod_name     = "from" | "to" | "in" | "with" | "before" | "after" | "on"
             | "during" | "has" | "is" | "hasmy" ;
mod_value    = quoted_value | bare_value ;
bare_value   = { char - WS } ;
quoted_value = '"' { char - '"' } '"' ;            (* in:"my channel" *)
```

### 修飾子の値文法
```
from:   @display_name | username | <userID解決済み> | me
to:     同上（DM 相手）
in:     #channel-name | @user(=DM) | チャンネル名素片（曖昧時は候補UIで解決）
with:   @user                                  (* DM/MPIM/スレッド参加 *)
before/after/on: YYYY-MM-DD | M/D/YYYY | "Month D, YYYY"
        | today | yesterday | week | month | year   (* 相対語 *)
during: "Month" | "Month YYYY" | YYYY | week | month
has:    link | file | star(=saved) | pin | reaction | :emoji_name:
        | pdf | doc | image | video | snippet | email   (* ファイル種別 *)
is:     thread | dm | saved | pinned | shared
hasmy:  :emoji_name:                           (* 自分が付けたリアクション *)
```

### パース規定
1. トークナイズは左→右。`-` は直後に空白がない場合のみ否定。
2. 不明な修飾子名（`foo:bar`）は**修飾子として扱わず**、リテラル語 "foo:bar" として全文一致に回す。
3. 同種修飾子の複数指定: `in:` `from:` は **OR**（`in:#a in:#b` = a または b）、
   日付系は範囲交差、`has:` `is:` は AND。
4. `before:` と `after:` の矛盾（空区間）は 0 件を返す（エラーにしない）。
5. 値の名前解決（@alice→U…、#general→C…）は検索実行前にクライアントで確定
   （オートコンプリートチップ化）。未解決文字列はサーバ側で best-effort 解決。

---

## 2. インデックス設計

```
index slack_messages:
  doc_id     = team_id + channel_id + ts
  text       : 解析フィールド（下記アナライザ）
  text_exact : phrase 用 (shingle)
  user, channel, team, thread_root : keyword
  ts_epoch   : numeric
  has        : keyword[]  (link/file/reaction/pin/star は更新イベントで再インデックス)
  reactions  : keyword[]  ("thumbsup" …)
  reactors   : keyword[]  (hasmy 用 user_id)
  lang       : 自動判定
analyzer:
  - Unicode 正規化 NFKC → 小文字化
  - 単語分割: 空白系言語=standard、日本語=形態素(kuromoji 相当)+2-gram 併用
  - URL/メールはドメイン単位サブトークン化
  - 絵文字ショートコードは ":name:" を 1 トークン保持
  - エンティティ <@U…> は表示名に展開してインデックス（名前で検索可能に）
権限フィルタ（必須・クエリ時）:
  channel ∈ (public_channels(team) ∪ my_private ∪ my_dms ∪ my_mpims)
  ※ public は「参加していなくても」検索対象。private/DM は本人のみ。
削除/編集: 編集=再インデックス（数秒以内）、削除=即時 doc 削除。
```

---

## 3. ランキング関数（sort=score の参照実装）

```
score(d, q) = BM25(d, q)                       # k1=1.2, b=0.75
            × recency(d)                        # 鮮度減衰
            × affinity(d, searcher)             # 行動親和性
            × engagement(d)                      # 反応量
            × type_boost(d)

recency(d)   = exp( -ln2 × age_days / 30 )      # 半減期30日、下限 0.05
affinity(d)  = 1
             + 0.5·[d.channel ∈ searcher.frequent_channels]   # 直近30日の閲覧上位10
             + 0.4·[d.user == searcher]                        # 自分の発言
             + 0.4·[mentions(d, searcher)]
             + 0.3·[d.channel ∈ searcher.member_channels]
             + 0.3·[is_dm_with(d, searcher)]
engagement(d)= 1 + min(0.3, 0.03·reply_count + 0.02·reaction_total)
type_boost(d)= 1.2 if d.is_thread_parent else 1.0
完全フレーズ一致 (text_exact ヒット) は BM25 項を 1.5 倍。
タイブレーク: ts 降順。
```

- `sort=timestamp` は score を無視し ts 順（フィルタのみ適用）。
- ハイライト: 一致トークン区間を ``〜``（私用領域）で囲んで返し、
  クライアントが `<mark>` 相当（黄背景 #FFF2B8）に変換。

---

## 4. 検索 UI の状態機械

```
[IDLE] --focus--> [SUGGEST]   最近の検索5件 + 修飾子テンプレ + 直近チャンネル
[SUGGEST] --入力--> [TYPEAHEAD] 300ms デバウンスで候補更新:
     行種別: 修飾子補完 / チャンネル / 人 / 「"q" を検索」行
[TYPEAHEAD] --Enter--> [RESULTS]
[RESULTS]: タブ Messages | Files | Channels | People
     左: フィルタ（並び順 / 日付範囲 / チャンネル / 投稿者 / その他のオプション）
     ページング: 20件/ページ、無限スクロール
     0 件: 提案（スペル/修飾子削除の提案リンク）
検索履歴: 直近 5 件をサーバ保存（端末間同期）、個別削除可。
```

### 結果行（Messages タブ）の描画
- グループ化: 同一チャンネル・近接時刻の連続ヒットは1カードにまとめ、
  カードヘッダに `#channel — M月D日`。
- 各ヒットは前後 1 メッセージの文脈付き（`previous`/`next`）。
- アクション: Jump（本流の該当位置へ遷移し2秒黄色フラッシュ）/ スレッドを開く / 共有 / 保存。

---

## 5. 受け入れ基準

```gherkin
Scenario: 未参加 public チャンネルもヒットする
  Given 自分が #random に参加していない（public）
  When  "deploy" を検索する
  Then  #random のメッセージが結果に含まれる

Scenario: private は漏れない
  Given 他人だけが参加する private チャンネルに "secret" がある
  When  "secret" を検索する
  Then  0 件である（存在も示唆しない）

Scenario: 否定修飾子
  When  "release -in:#random" を検索する
  Then  #random のメッセージは含まれない

Scenario: hasmy
  Given 自分が :eyes: を付けたメッセージが3件ある
  When  "hasmy::eyes:" を検索する
  Then  その3件のみが返る
```
