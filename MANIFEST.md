# アプリケーション設計書（MANIFEST）

> 本書はプロジェクトのアプリケーション設計の **叩き台** である。各項目は **概要・骨子** レベルにとどめ、詳細仕様（API リクエスト／レスポンス例の全項目、テーブル定義の全カラム等）は別紙にて段階的に詳細化する。

---

## 1. プロジェクト概要

### 1.1 目的
複数プロジェクトをまたいで、タスクの可視化と「依頼／承諾」フローによる担当責任の明確化を提供する。「付箋を動かす感覚」(R-05) でタスクのステータスを操作でき、管理者向けには進捗・負荷を一覧把握できるダッシュボード (R-11) を提供する。

### 1.2 スコープ
- ユーザー管理（招待制アカウント作成、論理削除）
- プロジェクト管理（作成・編集・アーカイブ、メンバー管理）
- タスクのカンバン（`TODO` / `IN_PROGRESS` / `DONE` の 3 列固定）
- タスクの依頼・承諾フロー（`UNASSIGNED` ↔ `REQUESTED` ↔ `ACCEPTED`）
- タスクへのコメント（プレーンテキスト）
- メール通知（承諾依頼など）
- 管理者ダッシュボード

### 1.3 設計上の主要決定
- **アーキテクチャ**: Monorepo（pnpm workspaces + Turborepo）、SPA + REST API の 2 層構成、同一オリジン運用、BFF なし
- **タイムゾーン**: BE / FE / DB すべて `Asia/Tokyo`（JST）で統一、ISO 8601 + `+09:00`
- **主キー**: 全エンティティで `cuid()` を採用
- **認証**: セッション認証（HTTP-only Cookie、PostgreSQL セッションストア）。JWT 不採用、Redis 不採用
- **型安全性**: `packages/shared` の Zod スキーマを Single Source of Truth とし、BE/FE 双方で利用

---

## 2. ユーザーロール定義 (RBAC)

### 2.1 ロール一覧

| ロール | コード | 概要 |
|--------|--------|------|
| 管理者 | `ADMIN` | システム全体の管理権限。ユーザー招待、プロジェクト作成、ダッシュボード閲覧、タスクの物理削除等が可能。 |
| メンバー | `MEMBER` | 所属プロジェクト内でのタスク操作、コメント、依頼・承諾を行う一般ユーザー。 |

- `User.role` enum: `ADMIN` / `MEMBER`
- `User.status` enum: `ACTIVE` / `INACTIVE`（論理削除時は `INACTIVE`、`deletedAt` を併用）

### 2.2 認可境界（多層防御）
1. **`SessionAuthGuard`** — ログイン済みかつ `status = ACTIVE` のセッションを検証
2. **`RolesGuard`** — ロール要件（例: ADMIN 限定エンドポイント）を検証
3. **`ProjectMemberGuard`** — プロジェクトスコープのリソースに対し `ProjectMember` 所属を確認
4. **Service 層クエリ** — `project: { members: { some: { userId } } }` を必ず付与し、Guard 漏れに対する最終防御線とする

### 2.3 主要操作の権限マトリクス（骨子）

| 操作 | ADMIN | MEMBER（所属プロジェクト内） | MEMBER（非所属） |
|------|:---:|:---:|:---:|
| ユーザー招待 | ○ | × | × |
| プロジェクト作成 | ○ | × | × |
| プロジェクトアーカイブ | ○ | × | × |
| メンバー追加・除外 | ○ | × | × |
| タスク作成・編集 | ○ | ○ | × |
| タスク物理削除 | ○ | × | × |
| タスクの依頼 | ○ | ○ | × |
| 依頼の承諾／辞退 | （被依頼者本人のみ） | （被依頼者本人のみ） | × |
| コメント投稿 | ○ | ○ | × |
| コメント削除 | ○（全件） | ○（本人投稿のみ） | × |
| 管理者ダッシュボード | ○ | × | × |

---

## 3. 機能要件

### 3.1 認証・ユーザー管理
- 招待制によるアカウント作成 (R-12): ADMIN がメールで招待し、トークン経由で初回パスワードを設定
  - 招待トークン: 使い切り、有効期限 7 日、メールアドレス紐付け
  - DB 保存はトークンの **SHA-256 ハッシュのみ**（平文非保存）
- ログイン／ログアウト（セッション認証、Cookie ベース）
- パスワード: bcrypt（コスト係数 12）、NIST 800-63B 準拠（最低 12 文字、複雑性ルール強制なし、定期変更なし）
- ユーザー論理削除（`status = INACTIVE` + `deletedAt`、削除時に全セッション失効）
- 退会済みユーザーの担当タスクとコメントは「退会済みユーザー」表示で残置
- MFA: 初期未導入、データモデル上の拡張余地のみ確保

### 3.2 プロジェクト管理
- 自分が所属するプロジェクトの一覧表示 (R-02)
- プロジェクト作成（ADMIN のみ）、編集、アーカイブ（論理削除、`archivedAt`）
- プロジェクトメンバーの追加・除外
  - 除外時: 担当タスクを `UNASSIGNED` に戻し、依頼中タスクは自動取り下げ、コメントは残置

### 3.3 タスク管理
- カンバン **3 列固定**（`TODO` / `IN_PROGRESS` / `DONE`、カスタム列なし）
- タスクの作成・編集・物理削除（削除は ADMIN 限定）
- 「付箋を動かす感覚」(R-05) のドラッグ＆ドロップによるステータス変更・並び替え
  - 列移動制限なし
  - 並び順は整数 `position` をトランザクション内で更新
  - 並び替え操作のみ楽観ロックを緩めて軽快さを優先
- タスク属性: タイトル、説明、担当者（任意・単一）、期日 `dueDate`（任意、R-11 の負荷集計に活用）
- 初期非対応（YAGNI）: 優先度・タグ・添付・サブタスク

### 3.4 タスクの依頼・承諾フロー
- 状態機械: `UNASSIGNED → REQUESTED → ACCEPTED`
- 遷移操作: **依頼 / 取り下げ / 辞退 / 承諾**
- 担当者の付け替えは一度 `UNASSIGNED` を経由（責任の所在を曖昧にしないため）
- 承諾は **被依頼者本人のみ** 実行可能
- 承諾時の通知メール送信 (R-09) は Outbox パターン + pg-boss により「タスク更新 + 通知レコード作成 + メール送信ジョブ投入」を単一トランザクションに統合

### 3.5 コメント機能 (R-10)
- プレーンテキスト
- 編集: 本人のみ
- 削除: 本人または ADMIN（論理削除、`deletedAt`）
- 初期非対応: メンション、添付、スレッド、Markdown レンダリング

### 3.6 通知
- メール送信のみ（タスク承諾依頼、メンバー追加等）
- 送信履歴は `notifications` テーブルに永続化
- アプリ内通知センターはデータモデルのみ準備し、UI は未提供

### 3.7 管理者ダッシュボード (R-11)
- 対象: ADMIN ロールのみ
- リアルタイム DB クエリで集計（バッチキャッシュなし）
- **進捗指標**: 完了タスク数 / 総数、直近 7 日完了数、期限超過数
- **負荷指標**: `ACCEPTED` 未完了数、`REQUESTED` 未承諾数、期限切迫数
- インデックス: `(projectId, status, position)`、`(assigneeId, status)`、`(dueDate)`

---

## 4. 画面遷移・構成

### 4.1 主要画面一覧

| 画面 | パス | アクセス権 | 概要 |
|------|------|------------|------|
| ログイン | `/login` | 未認証 | メール + パスワードでログイン |
| 招待受諾 | `/invite/:token` | 未認証 | 招待トークンから初回パスワード設定 |
| プロジェクト一覧 | `/projects` | 認証済（所属するもののみ） | R-02 対応 |
| プロジェクト詳細（カンバン） | `/projects/:id` | プロジェクトメンバー | R-05 のドラッグ&ドロップ画面 |
| タスク詳細（モーダル等） | `/projects/:id/tasks/:taskId` | プロジェクトメンバー | 詳細編集・コメント・依頼操作 |
| プロジェクト設定 | `/projects/:id/settings` | ADMIN | メンバー管理・編集・アーカイブ |
| 管理者ダッシュボード | `/admin/dashboard` | ADMIN | R-11 |
| ユーザー管理 | `/admin/users` | ADMIN | 招待・無効化 |
| アカウント設定 | `/me` | 認証済 | パスワード変更・プロフィール |

