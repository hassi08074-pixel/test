# Slack 完全再現仕様書 — 10. 管理 / ロール / Enterprise Grid / SSO / コンプライアンス

## 10.1 ロールと権限

| ロール | 権限 |
|---|---|
| Primary Owner | 最上位。譲渡可。請求/解約/全権 |
| Owner | ほぼ全権（解約除く）。他 Owner 任命可 |
| Admin | メンバー管理、チャンネル管理、設定、App 承認 |
| Member（フル） | 通常利用 |
| Multi-channel Guest | 招待された複数チャンネルのみ |
| Single-channel Guest | 1チャンネルのみ |
| **Custom roles (System roles)** | Grid: チャンネル管理者、ロール管理者、ユーザー管理者など細分化 |
| Bot/App | スコープ準拠 |

- Grid では**System Roles**（Role-based admin）で「監査ログ閲覧のみ」「ユーザー管理のみ」等を細粒度付与。

---

## 10.2 ワークスペース管理（Admin 設定）

主要設定カテゴリ（team_prefs を UI 化）:
- **メンバー管理**: 招待、無効化(deactivate)、ロール変更、ゲスト管理、一括操作、招待リンク/承認制。
- **チャンネル管理**: 作成/アーカイブ/削除権限、命名ポリシー、デフォルトチャンネル、`#general` 投稿制限。
- **メッセージ&ファイル**: 編集/削除許可と猶予、保持ポリシー、ファイルダウンロード制御。
- **認証**: SSO 強制、2FA 必須、セッション期間、許可メールドメイン。
- **App 管理**: 承認制、許可/ブロックリスト、スコープ審査、Webhook 制御。
- **通知/絵文字/Slackbot 応答**のカスタム。
- **エクスポート**: 管理者によるデータエクスポート（public のみ / 全部はプラン依存）。

---

## 10.3 認証・プロビジョニング
- **SSO**: SAML 2.0 / OpenID Connect。IdP（Okta/Entra ID/Google/OneLogin 等）。SSO 強制設定。
- **SCIM 2.0**: ユーザー/グループの自動プロビジョニング・デプロビジョニング（入退社連動）。
- **2FA/MFA**: TOTP/SMS、組織で必須化。
- **セッション管理**: 期限、強制サインアウト、端末一覧、リモートワイプ。
- **ドメイン要求 (Domain Claiming)**: メールドメインを組織が要求し、該当ユーザーを統制。
- **EMM/MDM**: モバイル端末管理連携（未管理端末でのコピー/ダウンロード制限）。

---

## 10.4 Enterprise Grid（大企業レイヤー）

- 複数 **Workspace** を1つの **Organization (E...)** に束ねる。
- **グローバルユーザー (`W...`)**: Org 横断で1アイデンティティ。複数 Workspace に所属。
- **Org 管理コンソール**: 全 Workspace のメンバー/チャンネル/App/セキュリティを一元管理。
- **Org 全体チャンネル / マルチワークスペースチャンネル**: 複数 Workspace にまたがって共有。
- **集中ポリシー**: 認証、保持、DLP、App 許可を Org レベルで強制し Workspace へ継承。
- **IDP/SCIM** は Org レベル。
- **Org 単位の検索/分析/監査ログ**。
- スケール: 数十万〜100万ユーザー級。

---

## 10.5 コンプライアンス & ガバナンス

### 保持ポリシー (Retention)
- メッセージ/ファイルの保持期間を設定（全消去なし/N日後削除/カスタム）。
- チャンネル単位の上書きポリシー（Grid）。
- 削除はジョブで物理削除（eDiscovery 例外あり）。

### eDiscovery / Legal Hold
- サードパーティ DLP/eDiscovery プロバイダ向けの**Discovery API**（メッセージ/編集履歴/ファイルを完全取得、削除済み含む）。
- **Legal Hold**: 特定ユーザー/チャンネルのデータを保持義務化（保持ポリシーより優先、削除されても保全）。

### DLP（Data Loss Prevention）
- メッセージ/ファイルをパターン/コネクタでスキャン、機密検出時にブロック/警告/隔離（tombstone 化）。

### 監査ログ (Audit Logs API)
- ログイン、ファイルダウンロード、チャンネル作成/アーカイブ、権限変更、App インストール等を記録。SIEM 連携。

### 暗号化
- 通信: TLS。保存: AES-256 等で暗号化。
- **Enterprise Key Management (EKM)**: 顧客管理鍵（KMS）でメッセージ/ファイルを暗号化、鍵失効でアクセス遮断・ログ可視化。

### 居住地・認証
- データレジデンシー（地域別保存）。SOC2/ISO27001/HIPAA/FedRAMP 等の準拠（再現プロダクトでは該当する管理策を実装）。

---

## 10.6 分析・運用
- **Analytics ダッシュボード**: アクティブメンバー、メッセージ数、チャンネル別アクティビティ、公開 vs DM 比率、利用率。
- メンバーのアクティビティ、招待状況、App 利用。
- 請求管理（シート課金、Fair Billing で非アクティブ分を自動調整）。

---

## 10.7 再現チェックリスト
- [ ] ロール階層（Primary Owner〜Guest＋System Roles）
- [ ] 管理設定の全カテゴリ（メンバー/チャンネル/メッセージ/認証/App/エクスポート）
- [ ] SAML/OIDC SSO 強制、SCIM、2FA、セッション管理、EMM
- [ ] Enterprise Grid（Org、グローバルユーザー、集中ポリシー継承、マルチWSチャンネル）
- [ ] Retention / Legal Hold / Discovery API / DLP / Audit Logs / EKM
- [ ] Analytics と Fair Billing
