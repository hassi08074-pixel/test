# 実装仕様 M9 — Huddle のメディア基盤・シグナリング・状態同期

> Slack 実機の Huddle はマネージド会議基盤（Amazon Chime SDK 系）上に構築されている。
> 本書は外形的挙動を同一にするための自前 SFU + シグナリングの参照設計を定義する。

---

## 1. アーキテクチャ

```
Client A ──┐  ① シグナリング: 既存 WS (M3) 上の huddle_* フレーム
Client B ──┼──► Huddle Service（ルーム状態・参加権・トークン発行）
Client C ──┘            │ ②ルーム割当
                        ▼
                  SFU クラスタ（リージョン別）
                  - 上り1本/人（simulcast 3層）→ 下り N-1 本選択転送
                  - TURN/STUN 併設（UDP 3478 / TCP+TLS 443 フォールバック）
録音/文字起こしワーカー（SFU から RTP 分岐 → ASR → トランスクリプト）
```

- メディアは **SFU（選択転送）**。MCU 合成はしない（遅延・コスト）。
- コーデック: 音声 Opus 48kHz（DTX/FEC 有効, 目標 24–32kbps）、
  映像 VP8/VP9 simulcast（180p/360p/720p の3層）、画面共有は VP9 高解像度1層+低fps。
- E2E は DTLS-SRTP（ホップ暗号）。

---

## 2. ルームのライフサイクルと状態機械

```
[NONE] --start--> [ACTIVE(participants≥1)] --全員退出--> [LINGER(60s)] --> [ENDED]
                                                  │ 60秒以内に誰か参加
                                                  └──────► ACTIVE に戻る（同一 huddle_id）
```

- Huddle はチャンネル/DM に **同時に1つ**。既に ACTIVE なら「開始」ボタンは「参加」になる。
- ENDED 時: チャンネルに要約システムメッセージ（参加者・時間・スレッドへのリンク）を投稿。
- 参加上限: 50 人。超過は `huddle_full` エラー。

---

## 3. シグナリングプロトコル（WS フレーム）

### 上り（クライアント→サーバ）
```jsonc
{ "type":"huddle_start",  "channel":"C…" }
{ "type":"huddle_join",   "huddle_id":"H…" }
{ "type":"huddle_leave",  "huddle_id":"H…" }
{ "type":"huddle_signal", "huddle_id":"H…",          // SDP/ICE 中継
  "payload": { "kind":"offer"|"answer"|"ice", "sdp"|"candidate":… } }
{ "type":"huddle_state",  "huddle_id":"H…",          // 自分の状態宣言
  "muted":bool, "video":bool, "screen_share":bool, "hand_raised":bool }
{ "type":"huddle_emoji",  "huddle_id":"H…", "name":"tada" }   // 揮発リアクション
```

### 下り（サーバ→全購読者）
```jsonc
{ "type":"huddle_changed", "channel":"C…", "huddle": {
    "id":"H…", "state":"active"|"ended",
    "participants":[ {"user":"U…","muted":true,"video":false,
                      "screen_share":false,"hand_raised":false} ],
    "started_at": 1718089543, "topic":"…" } }
   // チャンネル全員に配信（非参加者のバナー表示用）
{ "type":"huddle_signal", … }                         // 参加者のみ
{ "type":"huddle_emoji", "user":"U…", "name":"tada" } // 参加者のみ
{ "type":"huddle_speaking", "users":["U…"] }          // 発話検出 200ms 間隔・差分時のみ
```

### 参加シーケンス（完全な順序）
```
1. C→S: huddle_join
2. S→C: { sfu_url, room_token(JWT 60s), ice_servers:[turn/stun+資格情報(24h)] }
3. C:   RTCPeerConnection 生成 → addTransceiver(audio sendrecv, video sendonly×2)
4. C→S: huddle_signal(offer)   ── SFU が answer を返す（S 経由）
5. ICE 交換（trickle）。優先: UDP srflx → TURN/UDP → TURN/TCP443
6. DTLS 確立 → 音声送出開始（参加から目標 2 秒以内）
7. S→全: huddle_changed（participants に追加）
失敗時: 10 秒で join タイムアウト → リトライ UI。ICE 全滅 → 「ネットワーク制限」エラー表示。
```

---

## 4. メディア制御の規定

| 項目 | 規定 |
|---|---|
| 入室時マイク | **既定ミュート**（Huddle 開始者のみオン） |
| 発話検出 | RMS + VAD。スピーキングリング（アバター緑枠）を 200ms 粒度で同期 |
| ノイズ抑制 | RNNoise 相当を既定 ON（設定でオフ可） |
| エコー | AEC3 相当必須 |
| 帯域適応 | 受信側 REMB/TWCC。下り逼迫時は 720p→360p→180p→音声のみ の順に降格 |
| 画面共有 | 同時 1 人。後発の開始要求は先行を奪う（先行者に通知）。最大 1080p/15fps |
| 共同描画 | 画面共有上にベクターストローク。データチャnetwork(SCTP) で {path, color, ttl=4s} を中継。4 秒でフェードアウト |
| ビデオタイル | グリッド: 1人=全面 / 2..4=2×2 / 5..9=3×3 / 10+=ページング。発話者優先昇格 |
| ステータス連動 | 参加中 user_profile.status_emoji=:headphones: を自動設定・退出で復元 |
| DND との関係 | Huddle 招待呼び出しは DND 中は鳴らさない（バッジのみ） |

---

## 5. Huddle スレッド（テキスト面）
- Huddle 開始時、チャンネルに `subtype=huddle_thread` の親メッセージを自動投稿。
- 参加者のチャット・共有リンク・絵文字はこのスレッドへの通常メッセージとして保存
  （= 終了後も履歴が残る、M1 のメッセージ仕様をそのまま使用）。
- 終了時に親メッセージを「Huddle 終了・⏱32分・参加者アバター列」に chat.update。

## 6. 文字起こし・要約
- 参加者の発話を ASR ワーカーがリアルタイム転写（言語自動判定、話者=接続単位で確定）。
- ライブキャプション: `huddle_caption {user, text, final:bool}` を参加者へ配信
  （final=false は逐次仮確定、置換表示）。
- 終了後: トランスクリプト全文を Huddle スレッドに添付（canvas 形式）。要約は LLM ジョブで生成し追記。
- プライバシー規定: 文字起こしは **Huddle 開始者がオンにした場合のみ**。オン時は全参加者にバナー表示。

## 7. 障害時挙動
| 障害 | 規定 |
|---|---|
| WS 切断（シグナリングのみ断） | メディアは継続。WS 再接続後に huddle_state を再宣言 |
| メディア断 > 5s | 「再接続中…」トースト + 音声アイコン点滅。ICE restart |
| メディア断 > 30s | 自動退出扱い。participants から除去 |
| SFU ノード障害 | ルームを別ノードへ再割当。クライアントは room_token 再取得→full renegotiation（目標 5 秒以内） |

## 8. 受け入れ基準
```gherkin
Scenario: 最後の1人が抜けても60秒は部屋が残る
  Given Huddle に1人だけ参加している
  When  その1人が退出し、40秒後に別の人が参加ボタンを押す
  Then  同一 huddle_id の部屋に参加し、Huddle スレッドも同一のものが使われる

Scenario: 画面共有の奪い合い
  Given Aが画面共有中
  When  Bが画面共有を開始する
  Then  Aの共有は停止し「Bが共有を開始しました」がAに表示される
```