### 4.2 グローバルレイアウト
- **ヘッダー**: ロゴ、現在のプロジェクト切替、ユーザーメニュー、ログアウト
- **サイドバー**: メインナビゲーション（管理メニューは ADMIN のみ表示）
- **メインエリア**: 各ルートのコンテンツ
- **トースト**: 操作結果のフィードバック表示

### 4.3 ルーティング方針
- React Router v6 のネスト型ルーティング
- 認証要否は `<RequireAuth>` / `<RequireRole>` のラッパーで判定
- 未認証アクセス時は `/login` にリダイレクト

---

## 5. データモデリング

### 5.1 エンティティ一覧（骨子のみ。カラム詳細は別紙）

| エンティティ | 主要責務 | 削除戦略 |
|------|------|------|
| `User` | 認証主体・ロール保持 | 論理（`status = INACTIVE` + `deletedAt`） |
| `Invitation` | 招待トークン管理（SHA-256 ハッシュ保存） | 使用後・失効後にハード削除可 |
| `Session` | セッションストア（`connect-pg-simple` 管理） | TTL ベースでハード削除 |
| `Project` | プロジェクト | 論理（`archivedAt`） |
| `ProjectMember` | User × Project の所属関係 | ハード削除（除外時） |
| `Task` | タスク本体 | 物理削除（ADMIN 限定） |
| `Comment` | タスクへのコメント | 論理（`deletedAt`） |
| `Notification` | 通知履歴 / Outbox レコード | 永続保持 |

### 5.2 主キー戦略
- 全エンティティで `cuid()`（String 型）。Prisma デフォルトに準拠し、URL 安全かつ衝突しにくい。

### 5.3 主要 enum 一覧
- `UserRole`: `ADMIN` / `MEMBER`
- `UserStatus`: `ACTIVE` / `INACTIVE`
- `TaskStatus`: `TODO` / `IN_PROGRESS` / `DONE`
- `TaskAssignmentStatus`: `UNASSIGNED` / `REQUESTED` / `ACCEPTED`

### 5.4 主要リレーション（骨子）
- `User` 1 – N `ProjectMember` N – 1 `Project`
- `Project` 1 – N `Task`
- `Task` N – 1 `User`（assignee、Nullable）
- `Task` 1 – N `Comment` N – 1 `User`
- `User` 1 – N `Notification`

### 5.5 並び順
- `Task.position`（整数）。同一 `(projectId, status)` 内で並び順を表現。並び替え／列移動はトランザクション内で再計算。

### 5.6 楽観ロック
- 原則: `updatedAt` 比較で実装
- 例外: `POST /tasks/:id/position` は R-05 の軽快さ優先で緩める

### 5.7 タイムゾーンとデータ型
- PostgreSQL の `TIMESTAMP WITH TIME ZONE` 型を採用
- DB セッション `timezone='Asia/Tokyo'`
- API シリアライズは ISO 8601 + `+09:00`

### 5.8 命名規約
- DB: snake_case（テーブル名・カラム名）
- アプリケーション層: camelCase
- Prisma の `@map` / `@@map` で橋渡し

---

## 6. API 設計

### 6.1 基本方針

REST 形式、ベースパス `/api/v1/`。リソースは複数形 kebab-case (`/projects`, `/tasks`, `/project-members`)。状態遷移は動詞パス (`/tasks/:id/accept` 等)。フィールド命名は API/TS が camelCase、DB が snake_case で Prisma の `@map` / `@@map` で橋渡し。日時は ISO 8601 + `+09:00`、日付のみは `YYYY-MM-DD`。

レスポンスは全て §6.2 の共通エンベロープに包む。バリデーションは `packages/shared` の Zod スキーマを Single Source of Truth とし、BE は `nestjs-zod` の `ZodValidationPipe`、FE は `react-hook-form` + `zodResolver` で利用する。API ドキュメントは `@nestjs/swagger` で OpenAPI を自動生成し `/api/docs` で公開する。

### 6.2 共通レスポンス形式

#### 成功レスポンス

```json
{
  "success": true,
  "data": { /* リソースまたはリスト */ }
}
```

ページネーションを伴うリスト取得では `data` を以下の形式とする。

```json
{
  "success": true,
  "data": {
    "items": [ /* T[] */ ],
    "meta": { "page": 1, "pageSize": 20, "total": 124, "totalPages": 7 }
  }
}
```

