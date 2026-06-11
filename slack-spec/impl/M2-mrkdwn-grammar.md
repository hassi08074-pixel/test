# 実装仕様 M2 — mrkdwn 文法・エンティティ・rich_text 変換の完全定義

> 保存形式（mrkdwn + エンティティ）⇄ 編集形式（rich_text blocks）⇄ 表示（DOM）の
> 3表現の相互変換を、パーサが書ける粒度で定義する。

---

## 1. 三層表現と正本

```
[コンポーザ編集状態]  rich_text ブロック (構造化 AST)
        │ serialize（送信時）
        ▼
[保存]  text: mrkdwn 原文（エンティティエンコード済）  ← 検索・通知・フォールバックの正本
        blocks: [{type:"rich_text", ...}]              ← 表示の正本
        │ render
        ▼
[表示]  DOM（メンション解決・絵文字置換・リンク化済み）
```

規定: **blocks が存在すれば blocks を描画し、text は描画しない**。text は (a) 検索インデックス (b) 通知本文 (c) blocks 非対応面（プッシュ通知、メール）のフォールバックに使う。両者は送信時に必ず同期生成する。

---

## 2. 字句仕様 — エスケープ（最初に適用）

ワイヤ上の `text` では次の 3 文字のみ HTML 風エスケープする（**それ以外はしない**）:

| 文字 | エンコード |
|---|---|
| `&` | `&amp;` |
| `<` | `&lt;` |
| `>` | `&gt;` |

`<` `>` はエンティティ構文のデリミタのため必須。`"` `'` はエスケープしない。
デコードは表示直前に行い、二重エスケープを検知しない（`&amp;amp;` は `&amp;` と表示）。

---

## 3. エンティティ文法（EBNF）

```ebnf
entity        = "<" body ">" ;
body          = user_ent | channel_ent | subteam_ent | special_ent
              | date_ent | url_ent ;

user_ent      = "@" user_id [ "|" label ] ;            (* <@U0123ABC>  <@U0123ABC|alice> *)
channel_ent   = "#" channel_id [ "|" label ] ;         (* <#C024BE91L|general> *)
subteam_ent   = "!subteam^" usergroup_id [ "|" label ];(* <!subteam^S0614TZR7|@eng> *)
special_ent   = "!" ( "here" | "channel" | "everyone" ) [ "|" label ] ;
date_ent      = "!date^" unix_ts "^" date_format
                [ "^" link_url ] "|" fallback_text ;
url_ent       = scheme "://" no_gt_chars [ "|" label ]
              | "mailto:" addr [ "|" label ] ;

user_id       = ("U"|"W") , 8*11 alnum_upper ;
channel_id    = ("C"|"D"|"G") , 8*11 alnum_upper ;
label         = { any_char - ">" } ;
```

### date_ent のフォーマットトークン（受信側 TZ・ロケールで展開）
| トークン | 出力例 (en) |
|---|---|
| `{date_num}` | 2026-06-11 |
| `{date}` | June 11th, 2026 |
| `{date_short}` | Jun 11, 2026 |
| `{date_long}` | Thursday, June 11th, 2026 |
| `{date_pretty}` 系 | 上記 + today/yesterday/tomorrow を相対語に置換 |
| `{time}` | 2:34 PM（24h 設定なら 14:34） |
| `{time_secs}` | 2:34:56 PM |

パース失敗時・不正トークン時は `fallback_text` を表示する。

---

## 4. インライン書式の文法と境界規則

```ebnf
inline   = bold | italic | strike | code | text ;
bold     = "*" content "*" ;
italic   = "_" content "_" ;
strike   = "~" content "~" ;
code     = "`" raw "`" ;
```

**境界規則（これを外すと再現にならない）** — デリミタが書式として有効なのは:
1. 開始デリミタの直前が「行頭・空白・開き括弧類・別デリミタ」かつ直後が非空白。
2. 終了デリミタの直前が非空白、直後が「行末・空白・約物・別デリミタ」。
3. 同一行内で完結（インライン書式は改行を跨がない）。
4. `code` 内は他のすべての書式・エンティティ解釈を停止する（リテラル）。
5. ネスト可: `*_bold italic_*` は両方適用。同種のネストは不可。
6. 数式風 `5*3*2` はルール1違反（`*`の前後が英数字）のため書式化しない。

```ebnf
blockline = codefence | quote | listitem | paragraph ;
codefence = "```" [lang] NL { any } "```" ;       (* lang は保存するが現行表示では未使用 *)
quote     = "&gt;" SP content ;                    (* 行頭 > 1行引用 *)
          | "&gt;&gt;&gt;" rest_of_message ;       (* >>> 以降メッセージ末尾まで全部引用 *)
```

---

## 5. オートリンク（送信時に url_ent へ昇格させる検出規則）

```
scheme 付き  : (https?|ftp)://[^\s<>]+        → そのまま <url>
scheme なし  : (\w[\w-]*\.)+[a-z]{2,}(/[^\s<>]*)? かつ既知TLD → https:// を補い <url|入力原文>
メール       : RFC5322 簡易形 → <mailto:addr|addr>
末尾の約物   : ) ] } . , ; : ! ? は、対応する開き括弧が URL 内に無い限りリンクに含めない
             例: "(see https://x.com/a)" → リンクは https://x.com/a
```

---

## 6. 絵文字ショートコードの解決アルゴリズム

```
function resolve_emoji(name, skin_tone?):
  # 解決順序が仕様。カスタムが標準を上書きできる
  1. team custom emoji 辞書を引く
       → alias なら alias_for を再帰解決 (深さ最大10、循環は失敗扱い)
       → 画像 URL を返す (アニメ GIF は再生)
  2. 標準 Unicode 辞書 (emoji-data 互換) を引く
       → skin_tone 指定 (`:wave::skin-tone-3:` = type 2..6) があれば
         EMOJI MODIFIER FITZPATRICK を合成
       → Unicode 文字列を返す（描画はプラットフォーム絵文字 or Slack 同梱画像）
  3. 未解決 → `:name:` をリテラル表示（エラーにしない）
```

