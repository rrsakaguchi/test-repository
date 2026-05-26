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
- REST 形式、ベースパス `/api/v1/`
- リソースは複数形 kebab-case（`/projects`, `/tasks`, `/project-members`）
- 状態遷移は **動詞パス**（後述）
- レスポンスはプレーン JSON、ページネーション必要箇所のみ `{ items, meta }` 形式
- 命名: API/TS は camelCase、DB は snake_case
- 日時: ISO 8601 + `+09:00`
- API ドキュメント: `@nestjs/swagger` で OpenAPI 自動生成、`/api/docs` で Swagger UI 公開

### 6.2 エラーレスポンス共通形式（骨子）

```json
{
  "statusCode": 400,
  "code": "TASK_NOT_REQUESTED",
  "message": "...",
  "details": { },
  "timestamp": "2026-05-26T10:00:00+09:00"
}
```

- `code` は UPPER_SNAKE_CASE（例: `TASK_NOT_REQUESTED`, `PROJECT_MEMBER_REQUIRED`）

### 6.3 主要エンドポイント一覧（概要）

**認証 / セッション**
- `POST /api/v1/auth/login`
- `POST /api/v1/auth/logout`
- `GET  /api/v1/auth/me`
- `POST /api/v1/auth/invitations/:token/accept`（招待受諾＋初期パスワード設定）

**ユーザー（ADMIN）**
- `GET    /api/v1/users`
- `POST   /api/v1/users/invite`
- `PATCH  /api/v1/users/:id`
- `DELETE /api/v1/users/:id`（論理削除）

**プロジェクト**
- `GET   /api/v1/projects`（R-02、所属するもののみ）
- `POST  /api/v1/projects`（ADMIN）
- `GET   /api/v1/projects/:id`
- `PATCH /api/v1/projects/:id`
- `POST  /api/v1/projects/:id/archive`
- `GET   /api/v1/projects/:id/members`
- `POST  /api/v1/projects/:id/members`
- `DELETE /api/v1/projects/:id/members/:userId`

**タスク**
- `GET    /api/v1/projects/:projectId/tasks`
- `POST   /api/v1/projects/:projectId/tasks`
- `GET    /api/v1/tasks/:id`
- `PATCH  /api/v1/tasks/:id`
- `DELETE /api/v1/tasks/:id`（ADMIN 限定）
- `POST   /api/v1/tasks/:id/position`（R-05 並び順／列移動）
- `POST   /api/v1/tasks/:id/request`（依頼）
- `POST   /api/v1/tasks/:id/cancel-request`（取り下げ）
- `POST   /api/v1/tasks/:id/accept`（承諾、本人のみ）
- `POST   /api/v1/tasks/:id/decline`（辞退、本人のみ）

**コメント**
- `GET    /api/v1/tasks/:taskId/comments`
- `POST   /api/v1/tasks/:taskId/comments`
- `PATCH  /api/v1/comments/:id`（本人）
- `DELETE /api/v1/comments/:id`（本人または ADMIN、論理削除）

**管理者ダッシュボード**
- `GET /api/v1/admin/dashboard/summary`
- `GET /api/v1/admin/dashboard/projects`
- `GET /api/v1/admin/dashboard/members`

### 6.4 バリデーション・型同期
- `packages/shared` の **Zod スキーマ** を Single Source of Truth とする
- BE: `nestjs-zod` の `ZodValidationPipe` で入力検証
- FE: `react-hook-form` + `zodResolver` で入力検証

### 6.5 セキュリティ
- 認証: HTTP-only Cookie のセッション（`Secure`、`SameSite=Lax`）
- CSRF: `SameSite=Lax` + Double Submit Cookie パターン
- XSS: React の自動エスケープ + `helmet` の CSP、入力は原則プレーンテキスト
- CORS: 本番は同一オリジン、開発のみフロントオリジンを明示許可
- レートリミット: `@nestjs/throttler` で認証系エンドポイントに限定適用

---

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