#### エラーレスポンス

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "リクエストが不正です",
    "details": { "title": ["必須項目です"] },
    "timestamp": "2026-05-26T10:00:00+09:00"
  }
}
```

`code` は UPPER_SNAKE_CASE。HTTP ステータスコードはエンベロープ外で従来通り設定する。`details` は任意で、フィールド単位の検証エラーや競合中リソースの ID 等を載せる。

### 6.3 共通エラーコード一覧

| HTTP | code | 説明 |
|---|---|---|
| 400 | `VALIDATION_ERROR` | リクエストの形式・値が不正 |
| 401 | `AUTH_REQUIRED` | 認証が必要 (未ログイン) |
| 401 | `INVALID_CREDENTIALS` | 認証情報が不一致 |
| 401 | `SESSION_EXPIRED` | セッション期限切れ |
| 403 | `FORBIDDEN` | 一般的な認可エラー |
| 403 | `CSRF_TOKEN_MISMATCH` | CSRF トークン不一致 |
| 403 | `ADMIN_REQUIRED` | ADMIN ロール必須 |
| 403 | `PROJECT_MEMBER_REQUIRED` | プロジェクト所属が必要 |
| 403 | `NOT_REQUESTEE` | 自分宛の依頼ではない |
| 403 | `NOT_AUTHOR` | 自分の投稿ではない |
| 403 | `NOT_REQUESTER` | 自分が出した依頼ではない |
| 404 | `USER_NOT_FOUND` / `PROJECT_NOT_FOUND` / `TASK_NOT_FOUND` / `COMMENT_NOT_FOUND` / `INVITATION_NOT_FOUND` | 各種リソース不存在 |
| 409 | `OPTIMISTIC_LOCK_FAILED` | `updatedAt` 不一致 |
| 409 | `EMAIL_ALREADY_EXISTS` | メールアドレス重複 |
| 409 | `ALREADY_MEMBER` | 既にプロジェクトメンバー |
| 409 | `INVITATION_ALREADY_USED` | 招待トークン使用済み |
| 410 | `INVITATION_EXPIRED` | 招待トークン期限切れ |
| 422 | `INVALID_STATE_TRANSITION` | 状態機械上不正な遷移 |
| 422 | `ASSIGNEE_NOT_PROJECT_MEMBER` | 担当者候補が非メンバー |
| 422 | `SELF_FORBIDDEN` | 自分自身を対象にできない操作 |
| 422 | `LAST_ADMIN_PROTECTED` | 最後の ADMIN を降格・削除しようとした |
| 429 | `RATE_LIMIT_EXCEEDED` | レートリミット超過 |
| 500 | `INTERNAL_ERROR` | サーバ内部エラー |

### 6.4 共通リソーススキーマ

各エンドポイントで `data` に格納される代表的なリソース型を本節で定義する。各エンドポイントは「`data` に `User` を返す」のように本節を参照する。

#### `User`

| フィールド | 型 | 説明 |
|---|---|---|
| `id` | `String` | cuid |
| `email` | `String` | メールアドレス |
| `name` | `String` | 表示名 |
| `role` | `"ADMIN" \| "MEMBER"` | ロール |
| `status` | `"ACTIVE" \| "INACTIVE"` | 状態 |
| `avatarUrl` | `String \| null` | 顔写真 URL (R-04) |
| `createdAt` | `String` | ISO 8601 +09:00 |
| `updatedAt` | `String` | 同上 |

#### `UserSummary` (埋め込み簡易形)

| フィールド | 型 | 説明 |
|---|---|---|
| `id` | `String` | - |
| `name` | `String` | - |
| `avatarUrl` | `String \| null` | - |
| `status` | `"ACTIVE" \| "INACTIVE"` | `INACTIVE` は UI で「退会済みユーザー」表示 |

#### `Project`

| フィールド | 型 | 説明 |
|---|---|---|
| `id` | `String` | cuid |
| `name` | `String` | 1〜100 文字 |
| `description` | `String \| null` | 0〜2000 文字 |
| `archivedAt` | `String \| null` | アーカイブ時のタイムスタンプ |
| `createdAt` | `String` | - |
| `updatedAt` | `String` | - |

#### `ProjectMember`

| フィールド | 型 | 説明 |
|---|---|---|
| `user` | `UserSummary` | - |
| `joinedAt` | `String` | 参加日時 |

#### `Task`

| フィールド | 型 | 説明 |
|---|---|---|
| `id` | `String` | cuid |
| `projectId` | `String` | 所属プロジェクト ID |
| `title` | `String` | 1〜200 文字 |
| `description` | `String \| null` | 0〜10000 文字 |
| `status` | `"TODO" \| "IN_PROGRESS" \| "DONE"` | カンバン列 |
| `assignmentStatus` | `"UNASSIGNED" \| "REQUESTED" \| "ACCEPTED"` | 担当の状態機械 |
| `assignee` | `UserSummary \| null` | `ACCEPTED` 時のみ非 null |
| `requestedTo` | `UserSummary \| null` | `REQUESTED` 時のみ非 null |
| `requestedBy` | `UserSummary \| null` | `REQUESTED` 時の依頼者 |
| `dueDate` | `String \| null` | `YYYY-MM-DD` |
| `position` | `Integer` | 列内の並び順 |
| `createdAt` | `String` | - |
| `updatedAt` | `String` | - |

#### `Comment`

| フィールド | 型 | 説明 |
|---|---|---|
| `id` | `String` | cuid |
| `taskId` | `String` | - |
| `author` | `UserSummary` | 投稿者 |
| `body` | `String` | 本文 (論理削除時は空文字) |
| `isEdited` | `Boolean` | 編集済みフラグ |
| `isDeleted` | `Boolean` | 論理削除フラグ |
| `deletedAt` | `String \| null` | - |
| `createdAt` | `String` | - |
| `updatedAt` | `String` | - |

#### `Invitation`

| フィールド | 型 | 説明 |
|---|---|---|
| `id` | `String` | cuid |
| `email` | `String` | 招待先メールアドレス |
| `role` | `"ADMIN" \| "MEMBER"` | 招待時に設定するロール |
| `invitedBy` | `UserSummary` | 招待者 |
| `expiresAt` | `String` | 有効期限 (発行から 7 日) |
| `acceptedAt` | `String \| null` | 受諾日時 |
| `createdAt` | `String` | - |

---

### 6.5 認証 / セッション

#### 6.5.1 ログイン

##### **`POST /api/v1/auth/login`**

- **機能概要:** メールアドレスとパスワードでユーザーを認証し、セッションを発行する。
- **認可:** 未認証可。レートリミット `5 req/min/IP`。

**リクエスト**

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `email` | `String` | ✅ | email 形式 |
| `password` | `String` | ✅ | 12〜128 文字 |

**レスポンス**

- **Success: `200 OK`**
  `data` に `{ user: User, csrfToken: String }` を返す。`Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax` と `Set-Cookie: csrf_token=...; Secure; SameSite=Lax` を併せて発行する。

- **Error Responses:**
  - `400 VALIDATION_ERROR` — フォーマット不正
  - `401 INVALID_CREDENTIALS` — 認証情報不一致、または対象ユーザーが `INACTIVE`
  - `429 RATE_LIMIT_EXCEEDED` — 試行過多

---

#### 6.5.2 ログアウト

##### **`POST /api/v1/auth/logout`**

- **機能概要:** 現在のセッションを破棄する。
- **認可:** 認証必須。

**レスポンス**

- **Success: `204 No Content`**
  本文なし。`session` および `csrf_token` Cookie を失効させる。

- **Error Responses:**
  - `401 AUTH_REQUIRED`

---

#### 6.5.3 現在のセッション情報取得

##### **`GET /api/v1/auth/me`**

- **機能概要:** 認証済みユーザーの情報と CSRF トークンを取得する (ページリロード後の再取得用)。
- **認可:** 認証必須。

**レスポンス**

- **Success: `200 OK`**
  `data` に `{ user: User, csrfToken: String }` を返す。

- **Error Responses:**
  - `401 AUTH_REQUIRED`
  - `401 SESSION_EXPIRED`

---

#### 6.5.4 招待トークン検証

##### **`GET /api/v1/auth/invitations/:token`**

- **機能概要:** 招待 URL から開いたページで、トークンが有効か事前に検証する (R-12)。
- **認可:** 未認証可。レートリミット `10 req/min/IP`。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `token` | `String` | 招待トークン本体 |

**レスポンス**

- **Success: `200 OK`**
  `data` に `{ email: String, role: "ADMIN" \| "MEMBER", expiresAt: String }` を返す。

- **Error Responses:**
  - `404 INVITATION_NOT_FOUND`
  - `409 INVITATION_ALREADY_USED`
  - `410 INVITATION_EXPIRED`

---

#### 6.5.5 招待受諾 (初回パスワード設定)

##### **`POST /api/v1/auth/invitations/:token/accept`**

- **機能概要:** 招待トークンを消化し、ユーザーアカウントを作成してセッションを発行する (R-12)。
- **認可:** 未認証可。レートリミット `5 req/min/IP`。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `token` | `String` | 招待トークン |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | `String` | ✅ | 表示名 (1〜64 文字) |
| `password` | `String` | ✅ | 12〜128 文字 (NIST 800-63B) |

**レスポンス**

- **Success: `201 Created`**
  `data` に `{ user: User, csrfToken: String }` を返す。User を `status=ACTIVE` で新規作成し、招待を `acceptedAt` で消化する。同時に `session` Cookie および `csrf_token` Cookie を発行する。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `404 INVITATION_NOT_FOUND`
  - `409 INVITATION_ALREADY_USED`
  - `409 EMAIL_ALREADY_EXISTS` — 招待メールと同一のアドレスのユーザーが既に存在
  - `410 INVITATION_EXPIRED`

---

### 6.6 自分のアカウント

#### 6.6.1 自分のプロフィール取得

##### **`GET /api/v1/me`**

- **機能概要:** 自分のプロフィール情報を取得する。
- **認可:** 認証必須。

**レスポンス**

- **Success: `200 OK`**
  `data` に `User` を返す。

- **Error Responses:**
  - `401 AUTH_REQUIRED`

---

#### 6.6.2 自分のプロフィール更新

##### **`PATCH /api/v1/me`**

- **機能概要:** 表示名・アバター URL を更新する。
- **認可:** 認証必須。

**リクエスト**

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | `String` |  | 1〜64 文字 |
| `avatarUrl` | `String \| null` |  | 顔写真 URL (R-04) |
| `updatedAt` | `String` | ✅ | 楽観ロック用 |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `User` を返す。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `409 OPTIMISTIC_LOCK_FAILED`

---

#### 6.6.3 自分のパスワード変更

##### **`PATCH /api/v1/me/password`**

- **機能概要:** 現パスワードを確認のうえ新パスワードに変更する。実行後、自セッションを除く他の全セッションを失効させる。
- **認可:** 認証必須。レートリミット `5 req/min/user`。

**リクエスト**

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `currentPassword` | `String` | ✅ | 現パスワード |
| `newPassword` | `String` | ✅ | 12〜128 文字 |

**レスポンス**

- **Success: `204 No Content`**
  本文なし。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `401 INVALID_CREDENTIALS` — 現パスワード不一致
  - `429 RATE_LIMIT_EXCEEDED`

---

### 6.7 ユーザー管理 (ADMIN)

#### 6.7.1 ユーザー一覧

##### **`GET /api/v1/users`**

- **機能概要:** 全ユーザーを一覧する。
- **認可:** ADMIN のみ。

**リクエスト**

- **Query Parameters**

| 名前 | 型 | 必須 | 既定 | 説明 |
|---|---|---|---|---|
| `page` | `Integer` |  | `1` | - |
| `pageSize` | `Integer` |  | `20` | 1〜100 |
| `status` | `"ACTIVE" \| "INACTIVE"` |  | - | 状態フィルタ |
| `role` | `"ADMIN" \| "MEMBER"` |  | - | ロールフィルタ |
| `q` | `String` |  | - | `name` / `email` の部分一致検索 |

**レスポンス**

- **Success: `200 OK`**
  `data` に `{ items: User[], meta }` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED`