- コンポーザの `:` 補完は **2文字目から** 起動し、前方一致→部分一致の順、最近使用 18 件を先頭に出す。
- 入力確定形 `:smile:` は送信 text にそのまま残る（Unicode に展開しない）。表示時に解決する。
- 絵文字だけのメッセージ（絵文字 1〜23 個・他の文字なし）は **拡大表示（jumbomoji）**: 1個=32px 相当、複数でも通常の約2倍で描画。1つでも文字が混ざれば通常サイズ。

---

## 7. メンション抽出（通知判定への入力）

```
function extract_mentions(text):                 # エンコード済み text に対して実行
  result = {users:set, usergroups:set, here:false, channel:false, everyone:false}
  for m in scan(text, /<(@[UW][A-Z0-9]+|!subteam\^S[A-Z0-9]+|!here|!channel|!everyone)/):
    分類して result へ
  # code span / code block 内のエンティティは送信時にエンコードされない
  # （コンポーザがコード内の @ を補完しない）ため、ここでの除外は不要
  return result
```

---

## 8. rich_text ブロック（編集・表示の AST）の完全スキーマ

```jsonc
{ "type": "rich_text", "block_id": "b1", "elements": [ SECTION... ] }

// SECTION は4種:
{ "type": "rich_text_section",      "elements": [ ELEM... ] }
{ "type": "rich_text_list",         "style": "bullet"|"ordered",
  "indent": 0..8, "offset": 0, "border": 0|1,
  "elements": [ rich_text_section... ] }                 // 1 section = 1 li
{ "type": "rich_text_quote",        "elements": [ ELEM... ] }
{ "type": "rich_text_preformatted", "elements": [ ELEM... ], "border": 0|1 }

// ELEM (葉要素):
{ "type":"text",      "text":"...", "style":{ "bold":bool,"italic":bool,
                                              "strike":bool,"code":bool } }
{ "type":"link",      "url":"...", "text":"...", "unsafe":bool, "style":{...} }
{ "type":"user",      "user_id":"U...", "style":{...} }
{ "type":"usergroup", "usergroup_id":"S..." }
{ "type":"channel",   "channel_id":"C..." }
{ "type":"emoji",     "name":"smile", "skin_tone":2..6, "unicode":"1f604" }
{ "type":"broadcast", "range":"here"|"channel"|"everyone" }
{ "type":"date",      "timestamp":123, "format":"{date_short}", "fallback":"..." }
{ "type":"color",     "value":"#FF0000" }                 // カラーチップ表示
```

### mrkdwn → rich_text 変換規則（決定的であること）
1. メッセージを `codefence` / `>>>quote` / 行 に分割。
2. 連続する `quote` 行は 1 つの `rich_text_quote` に併合。
3. 箇条書きはコンポーザでのみ生成（mrkdwn 由来の `- ` はリスト化**しない**。プレーン文字のまま）。
4. 各行内をトークナイズ: entity → code span → bold/italic/strike の順で左から最長一致。
5. 逆変換（rich_text → mrkdwn）は表の対応で機械的に行い、**round-trip で不変**であること（テスト必須）。

---

## 9. 表示レンダリング規定

| 要素 | 規定 |
|---|---|
| user mention | `@DisplayName` を薄青背景 (#1D9BD11A)・青文字 (#1264A3) のチップで描画。クリック→プロフィール。自分宛は黄背景 (#F2C74466) |
| @here/@channel | 同チップ。自分が対象なら黄背景 |
| channel link | `#channel-name` 青文字。クリックで遷移。非メンバーならプレビュー+参加ボタン |
| link | 青 (#1264A3)、下線なし、hover 下線。`unsafe` は確認ダイアログ |
| code span | 等幅、背景 #1D1C1D0A、枠 1px #1D1C1D21、радиус 3px、文字色 #E01E5A |
| preformatted | 等幅ブロック、背景 #1D1C1D0A、枠、radius 4px、横スクロール、右上に Copy ボタン(hover時) |
| quote | 左ボーダー 4px #DDDDDD、左 padding 12px |
| 改行 | `\n` は `<br>`。連続空行は最大 1 行に圧縮しない（そのまま） |
| 長文 | 行数換算 ~35 行 or 700 文字超で「Show more」折りたたみ |

---

## 10. 受け入れテストベクタ（パーサ実装の検収用）

| 入力 (ワイヤ text) | 期待表示 |
|---|---|
| `*bold*` | **bold** |
| `**bold**` | \*<b>bold</b>\*（外側の * はリテラル。Slack は ** を二重適用しない） |
| `a*b*c` | a\*b\*c（境界規則1違反、書式化しない） |
| `*a _b_ c*` | 全体太字+b斜体 |
| `` `*x*` `` | コードスパン内リテラル `*x*` |
| `&lt;@U123&gt;` 形式の `<@U123>` | @名前チップ |
| `<@U123|al>` | @al（label 優先表示はせず **常に最新の表示名を解決**。label は解決失敗時のみ） |
| `<!date^1718089543^{date_short} at {time}|Jun 11>` | 受信者 TZ で "Jun 11, 2026 at 2:34 PM" |
| `>>>a\nb` | a と b 両行が引用 |
| `:wave::skin-tone-3:` | 👋🏼（合成） |
| `:smile: :smile:`（のみ） | jumbomoji 拡大表示 |
| `5*3*2=30` | リテラルのまま |
| `https://x.com/a, see` | リンクは `,` を含まない |