---

#### 6.7.2 ユーザー詳細

##### **`GET /api/v1/users/:id`**

- **機能概要:** 特定ユーザーの情報を取得する。
- **認可:** ADMIN のみ。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | ユーザー ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に `User` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED`
  - `404 USER_NOT_FOUND`

---

#### 6.7.3 ユーザー招待 (R-12)

##### **`POST /api/v1/users/invite`**

- **機能概要:** 招待メールを送付する。アカウント本体はこの時点では作成されず、受諾時 (§6.5.5) に作成される。
- **認可:** ADMIN のみ。レートリミット `30 req/min/user`。

**リクエスト**

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `email` | `String` | ✅ | email 形式 |
| `role` | `"ADMIN" \| "MEMBER"` | ✅ | 招待時のロール |

**レスポンス**

- **Success: `201 Created`**
  `data` に `Invitation` を返す。招待トークンを生成し SHA-256 ハッシュのみ DB 保存。原文トークン入りの招待 URL をメール送信 (pg-boss + Outbox)。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `403 ADMIN_REQUIRED`
  - `409 EMAIL_ALREADY_EXISTS` — 既存ユーザーと重複
  - `409 INVITATION_ALREADY_USED` — 同一メール宛の未消化招待が存在 (`details.invitationId` で既存招待 ID を返す)

---

#### 6.7.4 ユーザー更新

##### **`PATCH /api/v1/users/:id`**

- **機能概要:** ユーザーの表示名・ロール・状態を更新する。`status=INACTIVE` への変更は §6.7.5 と同等のクリーンアップを伴う。
- **認可:** ADMIN のみ。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | ユーザー ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | `String` |  | - |
| `role` | `"ADMIN" \| "MEMBER"` |  | - |
| `status` | `"ACTIVE" \| "INACTIVE"` |  | - |
| `updatedAt` | `String` | ✅ | 楽観ロック用 |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `User` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED`
  - `404 USER_NOT_FOUND`
  - `409 OPTIMISTIC_LOCK_FAILED`
  - `422 SELF_FORBIDDEN` — 自分自身を `INACTIVE` にしようとした
  - `422 LAST_ADMIN_PROTECTED` — 最後の ADMIN を降格しようとした

---

#### 6.7.5 ユーザー論理削除

##### **`DELETE /api/v1/users/:id`**

- **機能概要:** ユーザーを論理削除する。単一トランザクション内で以下を実行する: ①`status=INACTIVE`・`deletedAt=now()` ②全セッション失効 ③担当中タスクを `UNASSIGNED` に戻す ④被依頼中の `REQUESTED` タスクを自動取り下げ。コメントは残置 (UI 上「退会済みユーザー」と表示)。
- **認可:** ADMIN のみ。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | ユーザー ID |

**レスポンス**

- **Success: `204 No Content`**
  本文なし。

- **Error Responses:**
  - `403 ADMIN_REQUIRED`
  - `404 USER_NOT_FOUND`
  - `422 SELF_FORBIDDEN`
  - `422 LAST_ADMIN_PROTECTED`

---

### 6.8 招待管理 (ADMIN)

#### 6.8.1 招待一覧

##### **`GET /api/v1/invitations`**

- **機能概要:** 招待レコードを一覧する (未消化・受諾済・期限切れの状態フィルタ可)。
- **認可:** ADMIN のみ。

**リクエスト**

- **Query Parameters**

| 名前 | 型 | 必須 | 既定 | 説明 |
|---|---|---|---|---|
| `status` | `"pending" \| "accepted" \| "expired"` |  | `pending` | - |
| `page` | `Integer` |  | `1` | - |
| `pageSize` | `Integer` |  | `20` | - |

**レスポンス**

- **Success: `200 OK`**
  `data` に `{ items: Invitation[], meta }` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED`

---

#### 6.8.2 招待再送

##### **`POST /api/v1/invitations/:id/resend`**

- **機能概要:** 既存トークンを破棄して新トークンを発行し、有効期限を再設定したうえでメールを再送する。
- **認可:** ADMIN のみ。レートリミット `10 req/min/user`。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | 招待 ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に新しい `expiresAt` を持つ `Invitation` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED`
  - `404 INVITATION_NOT_FOUND`
  - `409 INVITATION_ALREADY_USED`
  - `429 RATE_LIMIT_EXCEEDED`

---

#### 6.8.3 招待取り消し

##### **`DELETE /api/v1/invitations/:id`**

- **機能概要:** 未消化の招待を取り消す (ハード削除)。
- **認可:** ADMIN のみ。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | 招待 ID |

**レスポンス**

- **Success: `204 No Content`**
  本文なし。

- **Error Responses:**
  - `403 ADMIN_REQUIRED`
  - `404 INVITATION_NOT_FOUND`
  - `409 INVITATION_ALREADY_USED` — 受諾済みは取り消し不可

---

### 6.9 プロジェクト

#### 6.9.1 自分の参加プロジェクト一覧 (R-02)

##### **`GET /api/v1/projects`**

- **機能概要:** 自分が `ProjectMember` として所属するプロジェクトを返す。ADMIN であっても本エンドポイントでは「自分が所属するもの」のみを返す (R-02 の「自分が招かれているプロジェクト」のセマンティクスに従う)。
- **認可:** 認証必須。

**リクエスト**

- **Query Parameters**

| 名前 | 型 | 必須 | 既定 | 説明 |
|---|---|---|---|---|
| `includeArchived` | `Boolean` |  | `false` | アーカイブ済みも含めるか |

**レスポンス**

- **Success: `200 OK`**
  `data` に `Project[]` を返す (`updatedAt` 降順、ページネーションなし)。

- **Error Responses:**
  - `401 AUTH_REQUIRED`

---

#### 6.9.2 プロジェクト作成

##### **`POST /api/v1/projects`**

- **機能概要:** 新規プロジェクトを作成する。初期メンバーを同一トランザクションで登録できる。作成者 ADMIN 自身は自動でメンバーには加わらない。
- **認可:** ADMIN のみ。

**リクエスト**

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | `String` | ✅ | 1〜100 文字 |
| `description` | `String \| null` |  | 0〜2000 文字 |
| `memberIds` | `String[]` |  | 初期メンバーの `User.id` 配列 |

**レスポンス**

- **Success: `201 Created`**
  `data` に `Project` を返す。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `403 ADMIN_REQUIRED`
  - `422 USER_NOT_FOUND` — `memberIds` に存在しないユーザー (`details.userIds` で該当 ID を返す)

---

#### 6.9.3 プロジェクト詳細

##### **`GET /api/v1/projects/:id`**

- **機能概要:** プロジェクトの詳細を取得する。
- **認可:** プロジェクトメンバーのみ。ADMIN であっても非メンバーは `PROJECT_MEMBER_REQUIRED` (Service 層クエリの多層防御方針)。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | プロジェクト ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に `Project` を返す。

- **Error Responses:**
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND`

---

#### 6.9.4 プロジェクト更新

##### **`PATCH /api/v1/projects/:id`**

- **機能概要:** プロジェクト名・説明を更新する。
- **認可:** ADMIN かつ プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | プロジェクト ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `name` | `String` |  | 1〜100 文字 |
| `description` | `String \| null` |  | 0〜2000 文字 |
| `updatedAt` | `String` | ✅ | 楽観ロック用 |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Project` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED` / `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND`
  - `409 OPTIMISTIC_LOCK_FAILED`

---

#### 6.9.5 プロジェクトアーカイブ

##### **`POST /api/v1/projects/:id/archive`**

- **機能概要:** プロジェクトを論理削除する (`archivedAt = now()`)。配下のタスクには影響しないが、`GET /projects` の既定一覧からは除外される。
- **認可:** ADMIN かつ プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | プロジェクト ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に `archivedAt` 設定後の `Project` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED` / `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND`
  - `409 CONFLICT` — 既にアーカイブ済み

---

#### 6.9.6 プロジェクトアーカイブ解除

##### **`POST /api/v1/projects/:id/unarchive`**

- **機能概要:** アーカイブ済みプロジェクトを復活させる (`archivedAt = null`)。
- **認可:** ADMIN かつ プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | プロジェクト ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に `Project` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED` / `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND`
  - `409 CONFLICT` — アーカイブされていない

---

### 6.10 プロジェクトメンバー

#### 6.10.1 メンバー一覧

##### **`GET /api/v1/projects/:projectId/members`**

- **機能概要:** プロジェクトに所属するメンバーを一覧する。
- **認可:** プロジェクトメンバーのみ。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `projectId` | `String` | プロジェクト ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に `ProjectMember[]` を返す (`joinedAt` 昇順)。

- **Error Responses:**
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND`

---

#### 6.10.2 メンバー追加

##### **`POST /api/v1/projects/:projectId/members`**

- **機能概要:** プロジェクトに既存ユーザーを追加する。
- **認可:** ADMIN かつ プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `projectId` | `String` | プロジェクト ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `userId` | `String` | ✅ | 追加対象の `User.id` |

**レスポンス**

- **Success: `201 Created`**
  `data` に `ProjectMember` を返す。

- **Error Responses:**
  - `403 ADMIN_REQUIRED` / `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND` / `404 USER_NOT_FOUND`
  - `409 ALREADY_MEMBER`
  - `422 VALIDATION_ERROR` — 対象ユーザーが `INACTIVE`

---

#### 6.10.3 メンバー除外

##### **`DELETE /api/v1/projects/:projectId/members/:userId`**

- **機能概要:** プロジェクトからメンバーを除外する。単一トランザクション内で以下を実行: ①`ProjectMember` をハード削除 ②当該プロジェクト内で担当中のタスクを `UNASSIGNED` に戻す ③被依頼中の `REQUESTED` タスクを自動取り下げ。コメントは残置。
- **認可:** ADMIN かつ プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `projectId` | `String` | プロジェクト ID |
| `userId` | `String` | 除外対象の `User.id` |

**レスポンス**

- **Success: `204 No Content`**
  本文なし。

- **Error Responses:**
  - `403 ADMIN_REQUIRED` / `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND` / `404 USER_NOT_FOUND`
  - `422 SELF_FORBIDDEN` — 自分自身を除外しようとした

---

### 6.11 タスク

> 本節の全エンドポイントは `認証必須` かつ 対象タスクが属するプロジェクトの `メンバー必須` を共通要件とする。担当者の割り当ては必ず §6.11.7 (依頼) → §6.11.9 (承諾) のフローを経由する (`UNASSIGNED → REQUESTED → ACCEPTED`)。直接担当者を指定する経路は提供しない。

#### 6.11.1 プロジェクトのタスク一覧 (カンバン用)

##### **`GET /api/v1/projects/:projectId/tasks`**

- **機能概要:** カンバン表示用のタスク全件を返す。ボードは全件読み込みを想定し、ページネーションは行わない。
- **認可:** プロジェクトメンバーのみ。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `projectId` | `String` | プロジェクト ID |

- **Query Parameters**

| 名前 | 型 | 必須 | 既定 | 説明 |
|---|---|---|---|---|
| `status` | `"TODO" \| "IN_PROGRESS" \| "DONE"` |  | - | 列フィルタ |
| `assigneeId` | `String` |  | - | 担当者フィルタ |
| `assignmentStatus` | `"UNASSIGNED" \| "REQUESTED" \| "ACCEPTED"` |  | - | - |

**レスポンス**

- **Success: `200 OK`**
  `data` に `Task[]` を返す (`status` 昇順、同列内は `position` 昇順)。

- **Error Responses:**
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND`

---

#### 6.11.2 タスク作成

##### **`POST /api/v1/projects/:projectId/tasks`**

- **機能概要:** プロジェクト内に新規タスクを作成する。作成時点では常に `assignmentStatus = UNASSIGNED`。担当者を付ける場合は作成後に §6.11.7 で依頼する。
- **認可:** プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `projectId` | `String` | プロジェクト ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `title` | `String` | ✅ | 1〜200 文字 |
| `description` | `String \| null` |  | 0〜10000 文字 |
| `status` | `"TODO" \| "IN_PROGRESS" \| "DONE"` |  | 既定 `TODO` |
| `dueDate` | `String \| null` |  | `YYYY-MM-DD` |

**レスポンス**

- **Success: `201 Created`**
  `data` に新規 `Task` を返す。`position` は同一 `(projectId, status)` 内の末尾値+1。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 PROJECT_NOT_FOUND`

---

#### 6.11.3 タスク詳細

##### **`GET /api/v1/tasks/:id`**

- **機能概要:** タスクの詳細を取得する。
- **認可:** タスク所属プロジェクトのメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に `Task` を返す。

- **Error Responses:**
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`

---

#### 6.11.4 タスク編集

##### **`PATCH /api/v1/tasks/:id`**

- **機能概要:** タイトル・説明・期日・ステータス (列) を編集する (R-06)。本エンドポイントでは担当者の変更は受け付けない (担当変更は §6.11.7〜6.11.10 の状態遷移エンドポイントを使う)。ドラッグ&ドロップによる列移動・並び替えは §6.11.6 を使うこと。
- **認可:** プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `title` | `String` |  | 1〜200 文字 |
| `description` | `String \| null` |  | 0〜10000 文字 |
| `status` | `"TODO" \| "IN_PROGRESS" \| "DONE"` |  | - |
| `dueDate` | `String \| null` |  | `YYYY-MM-DD` |
| `updatedAt` | `String` | ✅ | 楽観ロック用 |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Task` を返す。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`
  - `409 OPTIMISTIC_LOCK_FAILED`

---

#### 6.11.5 タスク物理削除

##### **`DELETE /api/v1/tasks/:id`**

- **機能概要:** タスクを物理削除する。関連するコメント (論理削除済みを含む) と通知レコードも併せて物理削除する。
- **認可:** ADMIN かつ プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

**レスポンス**

- **Success: `204 No Content`**
  本文なし。

- **Error Responses:**
  - `403 ADMIN_REQUIRED` / `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`

---

#### 6.11.6 タスクの位置変更 / 列移動 (R-05)

##### **`POST /api/v1/tasks/:id/position`**

- **機能概要:** ドラッグ&ドロップによる列移動・並び替えを反映する高頻度エンドポイント。「付箋を動かす感覚」を実現するため、本エンドポイントのみ楽観ロックを適用しない (最終勝者が並びを決定)。`assignmentStatus` は変更しない。
- **認可:** プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `status` | `"TODO" \| "IN_PROGRESS" \| "DONE"` | ✅ | 移動先の列 |
| `position` | `Integer` | ✅ | 0 始まり、移動先列内の挿入位置 |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Task` を返す。同一 `(projectId, status)` 内の `position` をトランザクション内で再計算し、元の列の `position` も詰める。

- **Error Responses:**
  - `400 VALIDATION_ERROR` — `position` が負、`status` が不正
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`

---

#### 6.11.7 タスクを依頼する (R-07)

##### **`POST /api/v1/tasks/:id/request`**

- **機能概要:** 未割り当てのタスクを特定のメンバーに依頼する。状態遷移: `UNASSIGNED → REQUESTED`。タスク更新・通知レコード作成・メール送信ジョブ投入を単一トランザクションで実行 (Outbox + pg-boss)。
- **認可:** プロジェクトメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `requesteeId` | `String` | ✅ | 依頼先の `User.id` |
| `message` | `String \| null` |  | 任意の依頼メッセージ |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Task` を返す (`assignmentStatus=REQUESTED`、`requestedTo`・`requestedBy` 設定済み)。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`
  - `422 INVALID_STATE_TRANSITION` — 現在の `assignmentStatus` が `UNASSIGNED` でない
  - `422 ASSIGNEE_NOT_PROJECT_MEMBER` — `requesteeId` が非メンバー
  - `422 SELF_FORBIDDEN` — 自分自身への依頼

---

#### 6.11.8 依頼の取り下げ

##### **`POST /api/v1/tasks/:id/cancel-request`**

- **機能概要:** 未承諾の依頼を取り下げる。状態遷移: `REQUESTED → UNASSIGNED`。被依頼者に取り下げ通知メールを送信する (Outbox)。
- **認可:** 依頼者本人 (`requestedBy`) または ADMIN。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Task` を返す (`assignmentStatus=UNASSIGNED`、`requestedTo`・`requestedBy` クリア)。

- **Error Responses:**
  - `403 NOT_REQUESTER` — 依頼者本人でも ADMIN でもない
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`
  - `422 INVALID_STATE_TRANSITION` — 現在 `REQUESTED` 状態でない

---

#### 6.11.9 依頼の承諾 (R-08)

##### **`POST /api/v1/tasks/:id/accept`**

- **機能概要:** 依頼を受けた本人のみが実行できる承諾操作。状態遷移: `REQUESTED → ACCEPTED`。単一トランザクションで以下を実行: ①`assignee=自分`、`requestedTo=null` ②`Notification` レコード作成 (依頼者宛、種別 `TASK_ACCEPTED`) ③メール送信ジョブ投入 (R-09)。
- **認可:** 被依頼者本人 (`requestedTo`) のみ。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Task` を返す (`assignmentStatus=ACCEPTED`、`assignee` 設定済み)。

- **Error Responses:**
  - `403 NOT_REQUESTEE` — 自分宛の依頼ではない
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`
  - `422 INVALID_STATE_TRANSITION` — 現在 `REQUESTED` 状態でない

---

#### 6.11.10 依頼の辞退

##### **`POST /api/v1/tasks/:id/decline`**

- **機能概要:** 依頼を受けた本人のみが実行できる辞退操作。状態遷移: `REQUESTED → UNASSIGNED`。依頼者に辞退通知メールを送信する (Outbox)。
- **認可:** 被依頼者本人 (`requestedTo`) のみ。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `reason` | `String \| null` |  | 任意の辞退理由 (依頼者宛メールに含める) |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Task` を返す (`assignmentStatus=UNASSIGNED`)。

- **Error Responses:**
  - `403 NOT_REQUESTEE`
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`
  - `422 INVALID_STATE_TRANSITION`

---

#### 6.11.11 担当の自己解除

##### **`POST /api/v1/tasks/:id/unassign`**

- **機能概要:** 担当者本人または ADMIN が、`ACCEPTED` 状態のタスクの担当を解除する。状態遷移: `ACCEPTED → UNASSIGNED`。担当の付け替えを行う際の経由点 (MANIFEST.md §3.4 「担当者の付け替えは一度 `UNASSIGNED` を経由」)。
- **認可:** 担当者本人 (`assignee`) または ADMIN。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | タスク ID |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Task` を返す (`assignmentStatus=UNASSIGNED`、`assignee=null`)。

- **Error Responses:**
  - `403 FORBIDDEN` — 担当者本人でも ADMIN でもない
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`
  - `422 INVALID_STATE_TRANSITION` — 現在 `ACCEPTED` 状態でない

---

### 6.12 コメント (R-10)

#### 6.12.1 タスクのコメント一覧

##### **`GET /api/v1/tasks/:taskId/comments`**

- **機能概要:** タスクに紐づくコメントを一覧する。論理削除済みも既定で返却し (UI 上「削除されたコメント」と表示)、`body` は空文字に置換する。
- **認可:** タスク所属プロジェクトのメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `taskId` | `String` | タスク ID |

- **Query Parameters**

| 名前 | 型 | 必須 | 既定 | 説明 |
|---|---|---|---|---|
| `page` | `Integer` |  | `1` | - |
| `pageSize` | `Integer` |  | `30` | 1〜100 |
| `includeDeleted` | `Boolean` |  | `true` | 論理削除コメントも返すか |

**レスポンス**

- **Success: `200 OK`**
  `data` に `{ items: Comment[], meta }` を返す (`createdAt` 昇順)。

- **Error Responses:**
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`

---

#### 6.12.2 コメント投稿

##### **`POST /api/v1/tasks/:taskId/comments`**

- **機能概要:** タスクにコメントを投稿する (プレーンテキスト)。
- **認可:** タスク所属プロジェクトのメンバー。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `taskId` | `String` | タスク ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `body` | `String` | ✅ | 1〜5000 文字、プレーンテキスト |

**レスポンス**

- **Success: `201 Created`**
  `data` に新規 `Comment` を返す。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 TASK_NOT_FOUND`

---

#### 6.12.3 コメント編集

##### **`PATCH /api/v1/comments/:id`**

- **機能概要:** コメント本文を編集する。
- **認可:** 投稿者本人のみ (ADMIN でも他人のコメントは編集不可)。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | コメント ID |

- **Body**

| 名前 | 型 | 必須 | 説明 |
|---|---|---|---|
| `body` | `String` | ✅ | 1〜5000 文字 |
| `updatedAt` | `String` | ✅ | 楽観ロック用 |

**レスポンス**

- **Success: `200 OK`**
  `data` に更新後の `Comment` を返す (`isEdited=true`)。

- **Error Responses:**
  - `400 VALIDATION_ERROR`
  - `403 NOT_AUTHOR`
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 COMMENT_NOT_FOUND`
  - `409 OPTIMISTIC_LOCK_FAILED`
  - `409 CONFLICT` — 論理削除済みコメント (`details.reason="ALREADY_DELETED"`)

---

#### 6.12.4 コメント削除 (論理削除)

##### **`DELETE /api/v1/comments/:id`**

- **機能概要:** コメントを論理削除する (`deletedAt` 設定)。レスポンスでは `body=""`、`isDeleted=true` で返される。
- **認可:** 投稿者本人 または ADMIN。

**リクエスト**

- **Path Parameters**

| 名前 | 型 | 説明 |
|---|---|---|
| `id` | `String` | コメント ID |

**レスポンス**

- **Success: `204 No Content`**
  本文なし。

- **Error Responses:**
  - `403 FORBIDDEN` — 投稿者本人でも ADMIN でもない (`details.reason="NOT_AUTHOR_NOR_ADMIN"`)
  - `403 PROJECT_MEMBER_REQUIRED`
  - `404 COMMENT_NOT_FOUND`
  - `409 CONFLICT` — 既に論理削除済み

---

### 6.13 管理者ダッシュボード (R-11)

本節は ADMIN ロール限定。集計はリアルタイム DB クエリで行い、バッチキャッシュは持たない。インデックス `(projectId, status, position)`、`(assigneeId, status)`、`(dueDate)` を前提とする。

#### 6.13.1 全社サマリ

##### **`GET /api/v1/admin/dashboard/summary`**

- **機能概要:** 全社の俯瞰指標を 1 リクエストで返す (進捗・負荷を集約)。
- **認可:** ADMIN のみ。

**リクエスト**

- **Query Parameters**

| 名前 | 型 | 必須 | 既定 | 説明 |
|---|---|---|---|---|
| `includeArchived` | `Boolean` |  | `false` | アーカイブ済みプロジェクトを集計に含めるか |

**レスポンス**

- **Success: `200 OK`**
  `data` に以下のオブジェクトを返す。

| フィールド | 型 | 説明 |
|---|---|---|
| `totals.activeProjects` | `Integer` | アクティブなプロジェクト数 |
| `totals.activeMembers` | `Integer` | `status=ACTIVE` のユーザー数 |
| `totals.totalTasks` | `Integer` | 総タスク数 |
| `totals.doneTasks` | `Integer` | 完了タスク数 |
| `progress.completionRate` | `Number` | 0〜1 |
| `progress.completedLast7Days` | `Integer` | 直近 7 日の完了タスク数 |
| `progress.overdue` | `Integer` | `dueDate < today` かつ `status != DONE` |
| `load.acceptedOpen` | `Integer` | 全社の `ACCEPTED` かつ未完了 |
| `load.requestedPending` | `Integer` | 全社の `REQUESTED` |
| `load.dueSoon` | `Integer` | `dueDate <= today + 3` かつ未完了 |
| `generatedAt` | `String` | 集計時刻 |

- **Error Responses:**
  - `403 ADMIN_REQUIRED`

---

#### 6.13.2 プロジェクト別進捗

##### **`GET /api/v1/admin/dashboard/projects`**

- **機能概要:** プロジェクト単位の進捗・期限超過状況を一覧する。
- **認可:** ADMIN のみ。

**リクエスト**

- **Query Parameters**

| 名前 | 型 | 必須 | 既定 | 説明 |
|---|---|---|---|---|
| `page` | `Integer` |  | `1` | - |
| `pageSize` | `Integer` |  | `20` | - |
| `includeArchived` | `Boolean` |  | `false` | - |
| `sort` | `"name" \| "completionRate" \| "overdue"` |  | `completionRate` | - |
| `order` | `"asc" \| "desc"` |  | `asc` | 既定で進捗の低いものから |

**レスポンス**

- **Success: `200 OK`**
  `data` に `{ items, meta }` を返す。`items` の各要素:

| フィールド | 型 | 説明 |
|---|---|---|
| `project` | `Project` | - |
| `totalTasks` | `Integer` | - |
| `doneTasks` | `Integer` | - |
| `completionRate` | `Number` | 0〜1 |
| `overdue` | `Integer` | - |
| `dueSoon` | `Integer` | - |
| `memberCount` | `Integer` | - |

- **Error Responses:**
  - `403 ADMIN_REQUIRED`

---

#### 6.13.3 メンバー別負荷

##### **`GET /api/v1/admin/dashboard/members`**

- **機能概要:** 各メンバーが抱えているタスクの負荷を一覧する。
- **認可:** ADMIN のみ。

**リクエスト**

- **Query Parameters**

| 名前 | 型 | 必須 | 既定 | 説明 |
|---|---|---|---|---|
| `page` | `Integer` |  | `1` | - |
| `pageSize` | `Integer` |  | `20` | - |
| `sort` | `"acceptedOpen" \| "requestedPending" \| "dueSoon" \| "name"` |  | `acceptedOpen` | - |
| `order` | `"asc" \| "desc"` |  | `desc` | 既定で負荷の重いものから |
| `includeInactive` | `Boolean` |  | `false` | `INACTIVE` ユーザーも含めるか |

**レスポンス**

- **Success: `200 OK`**
  `data` に `{ items, meta }` を返す。`items` の各要素:

| フィールド | 型 | 説明 |
|---|---|---|
| `user` | `UserSummary` | - |
| `acceptedOpen` | `Integer` | `ACCEPTED` かつ未完了 |
| `requestedPending` | `Integer` | `REQUESTED` |
| `overdue` | `Integer` | 期限超過 |
| `dueSoon` | `Integer` | 期限切迫 (3 日以内) |
| `completedLast7Days` | `Integer` | 直近 7 日の完了数 |

- **Error Responses:**
  - `403 ADMIN_REQUIRED`

---

### 6.14 状態機械まとめ

#### `Task.assignmentStatus` の遷移

```
                  POST /tasks/:id/request
       ┌─────────────────────────────────────────┐
       │                                         ▼
   UNASSIGNED ◀──────────────────────────────  REQUESTED
       ▲  ▲                                      │
       │  │                                      │ POST /tasks/:id/accept
       │  │  POST /tasks/:id/cancel-request      │
       │  │  POST /tasks/:id/decline             ▼
       │  └──────────────────────────────────  ACCEPTED
       │                                         │
       └─────────────────────────────────────────┘
                  POST /tasks/:id/unassign
```

- 担当者の付け替えは `ACCEPTED → UNASSIGNED → REQUESTED → ACCEPTED` の経路に限る (責任の所在を曖昧にしないため、必ず `UNASSIGNED` を経由する)。
- `PATCH /tasks/:id` ・ `POST /projects/:projectId/tasks` のいずれも `assigneeId` の直接指定は受け付けない。

#### `Task.status` (R-05)

- 3 列固定 (`TODO`, `IN_PROGRESS`, `DONE`)。
- 列移動・並び替えは `POST /tasks/:id/position` を経由する (列移動制限なし)。
- `PATCH /tasks/:id` でも `status` の単純変更は可能だが、ドラッグ&ドロップ操作では position エンドポイントを使うこと。

---

### 6.15 要求番号と API の対応

| 要求 | 内容 | 関連 API |
|---|---|---|
| R-01 | 複数プロジェクト並行管理 | §6.9 全般 |
| R-02 | 自分が招かれているプロジェクト一覧 | §6.9.1 `GET /projects` |
| R-03 | 進捗の視覚的把握 | §6.11.1 `GET /projects/:projectId/tasks` (`status` フィルタ可) |
| R-04 | 担当者の可視化 (氏名・顔写真) | `Task.assignee` / `UserSummary.avatarUrl` |
| R-05 | D&D による状態変更 | §6.11.6 `POST /tasks/:id/position` |
| R-06 | ポップアップによる軽快な編集 | §6.11.3 `GET /tasks/:id` / §6.11.4 `PATCH /tasks/:id` |
| R-07 | タスク依頼 | §6.11.7 `POST /tasks/:id/request` |
| R-08 | 承諾による意思表示 | §6.11.9 `POST /tasks/:id/accept` |
| R-09 | 承諾時のメール自動通知 | §6.11.9 の副作用 (Outbox + pg-boss) |
| R-10 | タスク内コメント | §6.12 全般 |
| R-11 | 管理者全体俯瞰 | §6.13 全般 |
| R-12 | 招待制によるアクセス制御 | §6.5.4 / §6.5.5 / §6.7.3 / §6.8 |

---

### 6.16 バリデーション・型同期

- `packages/shared` の Zod スキーマを Single Source of Truth とする。
- BE: `nestjs-zod` の `ZodValidationPipe` で入力検証を行い、検証エラーは `400 VALIDATION_ERROR` の `details` にフィールド単位のメッセージ配列を格納する。
- FE: `react-hook-form` + `zodResolver` で同一スキーマを使用する。
- 型整合性は CI で `openapi.json` (Swagger 自動生成) と `packages/shared` の Zod スキーマを照合して検証する。

### 6.17 セキュリティ

- 認証: HTTP-only Cookie のセッション (`Secure`、`SameSite=Lax`)、PostgreSQL セッションストア。
- CSRF: `SameSite=Lax` + Double Submit Cookie パターン。状態変更系リクエストは `X-CSRF-Token` ヘッダの一致を検証する (`csrf_token` Cookie は非 HttpOnly、JS から読める)。
- XSS: React の自動エスケープに加え `helmet` の CSP を適用。ユーザー入力は原則プレーンテキストとして扱う。
- CORS: 本番は同一オリジン、開発のみフロントオリジン (`http://localhost:5173`) を明示的に許可する。
- レートリミット: `@nestjs/throttler` で認証系エンドポイント (`/auth/login`、`/auth/invitations/:token/accept`、`/me/password`、`/users/invite`、`/invitations/:id/resend`) に限定的に適用する。

## 7. 技術スタック

### 7.1 フロントエンド

| 領域 | 採用技術 |
|------|----------|
| 言語 | TypeScript |
| ビルド | Vite |
| UI フレームワーク | React 18 |
| ルーティング | React Router v6 |
| サーバ状態 | TanStack Query |
| クライアント状態 | Zustand |
| スタイリング | Tailwind CSS + shadcn/ui |
| フォーム | react-hook-form + zodResolver |
| ドラッグ&ドロップ | dnd-kit |
| 日時 | date-fns + date-fns-tz |
| ユニットテスト | Vitest + React Testing Library |
| E2E | Playwright |

### 7.2 バックエンド

| 領域 | 採用技術 |
|------|----------|
| 言語 | TypeScript |
| フレームワーク | NestJS 10 |
| ORM | Prisma |
| バリデーション | nestjs-zod（Zod スキーマ） |
| API ドキュメント | @nestjs/swagger（OpenAPI） |
| セッション | express-session + connect-pg-simple |
| 非同期ジョブ | pg-boss（PostgreSQL 駆動） |
| メール | @nestjs-modules/mailer + Nodemailer + Handlebars |
| メール送信基盤 | Amazon SES（本番）／ Mailpit（開発） |
| レートリミット | @nestjs/throttler |
| パスワードハッシュ | bcrypt（コスト係数 12） |
| テスト | Jest |

### 7.3 データベース・インフラ

| 領域 | 採用技術 |
|------|----------|
| RDBMS | PostgreSQL 16（`timezone='Asia/Tokyo'`、`TIMESTAMPTZ`） |
| セッションストア | PostgreSQL（同居） |
| ジョブキュー基盤 | PostgreSQL（pg-boss） |
| Redis | **不採用** |

### 7.4 開発支援

| 領域 | 採用技術 |
|------|----------|
| Monorepo | pnpm workspaces + Turborepo |
| Lint / Format | ESLint + Prettier |
| 型 / スキーマ共有 | TypeScript + Zod（`packages/shared`） |
| Git フック | husky + lint-staged |
| 開発コンテナ | Docker / docker compose（PostgreSQL、Mailpit） |

---

## 8. ディレクトリ構成

機能別（feature-based）構成を採用する。`apps/{web, api}` と `packages/shared` の 3 ワークスペースを基本とする。

```
.
├── apps/
│   ├── web/                          # フロントエンド (React + Vite)
│   │   ├── src/
│   │   │   ├── features/             # 機能単位 (auth, projects, tasks, comments, admin, ...)
│   │   │   │   └── tasks/
│   │   │   │       ├── api/          # TanStack Query フック
│   │   │   │       ├── components/
│   │   │   │       ├── hooks/
│   │   │   │       └── pages/
│   │   │   ├── components/           # 横断 UI (shadcn/ui ラッパー含む)
│   │   │   ├── lib/                  # 横断ユーティリティ
│   │   │   ├── routes/               # React Router 定義
│   │   │   ├── stores/               # Zustand ストア
│   │   │   └── main.tsx
│   │   ├── public/
│   │   └── vite.config.ts
│   │
│   └── api/                          # バックエンド (NestJS)
│       ├── src/
│       │   ├── modules/              # 機能モジュール (auth, users, projects, tasks, comments, notifications, admin)
│       │   │   └── tasks/
│       │   │       ├── tasks.controller.ts
│       │   │       ├── tasks.service.ts
│       │   │       ├── tasks.module.ts
│       │   │       └── dto/
│       │   ├── common/               # ガード, インターセプタ, フィルタ
│       │   ├── infra/                # prisma, mailer, pg-boss, session
│       │   ├── config/
│       │   └── main.ts
│       ├── prisma/
│       │   ├── schema.prisma
│       │   ├── migrations/
│       │   └── seed.ts
│       └── test/
│
├── packages/
│   └── shared/                       # BE/FE 共有 Zod スキーマ・型・定数
│       ├── src/
│       │   ├── schemas/              # Zod スキーマ (機能別)
│       │   ├── types/
│       │   └── constants/
│       └── package.json
│
├── docker-compose.yml                # PostgreSQL, Mailpit
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
└── MANIFEST.md
```

---

## 9. 開発環境構築

### 9.1 前提
- Node.js（LTS 推奨）、pnpm
- Docker / Docker Compose
- Git

### 9.2 初回セットアップ手順（骨子）
1. リポジトリをクローン
2. 依存関係インストール: `pnpm install`
3. 環境変数ファイル `.env` を `apps/api` および `apps/web` に配置（`.env.example` から複製）
4. ローカル PostgreSQL / Mailpit 起動: `docker compose up -d`
5. Prisma マイグレーション適用: `pnpm --filter @app/api prisma migrate dev`
6. シードデータ投入（初期 ADMIN 含む）: `pnpm --filter @app/api prisma db seed`
7. 開発サーバ起動: `pnpm dev`（Turborepo で `web` と `api` を並列起動）

### 9.3 環境変数(骨子)
- **共通**: `NODE_ENV`, `TZ=Asia/Tokyo`
- **API**: `DATABASE_URL`, `SESSION_SECRET`, `COOKIE_DOMAIN`, `MAIL_HOST`, `MAIL_PORT`, `MAIL_FROM`, `APP_BASE_URL`
- **Web**: `VITE_API_BASE_URL`（同一オリジン本番では未設定、開発のみ設定）

### 9.4 起動ポート(既定)
- Web (Vite): `5173`
- API (NestJS): `3000`
- PostgreSQL: `5432`
- Mailpit: `8025` (UI) / `1025` (SMTP)

### 9.5 主要スクリプト(Turborepo タスク)
- `pnpm dev` — 全アプリ並列起動
- `pnpm build` — 全アプリビルド
- `pnpm lint` — 全ワークスペース Lint
- `pnpm test` — 全ワークスペーステスト(Vitest / Jest)
- `pnpm e2e` — Playwright E2E

---

## 10. 将来の拡張（スコープ外）

初期リリースには含めないが、データモデル・API 設計の段階で **拡張余地を確保しておく** 項目。

### 10.1 機能拡張
- **MFA / SSO**: `User` テーブルに認証手段拡張カラムを追加可能な構造とする
- **アプリ内通知センター**: `Notification` テーブルは初期から永続化、UI のみ後付け
- **タスク属性の拡張**: 優先度、タグ、添付ファイル、サブタスク、依存関係
- **コメント機能の拡張**: メンション、Markdown、添付、スレッド、リアクション
- **カンバン列のカスタマイズ**: 現状は固定 3 列だが、列定義テーブル化で拡張可
- **Webhook / 外部連携**: Slack 通知等
- **検索・全文検索**: PostgreSQL の全文検索または OpenSearch 導入
- **国際化 (i18n)**: 現状は JST / 日本語のみ前提

### 10.2 非機能拡張
- **キャッシュ層**: 必要に応じて Redis を追加（現状は不要）
- **読み取り専用レプリカ**: ダッシュボード集計が重くなった場合
- **オブザーバビリティ**: OpenTelemetry、構造化ログ集約、メトリクス基盤
- **CI/CD**: GitHub Actions による Lint / Test / Build / Deploy パイプライン整備
- **監査ログ**: 重要操作の永続記録（現状は通知履歴のみ）

### 10.3 セキュリティ拡張
- パスワードリセットフロー（招待制の現状では緊急度低）
- IP 制限、ジオフェンシング
- 監査用ログイン履歴
- きめ細やかなレートリミット（現状は認証系のみ）

---

> 本書はあくまで叩き台である。レビュー後、各セクションを詳細化した分冊（API 仕様書、データモデル仕様書、画面仕様書、運用設計書）へ展開していく。
