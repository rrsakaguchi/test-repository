# アプリケーション設計書（MANIFEST）

> 本書はプロジェクトのアプリケーション設計の **最終版** である。要求リスト（R-01〜R-12）、データモデル（§5）、API 仕様（§6）、画面構成（§4）の各詳細設計を統合し、実装段階で必要となる横断的決定をすべて確定済みとする。実装の指針として本書を単一の参照点（Single Source of Truth）とし、変更が必要となった場合は本書を更新したうえで派生ドキュメント（OpenAPI、Prisma マイグレーション、Storybook 等）へ反映する運用とする。

---

## 1. プロジェクト概要

### 1.1 目的

複数プロジェクトをまたいだタスクの可視化と、「依頼 / 承諾」フローによる担当責任の明確化を提供する社内向け業務ツールである。具体的には以下の三つを中核価値とする。

- **付箋を動かす感覚でのタスク操作（R-05）**: 3 列固定カンバン（`TODO` / `IN_PROGRESS` / `DONE`）でドラッグ&ドロップ操作を実現する。操作の軽快さを最優先とし、並び替えエンドポイント `POST /tasks/:id/position`（§6.11.6）のみ楽観ロックを緩める。
- **責任の所在を曖昧にしない依頼承諾（R-07, R-08）**: タスク担当の付与は `UNASSIGNED → REQUESTED → ACCEPTED` の状態機械（§5.3.5, §6.14）でのみ進行させる。直接 `assigneeId` を設定する API 経路は提供しない。
- **管理者の俯瞰（R-11）**: 進捗・負荷の二軸を、リアルタイム DB 集計で `/api/v1/admin/dashboard/*`（§6.13）として提供する。

### 1.2 スコープ

#### 含む（初期リリース対象）

| 機能 | 関連要求 | データモデル | API |
|---|---|---|---|
| 招待制ユーザー管理（招待・受諾・無効化・パスワード変更） | R-12 | `users` / `invitations` | §6.5〜6.8 |
| プロジェクト管理（作成・編集・アーカイブ） | R-01 | `projects` | §6.9 |
| 自分の参加プロジェクト一覧 | R-02 | `project_members` | §6.9.1 |
| プロジェクトメンバー管理 | R-04 | `project_members` | §6.10 |
| 3 列固定カンバンとドラッグ&ドロップ | R-03, R-05 | `tasks` | §6.11.1, §6.11.6 |
| タスクの依頼・承諾・取り下げ・辞退 | R-07, R-08 | `tasks.assignment_status` | §6.11.7〜6.11.11 |
| タスクへのコメント（プレーンテキスト・編集・論理削除） | R-10 | `comments` | §6.12 |
| メール通知（承諾時 R-09 ほか） | R-09 | `notifications` | §6.11.9 ほか副作用 |
| 管理者ダッシュボード（全体俯瞰） | R-11 | 全テーブル横断集計 | §6.13 |

#### 含まない（スコープ外、将来拡張は §10）

優先度・タグ・添付ファイル・サブタスク・依存関係、コメントのメンション / Markdown / スレッド / リアクション、アプリ内通知センターの UI、MFA / SSO、パスワードリセットフロー、カンバン列のカスタマイズ、Webhook / 外部連携、全文検索、国際化。

### 1.3 設計上の主要決定（確定事項）

| カテゴリ | 決定 | 根拠・参照 |
|---|---|---|
| アーキテクチャ | SPA + REST API の 2 層構成、同一オリジン運用、BFF なし | フロント・バック間で別オリジン運用しないため CORS の本番設定は不要（§6.17） |
| Monorepo | pnpm workspaces + Turborepo、`apps/{web, api}` + `packages/shared` の 3 ワークスペース | §8 |
| タイムゾーン | BE / FE / DB すべて `Asia/Tokyo`（JST）で統一、API シリアライズは ISO 8601 + `+09:00`、`Task.dueDate` のみ日付型 | §5.1, §6.1 |
| 主キー | `id` 列を持つ全モデルで `cuid()`（String）。例外: `ProjectMember`（複合 PK）、外部管理の `session` テーブル（`sid`） | §5.1, §5.3.4 |
| 認証 | セッション認証（HTTP-only Cookie、PostgreSQL セッションストア）。JWT・Redis 不採用 | §3.1, §6.17 |
| CSRF 対策 | `SameSite=Lax` + Double Submit Cookie（`X-CSRF-Token` ヘッダ検証） | §6.17 |
| 楽観ロック | `updatedAt` 比較で実装。例外: `POST /tasks/:id/position`（R-05 の軽快さ優先） | §5.1, §6.11.6 |
| 型 / バリデーション | `packages/shared` の Zod スキーマを Single Source of Truth とし、BE は `nestjs-zod`、FE は `react-hook-form + zodResolver` で利用 | §6.16, §7 |
| API ドキュメント | `@nestjs/swagger` で OpenAPI を `/api/docs` に自動生成 | §6.1, §6.16 |
| 非同期処理 | pg-boss（PostgreSQL 駆動）。Outbox パターンで「DB 変更 + ジョブ投入」をトランザクションに統合 | §3.4, §3.6, §6.11.9 |
| 削除戦略 | User / Project / Comment は論理削除、Task は ADMIN による物理削除、ProjectMember はハード削除、Notification は永続保持 | §5.4 |

---

## 2. ユーザーロール定義 (RBAC)

### 2.1 ロール一覧

| ロール | コード | 概要 | 主体カラム |
|---|---|---|---|
| 管理者 | `ADMIN` | ユーザー招待、プロジェクト作成、メンバー管理、タスク物理削除、ダッシュボード閲覧、招待管理を担う。プロジェクト固有の操作は **当該プロジェクトのメンバーである場合に限り** 実行可能。 | `User.role = 'ADMIN'` |
| メンバー | `MEMBER` | 自分が所属するプロジェクト内でのタスク操作・コメント・依頼/承諾を行う一般ユーザー。 | `User.role = 'MEMBER'` |

ロールとは独立に **ユーザーの状態** が認可に影響する。

- `User.status = ACTIVE`: ログイン可・全操作対象。
- `User.status = INACTIVE`: ログイン不可（`401 INVALID_CREDENTIALS`）。`deleted_at` が併設される。

### 2.2 認可境界（多層防御）

NestJS の Guard・Service・Prisma クエリの 4 層で防御する。前段の Guard 漏れに対して後段が必ず最終防御線として機能する。

1. **`SessionAuthGuard`** — Cookie のセッションを検証し、`status = ACTIVE` のユーザーであることを確認する。不通過は `401 AUTH_REQUIRED` / `401 SESSION_EXPIRED`。
2. **`CsrfGuard`** — 状態変更系（POST / PATCH / DELETE）は `X-CSRF-Token` ヘッダと `csrf_token` Cookie の一致を検証する（§6.17 Double Submit Cookie）。不通過は `403 CSRF_TOKEN_MISMATCH`。
3. **`RolesGuard`** — `@Roles('ADMIN')` デコレータが付与されたエンドポイントで `User.role = 'ADMIN'` を要求する。不通過は `403 ADMIN_REQUIRED`。
4. **`ProjectMemberGuard`** — プロジェクトスコープのエンドポイント（パスに `projectId` または `taskId` / `commentId` を含み、当該リソースが特定プロジェクトに紐づくもの）で `ProjectMember` 行の存在を確認する。不通過は `403 PROJECT_MEMBER_REQUIRED`（ADMIN であっても非メンバーであれば不通過）。
5. **Service 層クエリ** — 取得・更新クエリには必ず `project: { members: { some: { userId } } }` フィルタを付与する。Guard を経由しない内部呼び出しに対する最終防御線。

### 2.3 API エンドポイント単位の認可ポリシー

「**ADMIN かつメンバー**」というセルは、ADMIN ロールであっても当該プロジェクトに `ProjectMember` として所属していなければ実行できないことを意味する（§6.9.3 等で明示されている `PROJECT_MEMBER_REQUIRED` の方針）。

#### 2.3.1 認証・自分のアカウント

| エンドポイント | 未認証 | MEMBER | ADMIN | 補足 |
|---|:---:|:---:|:---:|---|
| `POST /auth/login`（§6.5.1） | ○ | ○ | ○ | レートリミット `5 req/min/IP` |
| `POST /auth/logout`（§6.5.2） | × | ○ | ○ | - |
| `GET /auth/me`（§6.5.3） | × | ○ | ○ | - |
| `GET /auth/invitations/:token`（§6.5.4） | ○ | ○ | ○ | レートリミット `10 req/min/IP` |
| `POST /auth/invitations/:token/accept`（§6.5.5） | ○ | × | × | 受諾後にセッション発行。レートリミット `5 req/min/IP` |
| `GET /me`（§6.6.1） | × | ○ | ○ | - |
| `PATCH /me`（§6.6.2） | × | ○ | ○ | - |
| `PATCH /me/password`（§6.6.3） | × | ○ | ○ | レートリミット `5 req/min/user`、他セッション失効 |

#### 2.3.2 ユーザー管理・招待管理（ADMIN 限定）

| エンドポイント | MEMBER | ADMIN | 補足 |
|---|:---:|:---:|---|
| `GET /users`（§6.7.1） | × | ○ | - |
| `GET /users/:id`（§6.7.2） | × | ○ | - |
| `POST /users/invite`（§6.7.3） | × | ○ | レートリミット `30 req/min/user` |
| `PATCH /users/:id`（§6.7.4） | × | ○ | 自分自身を INACTIVE 化不可、最後の ADMIN 降格不可 |
| `DELETE /users/:id`（§6.7.5） | × | ○ | 同条件、論理削除 |
| `GET /invitations`（§6.8.1） | × | ○ | - |
| `POST /invitations/:id/resend`（§6.8.2） | × | ○ | レートリミット `10 req/min/user` |
| `DELETE /invitations/:id`（§6.8.3） | × | ○ | 未消化のみハード削除可 |

#### 2.3.3 プロジェクト

| エンドポイント | MEMBER（メンバー） | MEMBER（非メンバー） | ADMIN（メンバー） | ADMIN（非メンバー） |
|---|:---:|:---:|:---:|:---:|
| `GET /projects`（§6.9.1） | ○ | (自分の所属分のみ返却) | ○ | (自分の所属分のみ返却) |
| `POST /projects`（§6.9.2） | × | × | ○ | ○ |
| `GET /projects/:id`（§6.9.3） | ○ | × | ○ | × |
| `PATCH /projects/:id`（§6.9.4） | × | × | ○ | × |
| `POST /projects/:id/archive`（§6.9.5） | × | × | ○ | × |
| `POST /projects/:id/unarchive`（§6.9.6） | × | × | ○ | × |

#### 2.3.4 プロジェクトメンバー

| エンドポイント | MEMBER（メンバー） | MEMBER（非メンバー） | ADMIN（メンバー） | ADMIN（非メンバー） |
|---|:---:|:---:|:---:|:---:|
| `GET /projects/:projectId/members`（§6.10.1） | ○ | × | ○ | × |
| `POST /projects/:projectId/members`（§6.10.2） | × | × | ○ | × |
| `DELETE /projects/:projectId/members/:userId`（§6.10.3） | × | × | ○ | × |

#### 2.3.5 タスク

| エンドポイント | MEMBER（メンバー） | MEMBER（非メンバー） | ADMIN（メンバー） | ADMIN（非メンバー） |
|---|:---:|:---:|:---:|:---:|
| `GET /projects/:projectId/tasks`（§6.11.1） | ○ | × | ○ | × |
| `POST /projects/:projectId/tasks`（§6.11.2） | ○ | × | ○ | × |
| `GET /tasks/:id`（§6.11.3） | ○ | × | ○ | × |
| `PATCH /tasks/:id`（§6.11.4） | ○ | × | ○ | × |
| `DELETE /tasks/:id`（§6.11.5） | × | × | ○ | × |
| `POST /tasks/:id/position`（§6.11.6） | ○ | × | ○ | × |
| `POST /tasks/:id/request`（§6.11.7） | ○（自己依頼禁止） | × | ○（自己依頼禁止） | × |
| `POST /tasks/:id/cancel-request`（§6.11.8） | （依頼者本人） | × | ○（または依頼者本人） | × |
| `POST /tasks/:id/accept`（§6.11.9） | （被依頼者本人のみ） | × | （被依頼者本人のみ） | × |
| `POST /tasks/:id/decline`（§6.11.10） | （被依頼者本人のみ） | × | （被依頼者本人のみ） | × |
| `POST /tasks/:id/unassign`（§6.11.11） | （担当者本人） | × | ○（または担当者本人） | × |

> 承諾・辞退は **被依頼者本人のみ** が実行可能（ADMIN であっても他者宛の依頼は承諾不可、`403 NOT_REQUESTEE`）。これは「責任の所在を曖昧にしない」という設計意図（§3.4）に基づく。

#### 2.3.6 コメント

| エンドポイント | MEMBER（メンバー） | ADMIN（メンバー） | ADMIN（非メンバー） |
|---|:---:|:---:|:---:|
| `GET /tasks/:taskId/comments`（§6.12.1） | ○ | ○ | × |
| `POST /tasks/:taskId/comments`（§6.12.2） | ○ | ○ | × |
| `PATCH /comments/:id`（§6.12.3） | （本人投稿のみ） | （本人投稿のみ。他者のコメントは ADMIN でも編集不可） | × |
| `DELETE /comments/:id`（§6.12.4） | （本人投稿のみ） | ○（他者のコメントも論理削除可） | × |

#### 2.3.7 管理者ダッシュボード（ADMIN 限定）

| エンドポイント | MEMBER | ADMIN |
|---|:---:|:---:|
| `GET /admin/dashboard/summary`（§6.13.1） | × | ○ |
| `GET /admin/dashboard/projects`（§6.13.2） | × | ○ |
| `GET /admin/dashboard/members`（§6.13.3） | × | ○ |

---

## 3. 機能要件

各機能は、要求番号 (R-XX) ・データモデル (§5) ・API (§6) ・画面 (§4) との対応を明示する。

### 3.1 認証・ユーザー管理

| 項目 | 仕様 | 参照 |
|---|---|---|
| 招待制アカウント作成（R-12） | ADMIN がメールで招待し、招待 URL の使い切りトークン経由で初回パスワードを設定 | `users` / `invitations`、§6.5.4〜6.5.5、§6.7.3 |
| 招待トークン | 使い切り、有効期限 7 日、メールアドレス紐付け、DB には SHA-256 ハッシュのみ保存 | §5.3.2 |
| ログイン / ログアウト | メール + パスワードのセッション認証。Cookie ベース（HTTP-only、`Secure`、`SameSite=Lax`） | §6.5.1〜6.5.2、§6.17 |
| パスワードポリシー | bcrypt（コスト係数 12）、NIST 800-63B 準拠（最低 12 文字、上限 128 文字、複雑性ルール強制なし、定期変更なし） | §6.5.5 Body、§6.6.3 Body |
| パスワード変更 | 現パスワードを確認したうえで更新。実行後、自セッションを除く他の全セッションを失効 | §6.6.3 |
| 自プロフィール編集 | 表示名・アバター URL の更新（楽観ロック付き） | §6.6.2 |
| ユーザー論理削除 | `status = INACTIVE` + `deleted_at` を単一トランザクションで設定。同時に全セッション失効、担当中タスクを `UNASSIGNED` に戻す、被依頼中 `REQUESTED` を自動取り下げ | §5.4、§6.7.5 |
| 退会済みユーザー残置 | コメント・通知履歴・所属（ProjectMember）・招待履歴を保持。UI 上は `UserSummary.status = INACTIVE` で「退会済みユーザー」と表示 | §5.3.6 |
| 最後の ADMIN 保護 | 自分自身を INACTIVE 化、または最後の ADMIN を MEMBER に降格しようとした場合は `422 LAST_ADMIN_PROTECTED` / `422 SELF_FORBIDDEN` | §6.7.4、§6.7.5 |
| MFA / SSO | 初期未導入。データモデル上の拡張余地のみ確保（§10.1） | - |

### 3.2 プロジェクト管理

| 項目 | 仕様 | 参照 |
|---|---|---|
| 自分の参加プロジェクト一覧（R-02） | 自分が `ProjectMember` として所属するプロジェクトのみ返す。ADMIN であっても非所属プロジェクトは表示されない | §6.9.1 |
| プロジェクト作成 | ADMIN のみ。初期メンバーを同一トランザクションで登録できる。作成者 ADMIN 自身は自動でメンバーには加わらない | §6.9.2 |
| プロジェクト編集 | ADMIN かつ当該プロジェクトのメンバー。名前・説明を更新（楽観ロック） | §6.9.4 |
| プロジェクトアーカイブ | 論理削除（`archived_at = now()`）。配下のタスクには影響しないが、既定の一覧からは除外。ダッシュボード集計の既定もアーカイブ除外 | §6.9.5、§6.13 |
| アーカイブ解除 | `archived_at = NULL` に戻す | §6.9.6 |
| メンバー追加 | ADMIN かつメンバー。`INACTIVE` ユーザーは追加不可（`422 VALIDATION_ERROR`） | §6.10.2 |
| メンバー除外 | ハード削除。同一トランザクションで担当中タスクを `UNASSIGNED` に戻し、被依頼中 `REQUESTED` を自動取り下げ。コメントは残置 | §6.10.3 |

### 3.3 タスク管理

| 項目 | 仕様 | 参照 |
|---|---|---|
| 3 列固定カンバン（R-03） | `TODO` / `IN_PROGRESS` / `DONE` の 3 列固定。カスタム列は提供しない | §5.3.5、§6.11.1 |
| タスク属性 | タイトル（1〜200 文字）、説明（0〜10000 文字、任意）、ステータス、担当の状態機械、期日 `due_date`（任意、`YYYY-MM-DD`） | §5.3.5、§6.4 Task |
| タスク作成 | プロジェクトメンバーが実行。作成時点では常に `assignment_status = UNASSIGNED`。`assignee_id` の直接指定は不可 | §6.11.2 |
| タスク編集 | プロジェクトメンバーが実行。タイトル・説明・期日・ステータス（列）を更新可能。担当者の変更は §3.4 の依頼承諾フローでのみ行う | §6.11.4 |
| タスク物理削除 | ADMIN かつメンバー。CASCADE で関連コメント・通知も物理削除 | §5.4、§6.11.5 |
| ドラッグ&ドロップ並び替え（R-05） | `POST /tasks/:id/position` 経由。同一 `(project_id, status)` 内の `position` をトランザクション内で再計算。**本エンドポイントのみ楽観ロックを適用しない**（最終勝者が並びを決定） | §5.1、§6.11.6 |
| 列移動制限 | なし。`TODO → DONE` のような飛び越し移動も許可 | §6.11.6 |
| 初期非対応（YAGNI） | 優先度、タグ、添付ファイル、サブタスク、依存関係 | §10.1 |

### 3.4 タスクの依頼・承諾フロー（R-07, R-08）

| 項目 | 仕様 | 参照 |
|---|---|---|
| 状態機械 | `UNASSIGNED → REQUESTED → ACCEPTED` の三状態 | §5.3.5、§6.14 |
| 遷移操作 | 依頼 / 取り下げ / 辞退 / 承諾 / 担当解除（unassign）の 5 種 | §6.11.7〜6.11.11 |
| 状態と担当系カラムの不変条件 | `UNASSIGNED`: 全担当系 NULL／`REQUESTED`: `requested_to_id` と `requested_by_id` が NOT NULL／`ACCEPTED`: `assignee_id` のみ NOT NULL | §5.3.5、CHECK 制約 §5.5.2 |
| 承諾者制限 | **被依頼者本人のみ** 実行可能。ADMIN であっても他人宛の依頼は承諾不可（`403 NOT_REQUESTEE`） | §6.11.9 |
| 担当付け替え経路 | 必ず `UNASSIGNED` を経由する（`ACCEPTED → UNASSIGNED → REQUESTED → ACCEPTED`）。`PATCH /tasks/:id` および `POST /projects/:projectId/tasks` で `assignee_id` を直接指定する経路は提供しない | §6.14 |
| 承諾時の通知（R-09） | 「タスク更新 + Notification レコード作成 + メール送信ジョブ投入」を **単一トランザクション**で実行（Outbox + pg-boss） | §3.6、§6.11.9 |
| 自己依頼の禁止 | 自分自身への依頼は `422 SELF_FORBIDDEN` | §6.11.7 |
| 非メンバー依頼の禁止 | プロジェクト非所属ユーザーへの依頼は `422 ASSIGNEE_NOT_PROJECT_MEMBER` | §6.11.7 |

### 3.5 コメント機能 (R-10)

| 項目 | 仕様 | 参照 |
|---|---|---|
| 本文 | 1〜5000 文字のプレーンテキスト。Markdown は解釈しない | §5.3.6、§6.12.2 |
| 編集 | 投稿者本人のみ。ADMIN でも他者コメントの編集は不可（`403 NOT_AUTHOR`）。編集後は `is_edited = true` | §6.12.3 |
| 削除 | 投稿者本人または ADMIN による論理削除（`deleted_at` 設定）。DB には原文を保持し、API レスポンスでのみ `body = ""` にマスク | §5.3.6、§6.12.4 |
| 退会者のコメント | 著者ユーザーが INACTIVE になっても残置。UI 上は「退会済みユーザー」表示 | §5.3.6 |
| 初期非対応 | メンション、添付、スレッド、Markdown レンダリング、リアクション | §10.1 |

### 3.6 通知

| 項目 | 仕様 | 参照 |
|---|---|---|
| チャネル | 初期実装は **メール送信のみ**。`notifications.channel` enum は `EMAIL` / `IN_APP` を準備済（IN_APP は将来の通知センター UI 用） | §5.3.7 |
| 通知種別 | `TASK_REQUESTED` / `TASK_ACCEPTED` / `TASK_DECLINED` / `TASK_REQUEST_CANCELLED` / `PROJECT_MEMBER_ADDED` の 5 種 | §5.3.7 |
| Outbox パターン | 業務トランザクション内で `notifications` 行を作成し、pg-boss にメール送信ジョブを投入する。送信成功時に `status = SENT` / `sent_at` を更新、失敗時に `status = FAILED` / `error_message` を記録 | §5.3.7、§6.11.9 |
| 招待メール | 受信者がまだ User として存在しないため、Notification テーブル経由ではなく pg-boss に直接ジョブを投入。送信履歴は `invitations` テーブルで代替 | §5.3.7 注 |
| 送信基盤 | 本番: Amazon SES、開発: Mailpit（§7.2） | - |
| 永続化 | `notifications` テーブルに履歴を永続保持。バッチ削除ポリシーは初期未導入 | §5.4 |

### 3.7 管理者ダッシュボード (R-11)

| 項目 | 仕様 | 参照 |
|---|---|---|
| 対象ロール | `ADMIN` のみ | §6.13、§2.3.7 |
| 集計方式 | リアルタイム DB クエリ。バッチキャッシュは持たない（運用上重くなった場合は §10.2 の読み取りレプリカ・キャッシュ層を検討） | §6.13 |
| 全社サマリ | プロジェクト数 / 総タスク数 / 完了率 / 直近 7 日完了 / 期限超過 / `ACCEPTED` 未完了 / `REQUESTED` 未承諾 / 期限切迫（3 日以内） | §6.13.1 |
| プロジェクト別進捗 | 進捗率昇順がデフォルト（介入優先度）。アーカイブは既定除外、`includeArchived` で含める | §6.13.2 |
| メンバー別負荷 | `acceptedOpen` / `requestedPending` / `overdue` / `dueSoon` / 直近 7 日完了。負荷の重い順がデフォルト | §6.13.3 |
| 期日のしきい値 | 「期限切迫」= `due_date <= today + 3`、「期限超過」= `due_date < today` かつ `status != DONE` | §6.13.1 |
| 支えるインデックス | `(project_id, status, position)`・`(assignee_id, status)`・`(due_date)` | §5.3.5 |

---

## 4. 画面遷移・構成

### 4.1 主要画面一覧

| 画面 | パス | アクセス権 | 主な機能 | 主要 API |
|---|---|---|---|---|
| ログイン | `/login` | 未認証 | メール + パスワードでログイン | `POST /auth/login` |
| 招待受諾 | `/invite/:token` | 未認証 | 招待トークン検証 + 初回パスワード設定 + 自動ログイン（R-12） | `GET /auth/invitations/:token`、`POST /auth/invitations/:token/accept` |
| プロジェクト一覧 | `/projects` | 認証済（所属するもののみ表示、R-02） | 自分の参加プロジェクト一覧、アーカイブ表示切替 | `GET /projects?includeArchived=` |
| プロジェクト詳細（カンバン） | `/projects/:id` | プロジェクトメンバー | 3 列カンバン、ドラッグ&ドロップ並び替え（R-03, R-05）、タスク作成、担当者の顔写真表示（R-04） | `GET /projects/:projectId/tasks`、`POST /tasks/:id/position`、`POST /projects/:projectId/tasks` |
| タスク詳細（モーダル） | `/projects/:id/tasks/:taskId` | プロジェクトメンバー | ポップアップ形式の編集（R-06）、コメント（R-10）、依頼・承諾・辞退・取り下げ・担当解除 | `GET /tasks/:id`、`PATCH /tasks/:id`、`POST /tasks/:id/request|accept|decline|cancel-request|unassign`、§6.12 全般 |
| プロジェクト設定 | `/projects/:id/settings` | ADMIN かつ プロジェクトメンバー | 名前・説明編集、アーカイブ／解除、メンバー追加・除外 | `PATCH /projects/:id`、`POST /projects/:id/archive|unarchive`、§6.10 全般 |
| 管理者ダッシュボード | `/admin/dashboard` | ADMIN | 全社サマリ・プロジェクト別進捗・メンバー別負荷（R-11） | §6.13 全般 |
| ユーザー管理 | `/admin/users` | ADMIN | ユーザー一覧、招待発行、ロール変更、無効化、招待の再送・取り消し | §6.7、§6.8 全般 |
| アカウント設定 | `/me` | 認証済 | プロフィール編集、パスワード変更 | `PATCH /me`、`PATCH /me/password` |

> プロジェクト設定画面（`/projects/:id/settings`）は ADMIN ロールが必須だが、当該プロジェクトに `ProjectMember` として所属していない ADMIN は `PROJECT_MEMBER_REQUIRED`（§2.3.3, §2.3.4）でアクセス不可。

### 4.2 グローバルレイアウト

| 領域 | 役割 |
|---|---|
| ヘッダー | ロゴ、現在のプロジェクト切替、ユーザーメニュー（自分のアバター→ アカウント設定 / ログアウト） |
| サイドバー | メインナビゲーション（プロジェクト一覧へのリンク、管理メニューは ADMIN のみ表示） |
| メインエリア | 各ルートのコンテンツ |
| トースト | 操作結果のフィードバック表示（成功・失敗・楽観ロック競合等） |
| モーダル | タスク詳細編集、依頼ダイアログ、削除確認等 |

### 4.3 ルーティング方針

- React Router v6 のネスト型ルーティング。
- 認証要否は `<RequireAuth>` ラッパー、ロール要件は `<RequireRole role="ADMIN">` ラッパーで判定する。判定は `GET /auth/me` の結果（TanStack Query でキャッシュ）に基づく。
- 未認証アクセス時は `/login?redirect=<元の URL>` にリダイレクトし、ログイン成功後に元の URL に戻す。
- 認可エラー（`403 ADMIN_REQUIRED` / `403 PROJECT_MEMBER_REQUIRED`）はトーストで通知し、`/projects` にリダイレクトする。
- 楽観ロック競合（`409 OPTIMISTIC_LOCK_FAILED`）はトーストで通知のうえ、最新データを再取得して再編集を促す。

---

## 5. データモデリング

本章は、要求 R-01〜R-12 および §3 機能要件・§6 API 設計を完全に支えるデータモデルを定義する。設計の前提（主キー、タイムゾーン、命名規約等）を §5.1 で示し、続けて (1) ER 図 → (2) テーブル定義 → (3) 削除・無効化ポリシー → (4) Prisma スキーマ の順に詳細を展開する。

### 5.1 設計方針概要

| 項目 | 採用 |
|---|---|
| 主キー | 全エンティティで `cuid()`（String 型）。URL 安全・衝突耐性。例外は `ProjectMember`（複合主キー `(projectId, userId)`）。 |
| タイムゾーン | PostgreSQL の `TIMESTAMP WITH TIME ZONE`（`@db.Timestamptz(3)`）。DB セッション `timezone='Asia/Tokyo'`、API シリアライズは ISO 8601 + `+09:00`。`Task.dueDate` のみ日付型 `@db.Date`（時刻情報を持たない期日）。 |
| 命名規約 | DB は snake_case、アプリは camelCase、Prisma の `@map` / `@@map` で橋渡し。 |
| 並び順 | `Task.position` を 0 始まりの整数で保持。同一 `(projectId, status)` 内のみで意味を持つ。並び替え／列移動はトランザクション内で再計算（一意制約は付けず、最終勝者方式 §6.11.6）。 |
| 楽観ロック | 原則 `updatedAt` 比較。例外は `POST /tasks/:id/position`（R-05 の軽快さを優先）。 |
| ID ハッシュ化 | パスワードは bcrypt（コスト 12）、招待トークンは SHA-256（平文非保存）。 |
| Enum | `UserRole` / `UserStatus` / `TaskStatus` / `TaskAssignmentStatus` / `NotificationType` / `NotificationChannel` / `NotificationStatus`。 |
| 外部管理テーブル | セッションは `connect-pg-simple` 管理の `session` テーブル、非同期ジョブは `pg-boss` 管理の `pgboss.*` スキーマ。これらは Prisma のスコープ外として共存させる（§5.5 末尾の補足を参照）。 |

---

### 5.2 ER 図

```mermaid
erDiagram
    User {
        string id PK
        string email UK "globally unique"
        string name
        string password_hash "bcrypt"
        UserRole role "ADMIN | MEMBER"
        UserStatus status "ACTIVE | INACTIVE"
        string avatar_url "nullable (R-04)"
        datetime deleted_at "nullable"
        datetime created_at
        datetime updated_at
    }

    Invitation {
        string id PK
        string email
        UserRole role
        string token_hash UK "SHA-256"
        string invited_by_id FK
        datetime expires_at "now + 7 days"
        datetime accepted_at "nullable"
        datetime created_at
        datetime updated_at
    }

    Project {
        string id PK
        string name "1..100"
        string description "nullable, 0..2000"
        datetime archived_at "nullable (logical delete)"
        datetime created_at
        datetime updated_at
    }

    ProjectMember {
        string project_id PK,FK
        string user_id PK,FK
        datetime joined_at
    }

    Task {
        string id PK
        string project_id FK
        string title "1..200"
        string description "nullable, 0..10000"
        TaskStatus status "TODO | IN_PROGRESS | DONE"
        TaskAssignmentStatus assignment_status "UNASSIGNED | REQUESTED | ACCEPTED"
        string assignee_id FK "non-null iff ACCEPTED"
        string requested_to_id FK "non-null iff REQUESTED"
        string requested_by_id FK "non-null iff REQUESTED"
        string request_message "nullable"
        date due_date "nullable"
        int position "0-based, per (project_id,status)"
        datetime created_at
        datetime updated_at
    }

    Comment {
        string id PK
        string task_id FK
        string author_id FK
        string body "1..5000, '' when deleted"
        boolean is_edited
        datetime deleted_at "nullable (logical delete)"
        datetime created_at
        datetime updated_at
    }

    Notification {
        string id PK
        string recipient_id FK
        NotificationType type
        NotificationChannel channel "EMAIL | IN_APP"
        NotificationStatus status "PENDING | SENT | FAILED"
        json payload "context fields"
        string task_id FK "nullable"
        string project_id FK "nullable"
        string actor_id FK "nullable"
        datetime read_at "nullable (in-app)"
        datetime sent_at "nullable"
        datetime failed_at "nullable"
        string error_message "nullable"
        string job_id "pg-boss job id, nullable"
        datetime created_at
        datetime updated_at
    }

    User           ||--o{ ProjectMember  : "joins"
    Project        ||--o{ ProjectMember  : "has"
    Project        ||--o{ Task           : "contains"
    User           |o--o{ Task           : "assignee (ACCEPTED)"
    User           |o--o{ Task           : "requested-to (REQUESTED)"
    User           |o--o{ Task           : "requested-by (REQUESTED)"
    Task           ||--o{ Comment        : "has"
    User           ||--o{ Comment        : "authors"
    User           ||--o{ Notification   : "recipient"
    User           |o--o{ Notification   : "actor"
    Task           |o--o{ Notification   : "about"
    Project        |o--o{ Notification   : "context"
    User           ||--o{ Invitation     : "invited by"
```

---

### 5.3 テーブル定義

凡例: 「制約」列は `PK`（主キー）／`FK`（外部キー）／`UK`（一意）／`NOT NULL`／`DEFAULT`／`INDEX` を併記する。タイムスタンプ型は特記なき限り `TIMESTAMPTZ(3)`、bool は `BOOLEAN`、整数は `INTEGER`、文字列は `TEXT` を用いる。

#### 5.3.1 `users`

`User` は認証主体であり、ロール（RBAC）と状態（ACTIVE/INACTIVE）を保持する。

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `id` | TEXT | PK | `cuid()` |
| `email` | TEXT | NOT NULL, UK | ログイン ID。グローバル一意。INACTIVE になっても保持され、同じメールでの再招待・新規作成は不可（既存ユーザーを ACTIVE に戻す運用）。 |
| `name` | TEXT | NOT NULL | 表示名（1〜64 文字、アプリ層 Zod で検証） |
| `password_hash` | TEXT | NOT NULL | bcrypt ハッシュ（コスト 12）。招待受諾 §6.5.5 で初期設定。 |
| `role` | UserRole | NOT NULL, DEFAULT `MEMBER` | `ADMIN` / `MEMBER` |
| `status` | UserStatus | NOT NULL, DEFAULT `ACTIVE`, INDEX | `ACTIVE` / `INACTIVE`。INACTIVE はログイン不可。 |
| `avatar_url` | TEXT | NULL | 顔写真 URL（R-04） |
| `deleted_at` | TIMESTAMPTZ | NULL | 論理削除タイムスタンプ。`status=INACTIVE` と整合させる（再有効化時は両方クリア）。 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | - |
| `updated_at` | TIMESTAMPTZ | NOT NULL | Prisma `@updatedAt` で自動更新。楽観ロックに利用。 |

**ビジネスルール**
- `status=INACTIVE` ⇔ `deleted_at IS NOT NULL`（アプリ層で整合性を保証）。
- 最後の ADMIN は降格・無効化不可（§6.7.4/§6.7.5、`422 LAST_ADMIN_PROTECTED`）。
- 自分自身を `INACTIVE` にできない（`422 SELF_FORBIDDEN`）。
- INACTIVE への遷移時は (a) 全セッション失効、(b) 担当タスクを `UNASSIGNED` に戻す、(c) 被依頼中の `REQUESTED` を自動取り下げ、を単一トランザクションで実行（§6.7.5）。コメントは残置。
- パスワードは平文を一切保持せず、`password_hash` のみ DB に保持。

**インデックス**: `(status)`, `(role)`, `(deleted_at)`。

#### 5.3.2 `invitations`

招待は「アカウント本体作成前のチケット」であり、受諾 §6.5.5 で User が生成される。

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `id` | TEXT | PK | `cuid()` |
| `email` | TEXT | NOT NULL, INDEX | 招待先メールアドレス |
| `role` | UserRole | NOT NULL | 招待時に設定するロール |
| `token_hash` | TEXT | NOT NULL, UK | 招待トークン本体の SHA-256 ハッシュ。原文はメール送信時のみ存在。 |
| `invited_by_id` | TEXT | NOT NULL, FK→`users.id` (ON DELETE RESTRICT) | 招待者 |
| `expires_at` | TIMESTAMPTZ | NOT NULL, INDEX | 既定: 発行から 7 日後 |
| `accepted_at` | TIMESTAMPTZ | NULL, INDEX | 受諾日時。NULL なら未消化。 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | - |
| `updated_at` | TIMESTAMPTZ | NOT NULL | - |

**ビジネスルール**
- 「未消化」= `accepted_at IS NULL` AND `expires_at > now()`。「受諾済」= `accepted_at IS NOT NULL`。「期限切れ」= `accepted_at IS NULL` AND `expires_at <= now()`。
- 同一メール宛の未消化招待は同時に 1 件のみ（アプリ層で検証、`409 INVITATION_ALREADY_USED`）。
- 再送 §6.8.2 では既存レコードの `token_hash` を新トークンの SHA-256 で上書きし、`expires_at` を更新する（古い URL は即時無効化）。
- 受諾済招待はハード削除不可（監査用途で残置）。未消化／期限切れの取り消しはハード削除可。
- DB には URL 安全な原文トークンを保持しない（漏洩耐性）。

**インデックス**: `(email)`, `(accepted_at)`, `(expires_at)`。

#### 5.3.3 `projects`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `id` | TEXT | PK | `cuid()` |
| `name` | TEXT | NOT NULL | 1〜100 文字 |
| `description` | TEXT | NULL | 0〜2000 文字 |
| `archived_at` | TIMESTAMPTZ | NULL, INDEX | 論理削除タイムスタンプ。NULL なら稼働中。 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | - |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 楽観ロック |

**ビジネスルール**
- アーカイブ条件: ADMIN かつ当該プロジェクトのメンバーが §6.9.5 を実行（`409 CONFLICT` if 既にアーカイブ済）。
- アーカイブ後も配下の `tasks` / `comments` / `notifications` は残置（物理削除はしない）。
- 既定の `GET /projects` ではアーカイブ済を除外、`includeArchived=true` で含める。
- 管理者ダッシュボード §6.13 の集計もデフォルトはアーカイブ除外。
- アーカイブ解除（unarchive）は可能（`archived_at = null`）。

**インデックス**: `(archived_at)`。

#### 5.3.4 `project_members`

`User × Project` の所属関係を表す結合テーブル。所属の有無が認可境界（§2.2 `ProjectMemberGuard`）の判定軸となる。

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `project_id` | TEXT | PK 構成要素, FK→`projects.id` (ON DELETE CASCADE) | プロジェクト ID |
| `user_id` | TEXT | PK 構成要素, FK→`users.id` (ON DELETE RESTRICT) | ユーザー ID |
| `joined_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | 参加日時 |

**ビジネスルール**
- 複合主キー `(project_id, user_id)` により重複所属を防止（`409 ALREADY_MEMBER`）。
- 追加対象が `INACTIVE` ユーザーの場合は `422 VALIDATION_ERROR`。
- 除外 §6.10.3 はハード削除。同一トランザクションで (a) 当該プロジェクト内の担当中タスクを `UNASSIGNED` に戻す、(b) 被依頼中 `REQUESTED` を自動取り下げ、(c) コメントは残置。
- 自分自身は除外不可（`422 SELF_FORBIDDEN`）。
- プロジェクトが物理削除される運用は無いが、仮に削除されれば CASCADE で本テーブルも削除。User の物理削除は FK RESTRICT で禁止（論理削除のみ）。

**インデックス**: 複合主キーで `(project_id, user_id)`、副次インデックスとして `(user_id)`（R-02 の自分のプロジェクト一覧で活用）。

#### 5.3.5 `tasks`

タスク本体。状態機械 §6.14 と並び順 §5.1 の中心。

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `id` | TEXT | PK | `cuid()` |
| `project_id` | TEXT | NOT NULL, FK→`projects.id` (ON DELETE CASCADE) | 所属プロジェクト |
| `title` | TEXT | NOT NULL | 1〜200 文字 |
| `description` | TEXT | NULL | 0〜10000 文字 |
| `status` | TaskStatus | NOT NULL, DEFAULT `TODO` | カンバン列 |
| `assignment_status` | TaskAssignmentStatus | NOT NULL, DEFAULT `UNASSIGNED` | 担当状態機械 |
| `assignee_id` | TEXT | NULL, FK→`users.id` (ON DELETE SET NULL) | `ACCEPTED` 時のみ非 NULL |
| `requested_to_id` | TEXT | NULL, FK→`users.id` (ON DELETE SET NULL) | `REQUESTED` 時のみ非 NULL |
| `requested_by_id` | TEXT | NULL, FK→`users.id` (ON DELETE SET NULL) | `REQUESTED` 時の依頼者 |
| `request_message` | TEXT | NULL | 依頼メッセージ（§6.11.7 で受領） |
| `due_date` | DATE | NULL | `YYYY-MM-DD`。負荷集計に活用（R-11）。 |
| `position` | INTEGER | NOT NULL, DEFAULT `0` | 0 始まり、同 `(project_id, status)` 内の並び順 |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | - |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 楽観ロック（position 変更時を除く） |

**ビジネスルール（不変条件 / Invariants）**

| `assignment_status` | `assignee_id` | `requested_to_id` | `requested_by_id` |
|---|---|---|---|
| `UNASSIGNED` | NULL 必須 | NULL 必須 | NULL 必須 |
| `REQUESTED` | NULL 必須 | NOT NULL 必須 | NOT NULL 必須 |
| `ACCEPTED` | NOT NULL 必須 | NULL 必須 | NULL 必須 |

これらは Service 層トランザクション内で保証する（DB レベルの CHECK 制約は明示的な migration で `ALTER TABLE tasks ADD CONSTRAINT chk_task_assignment_consistency CHECK (...)` を追加することを推奨）。

**状態遷移ルール（§6.14 と整合）**
- `UNASSIGNED → REQUESTED`: `POST /tasks/:id/request`（プロジェクトメンバーが他メンバーに依頼。自己依頼禁止 `422 SELF_FORBIDDEN`、非メンバー依頼禁止 `422 ASSIGNEE_NOT_PROJECT_MEMBER`）
- `REQUESTED → UNASSIGNED`: `POST /tasks/:id/cancel-request`（依頼者 or ADMIN）または `POST /tasks/:id/decline`（被依頼者本人）
- `REQUESTED → ACCEPTED`: `POST /tasks/:id/accept`（被依頼者本人のみ。同時に通知 + メール送信ジョブ投入 R-09）
- `ACCEPTED → UNASSIGNED`: `POST /tasks/:id/unassign`（担当者本人 or ADMIN）
- 担当者の付け替えは必ず `UNASSIGNED` を経由する（責任の所在を曖昧にしない、§3.4）。
- `PATCH /tasks/:id` および `POST /projects/:projectId/tasks` で `assignee_id` を直接指定する経路は提供しない。

**`status` (カンバン列) ルール**
- 3 列固定（`TODO` / `IN_PROGRESS` / `DONE`）。列移動制限なし。
- ドラッグ&ドロップは `POST /tasks/:id/position` 経由（楽観ロックなし、§6.11.6）。
- `PATCH /tasks/:id` でも `status` 変更は可能。

**`position` ルール**
- `INSERT` 時は同 `(project_id, status)` の最大値 + 1。
- 列移動／並び替え時はトランザクション内で移動先列の `position >= 挿入位置` を +1 し、元の列の `position > 元位置` を -1 する（または再採番）。
- 一意制約は付けない（最終勝者方式、軽快さ優先）。

**インデックス**: `(project_id, status, position)` ★カンバン取得 §6.11.1 と並び替えの主用途、`(assignee_id, status)` ★ダッシュボード負荷集計 §6.13.3、`(requested_to_id)`、`(due_date)` ★期限超過／切迫の集計、`(project_id, status)` は前出複合に包含。

#### 5.3.6 `comments`

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `id` | TEXT | PK | `cuid()` |
| `task_id` | TEXT | NOT NULL, FK→`tasks.id` (ON DELETE CASCADE) | 親タスク |
| `author_id` | TEXT | NOT NULL, FK→`users.id` (ON DELETE RESTRICT) | 投稿者。ユーザーは論理削除のみのため RESTRICT で物理削除を防止。 |
| `body` | TEXT | NOT NULL | 1〜5000 文字、プレーンテキスト。DB には原文を保持し、論理削除済みコメントの API レスポンスでのみ `""` にマスクする（§6.12.4）。 |
| `is_edited` | BOOLEAN | NOT NULL, DEFAULT `false` | 編集済みフラグ。`PATCH` 成功時に `true`。 |
| `deleted_at` | TIMESTAMPTZ | NULL | 論理削除タイムスタンプ |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | - |
| `updated_at` | TIMESTAMPTZ | NOT NULL | 楽観ロック |

**ビジネスルール**
- 編集は投稿者本人のみ（ADMIN も他人のコメントは編集不可、`403 NOT_AUTHOR`）。
- 削除（論理）は投稿者本人または ADMIN（`403 FORBIDDEN if not author nor admin`）。
- 論理削除時は `deleted_at` を設定し、レスポンスの `body` を `""` に置換、`isDeleted=true` を返す。UI 上は「削除されたコメント」と表示。
- 物理削除は親タスクの物理削除（§6.11.5）に追随する CASCADE のみ。
- 著者ユーザーが INACTIVE になっても残置し、UI 上は「退会済みユーザー」と表示（`UserSummary.status` で判別）。

**インデックス**: `(task_id, created_at)` ★コメント一覧（昇順）、`(author_id)`。

#### 5.3.7 `notifications`

通知履歴 + Outbox レコードを兼ねる。承諾通知（R-09）など主要イベントは「タスク更新 + 通知レコード作成 + pg-boss ジョブ投入」を単一トランザクションで実行する（§3.4 / §6.11.9）。アプリ内通知センター（将来）の永続化先も兼ねる。

| カラム | 型 | 制約 | 説明 |
|---|---|---|---|
| `id` | TEXT | PK | `cuid()` |
| `recipient_id` | TEXT | NOT NULL, FK→`users.id` (ON DELETE CASCADE) | 通知受信者 |
| `type` | NotificationType | NOT NULL | 通知種別（下記 enum） |
| `channel` | NotificationChannel | NOT NULL, DEFAULT `EMAIL` | `EMAIL` / `IN_APP`。初期は実質 `EMAIL` のみ稼働。 |
| `status` | NotificationStatus | NOT NULL, DEFAULT `PENDING` | `PENDING` / `SENT` / `FAILED` |
| `payload` | JSONB | NOT NULL, DEFAULT `'{}'` | 文脈情報（タスクタイトル、依頼者名、メッセージ等）。テンプレートレンダリングに利用。 |
| `task_id` | TEXT | NULL, FK→`tasks.id` (ON DELETE CASCADE) | 関連タスク。タスク物理削除に追随。 |
| `project_id` | TEXT | NULL, FK→`projects.id` (ON DELETE SET NULL) | 関連プロジェクト |
| `actor_id` | TEXT | NULL, FK→`users.id` (ON DELETE SET NULL) | 通知を引き起こした主体（依頼者・承諾者など） |
| `read_at` | TIMESTAMPTZ | NULL | 通知センター UI での既読タイムスタンプ（将来） |
| `sent_at` | TIMESTAMPTZ | NULL | 送信成功タイムスタンプ |
| `failed_at` | TIMESTAMPTZ | NULL | 送信失敗タイムスタンプ |
| `error_message` | TEXT | NULL | 失敗時のエラーメッセージ |
| `job_id` | TEXT | NULL | pg-boss ジョブ ID（突合用） |
| `created_at` | TIMESTAMPTZ | NOT NULL, DEFAULT `now()` | - |
| `updated_at` | TIMESTAMPTZ | NOT NULL | - |

**`NotificationType` enum**（初期実装）

| 値 | 発火タイミング | 受信者 |
|---|---|---|
| `TASK_REQUESTED` | §6.11.7 タスク依頼 | 被依頼者（`requested_to_id`） |
| `TASK_ACCEPTED` | §6.11.9 承諾（R-09） | 依頼者（`requested_by_id`） |
| `TASK_DECLINED` | §6.11.10 辞退 | 依頼者 |
| `TASK_REQUEST_CANCELLED` | §6.11.8 取り下げ | 被依頼者 |
| `PROJECT_MEMBER_ADDED` | §6.10.2 メンバー追加 | 追加されたユーザー |

> **注:** 招待メール（§6.7.3）は受信者が未だ User として存在しないため Notification テーブル経由とせず、pg-boss に直接ジョブを投入する設計とする（`notifications.recipient_id` の NOT NULL 制約と矛盾するため）。送信履歴は `invitations.created_at` / `invitations.updated_at` で代替する。

**ビジネスルール**
- 永続保持（履歴）。バッチ削除ポリシーは導入しない（運用判断で将来追加）。
- pg-boss が SMTP 送信を完了した時点で `status=SENT`、`sent_at=now()` を更新する（Outbox の対応関係を `job_id` で保つ）。
- 送信失敗時は `status=FAILED` で `error_message` を記録。再送はジョブのリトライ機構（pg-boss）に委ねる。

**インデックス**: `(recipient_id, created_at)` ★通知一覧（降順）、`(status, created_at)` ★失敗・未送信の運用検知。

---

### 5.4 削除・無効化ポリシー

主要リソースの削除方針を一覧する。「論理削除」「ハード削除」「物理削除（CASCADE）」を明確に分離し、CASCADE / RESTRICT / SET NULL の各 ON DELETE ルールも併記する。

| リソース | 方針 | トリガ | 連鎖アクション | 残置するもの |
|---|---|---|---|---|
| `User` | **論理削除のみ**。`status=INACTIVE` + `deleted_at=now()` | §6.7.5 `DELETE /users/:id`（ADMIN） | 同一トランザクション内: ①全セッション失効、②担当中タスクを `UNASSIGNED` に戻す、③被依頼中 `REQUESTED` を自動取り下げ | コメント、通知履歴、所属（`ProjectMember`）、招待履歴。UI 上は「退会済みユーザー」と表示。 |
| `User`（再有効化） | `status=ACTIVE` + `deleted_at=NULL` | §6.7.4 `PATCH /users/:id` | なし | - |
| `Invitation`（未消化／期限切れ） | **ハード削除可** | §6.8.3 `DELETE /invitations/:id`（ADMIN） | なし | - |
| `Invitation`（受諾済） | **削除不可**（監査用途） | - | - | レコードを永続保持。 |
| `Project` | **論理削除（アーカイブ）**。`archived_at=now()` | §6.9.5 `POST /projects/:id/archive`（ADMIN + メンバー） | なし（配下タスクは影響なし、ボードからは消えない） | 配下の Task / Comment / Notification はすべて残置。 |
| `Project`（解除） | `archived_at=NULL` | §6.9.6 `POST /projects/:id/unarchive` | なし | - |
| `ProjectMember` | **ハード削除** | §6.10.3 `DELETE /projects/:projectId/members/:userId`（ADMIN + メンバー） | 同一トランザクション内: ①当該プロジェクト内で担当中のタスクを `UNASSIGNED` に戻す、②被依頼中 `REQUESTED` を自動取り下げ | コメント、通知履歴。過去の担当履歴自体は保持されない（必要となれば将来の監査ログ機能 §10.2 で対応）。 |
| `Task` | **物理削除（ハード）** | §6.11.5 `DELETE /tasks/:id`（ADMIN + メンバー） | CASCADE で `comments`（論理削除済を含む）と `notifications`（`task_id` 経由）を物理削除 | 何も残らない。誤削除リスクのため ADMIN 限定。 |
| `Comment` | **論理削除**。`deleted_at=now()`、API レスポンスでは `body=""` で返す | §6.12.4 `DELETE /comments/:id`（投稿者 or ADMIN） | なし | レコード自体は残置（議論の継続性を担保）。 |
| `Comment`（物理） | **タスク物理削除時のみ CASCADE で消滅** | §6.11.5 の連鎖 | - | - |
| `Notification` | **永続保持**（明示的削除なし） | - | - | バッチ削除は初期未導入。 |
| `Session`（`connect-pg-simple`） | **TTL ベースのハード削除** | ライブラリ自動 + ログアウト / パスワード変更時の手動失効 | - | - |
| `pg-boss` ジョブレコード | **完了後の保存期間で自動消滅**（pg-boss 設定に準拠） | - | - | - |

**ON DELETE 整理**

| 子テーブル.外部キー | 親 | ON DELETE | 理由 |
|---|---|---|---|
| `project_members.project_id` | `projects.id` | CASCADE | プロジェクトが物理削除されれば所属レコードも不要（運用上は archive のみで物理削除は発生しないが安全側）。 |
| `project_members.user_id` | `users.id` | RESTRICT | User は論理削除のみ。物理削除は禁止しデータ整合を守る。 |
| `tasks.project_id` | `projects.id` | CASCADE | 物理削除時の整合性保持。 |
| `tasks.assignee_id` / `requested_to_id` / `requested_by_id` | `users.id` | SET NULL | 想定外の物理削除に対する防御（通常は論理削除で到達しない経路）。 |
| `comments.task_id` | `tasks.id` | CASCADE | タスク物理削除（§6.11.5）と整合。 |
| `comments.author_id` | `users.id` | RESTRICT | 投稿履歴を守る。論理削除運用と整合。 |
| `notifications.recipient_id` | `users.id` | CASCADE | 受信者が物理削除されれば通知は無意味（通常運用では論理削除のみ）。 |
| `notifications.task_id` | `tasks.id` | CASCADE | タスク物理削除と整合。 |
| `notifications.project_id` | `projects.id` | SET NULL | プロジェクト消失でも履歴は残す。 |
| `notifications.actor_id` | `users.id` | SET NULL | 通知履歴を守る。 |
| `invitations.invited_by_id` | `users.id` | RESTRICT | 招待者の論理削除運用と整合。 |

---

### 5.5 Prisma スキーマ

以下を `apps/api/prisma/schema.prisma` として配置する。`@map` / `@@map` で DB の snake_case と TS の camelCase を橋渡しする。

```prisma
// =====================================================================
//  TaskBoard — Prisma Schema
//  PostgreSQL 16 / timezone='Asia/Tokyo' / TIMESTAMPTZ
//  Single source of DB schema. BE/FE の型は packages/shared (Zod) で共有。
// =====================================================================

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ---------------------------------------------------------------------
//  Enums
// ---------------------------------------------------------------------

enum UserRole {
  ADMIN
  MEMBER
}

enum UserStatus {
  ACTIVE
  INACTIVE
}

enum TaskStatus {
  TODO
  IN_PROGRESS
  DONE
}

enum TaskAssignmentStatus {
  UNASSIGNED
  REQUESTED
  ACCEPTED
}

enum NotificationType {
  TASK_REQUESTED
  TASK_ACCEPTED
  TASK_DECLINED
  TASK_REQUEST_CANCELLED
  PROJECT_MEMBER_ADDED
}

enum NotificationChannel {
  EMAIL
  IN_APP
}

enum NotificationStatus {
  PENDING
  SENT
  FAILED
}

// ---------------------------------------------------------------------
//  User
// ---------------------------------------------------------------------

model User {
  id           String     @id @default(cuid())
  email        String     @unique
  name         String
  passwordHash String     @map("password_hash")
  role         UserRole   @default(MEMBER)
  status       UserStatus @default(ACTIVE)
  avatarUrl    String?    @map("avatar_url")
  deletedAt    DateTime?  @map("deleted_at") @db.Timestamptz(3)
  createdAt    DateTime   @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt    DateTime   @updatedAt @map("updated_at") @db.Timestamptz(3)

  // Relations
  projectMembers       ProjectMember[]
  assignedTasks        Task[]          @relation("TaskAssignee")
  requestedToTasks     Task[]          @relation("TaskRequestedTo")
  requestedByTasks     Task[]          @relation("TaskRequestedBy")
  comments             Comment[]
  invitationsSent      Invitation[]    @relation("InvitedBy")
  notificationsReceived Notification[] @relation("NotificationRecipient")
  notificationsActed   Notification[]  @relation("NotificationActor")

  @@index([status])
  @@index([role])
  @@index([deletedAt])
  @@map("users")
}

// ---------------------------------------------------------------------
//  Invitation
// ---------------------------------------------------------------------

model Invitation {
  id           String    @id @default(cuid())
  email        String
  role         UserRole
  tokenHash    String    @unique @map("token_hash")
  invitedById  String    @map("invited_by_id")
  expiresAt    DateTime  @map("expires_at") @db.Timestamptz(3)
  acceptedAt   DateTime? @map("accepted_at") @db.Timestamptz(3)
  createdAt    DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt    DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)

  invitedBy User @relation("InvitedBy", fields: [invitedById], references: [id], onDelete: Restrict)

  @@index([email])
  @@index([acceptedAt])
  @@index([expiresAt])
  @@map("invitations")
}

// ---------------------------------------------------------------------
//  Project
// ---------------------------------------------------------------------

model Project {
  id          String    @id @default(cuid())
  name        String
  description String?
  archivedAt  DateTime? @map("archived_at") @db.Timestamptz(3)
  createdAt   DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt   DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)

  members       ProjectMember[]
  tasks         Task[]
  notifications Notification[]

  @@index([archivedAt])
  @@map("projects")
}

// ---------------------------------------------------------------------
//  ProjectMember (join table, composite PK)
// ---------------------------------------------------------------------

model ProjectMember {
  projectId String   @map("project_id")
  userId    String   @map("user_id")
  joinedAt  DateTime @default(now()) @map("joined_at") @db.Timestamptz(3)

  project Project @relation(fields: [projectId], references: [id], onDelete: Cascade)
  user    User    @relation(fields: [userId], references: [id], onDelete: Restrict)

  @@id([projectId, userId])
  @@index([userId])
  @@map("project_members")
}

// ---------------------------------------------------------------------
//  Task
//
//  Invariants (Service layer enforced, optional DB CHECK constraint):
//    UNASSIGNED : assignee_id IS NULL AND requested_to_id IS NULL AND requested_by_id IS NULL
//    REQUESTED  : assignee_id IS NULL AND requested_to_id IS NOT NULL AND requested_by_id IS NOT NULL
//    ACCEPTED   : assignee_id IS NOT NULL AND requested_to_id IS NULL AND requested_by_id IS NULL
//
//  position: 0-based integer, unique within (project_id, status) by convention only
//            (no DB unique constraint; last-writer-wins for R-05 responsiveness).
// ---------------------------------------------------------------------

model Task {
  id               String               @id @default(cuid())
  projectId        String               @map("project_id")
  title            String
  description      String?
  status           TaskStatus           @default(TODO)
  assignmentStatus TaskAssignmentStatus @default(UNASSIGNED) @map("assignment_status")
  assigneeId       String?              @map("assignee_id")
  requestedToId    String?              @map("requested_to_id")
  requestedById    String?              @map("requested_by_id")
  requestMessage   String?              @map("request_message")
  dueDate          DateTime?            @map("due_date") @db.Date
  position         Int                  @default(0)
  createdAt        DateTime             @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt        DateTime             @updatedAt @map("updated_at") @db.Timestamptz(3)

  project       Project        @relation(fields: [projectId], references: [id], onDelete: Cascade)
  assignee      User?          @relation("TaskAssignee", fields: [assigneeId], references: [id], onDelete: SetNull)
  requestedTo   User?          @relation("TaskRequestedTo", fields: [requestedToId], references: [id], onDelete: SetNull)
  requestedBy   User?          @relation("TaskRequestedBy", fields: [requestedById], references: [id], onDelete: SetNull)
  comments      Comment[]
  notifications Notification[]

  @@index([projectId, status, position])
  @@index([assigneeId, status])
  @@index([requestedToId])
  @@index([dueDate])
  @@map("tasks")
}

// ---------------------------------------------------------------------
//  Comment (logical delete via deletedAt; body replaced with "" in API)
// ---------------------------------------------------------------------

model Comment {
  id        String    @id @default(cuid())
  taskId    String    @map("task_id")
  authorId  String    @map("author_id")
  body      String
  isEdited  Boolean   @default(false) @map("is_edited")
  deletedAt DateTime? @map("deleted_at") @db.Timestamptz(3)
  createdAt DateTime  @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt DateTime  @updatedAt @map("updated_at") @db.Timestamptz(3)

  task   Task @relation(fields: [taskId], references: [id], onDelete: Cascade)
  author User @relation(fields: [authorId], references: [id], onDelete: Restrict)

  @@index([taskId, createdAt])
  @@index([authorId])
  @@map("comments")
}

// ---------------------------------------------------------------------
//  Notification (history + Outbox for pg-boss email jobs)
// ---------------------------------------------------------------------

model Notification {
  id           String              @id @default(cuid())
  recipientId  String              @map("recipient_id")
  type         NotificationType
  channel      NotificationChannel @default(EMAIL)
  status       NotificationStatus  @default(PENDING)
  payload      Json                @default("{}") @db.JsonB
  taskId       String?             @map("task_id")
  projectId    String?             @map("project_id")
  actorId      String?             @map("actor_id")
  readAt       DateTime?           @map("read_at") @db.Timestamptz(3)
  sentAt       DateTime?           @map("sent_at") @db.Timestamptz(3)
  failedAt     DateTime?           @map("failed_at") @db.Timestamptz(3)
  errorMessage String?             @map("error_message")
  jobId        String?             @map("job_id")
  createdAt    DateTime            @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt    DateTime            @updatedAt @map("updated_at") @db.Timestamptz(3)

  recipient User     @relation("NotificationRecipient", fields: [recipientId], references: [id], onDelete: Cascade)
  actor     User?    @relation("NotificationActor", fields: [actorId], references: [id], onDelete: SetNull)
  task      Task?    @relation(fields: [taskId], references: [id], onDelete: Cascade)
  project   Project? @relation(fields: [projectId], references: [id], onDelete: SetNull)

  @@index([recipientId, createdAt])
  @@index([status, createdAt])
  @@map("notifications")
}
```

#### 5.5.1 Prisma 管理外テーブルに関する補足

以下のテーブルは Prisma のスコープ外として共存させる。マイグレーションは各ライブラリ側で行われる。

- **`session` テーブル**: `connect-pg-simple` がアプリ起動時に `CREATE TABLE IF NOT EXISTS` で生成。スキーマは `(sid TEXT PK, sess JSON, expire TIMESTAMP(6))`。Prisma の `schema.prisma` には含めず、ライブラリの管理に委ねる。`prisma db pull` 時に取り込まれないよう `.prisma/ignore` を使うか、`schema.prisma` 側で `previewFeatures` の `multiSchema` 等で分離する運用とする。
- **`pgboss.*` スキーマ**: `pg-boss` が独自スキーマ `pgboss` 配下に `job` 等を作成。Prisma DB（同一 PostgreSQL 物理インスタンス）に同居するが、論理的に分離されている。

#### 5.5.2 DB レベルの CHECK 制約（推奨追加マイグレーション）

`tasks.assignment_status` と担当系カラムの整合性は Service 層で保証するが、二重防御として以下の CHECK 制約を migration に追加することを推奨する。

```sql
ALTER TABLE tasks
ADD CONSTRAINT chk_task_assignment_consistency CHECK (
  (assignment_status = 'UNASSIGNED'
    AND assignee_id IS NULL
    AND requested_to_id IS NULL
    AND requested_by_id IS NULL)
  OR
  (assignment_status = 'REQUESTED'
    AND assignee_id IS NULL
    AND requested_to_id IS NOT NULL
    AND requested_by_id IS NOT NULL)
  OR
  (assignment_status = 'ACCEPTED'
    AND assignee_id IS NOT NULL
    AND requested_to_id IS NULL
    AND requested_by_id IS NULL)
);
```

また、ユーザー無効化と削除時刻の整合性も推奨:

```sql
ALTER TABLE users
ADD CONSTRAINT chk_user_status_deleted_at CHECK (
  (status = 'ACTIVE'   AND deleted_at IS NULL) OR
  (status = 'INACTIVE' AND deleted_at IS NOT NULL)
);
```

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

採用バージョンは執筆時点の安定版を基準とする。マイナーバージョンの更新は通常通り追従する。

### 7.1 フロントエンド

| 領域 | 採用技術 | バージョン目安 | 用途 |
|---|---|---|---|
| 言語 | TypeScript | 5.x | `strict: true` で運用 |
| ビルド | Vite | 5.x | 開発サーバ（HMR）、本番ビルド |
| UI フレームワーク | React | 18.x | 関数コンポーネント中心 |
| ルーティング | React Router | 6.x | ネスト型、`<RequireAuth>` / `<RequireRole>` を §4.3 で利用 |
| サーバ状態 | TanStack Query | 5.x | API レスポンスのキャッシュ、楽観更新、エラーリトライ |
| クライアント状態 | Zustand | 4.x | カンバンの D&D 一時状態、トースト等。最小限の利用 |
| スタイリング | Tailwind CSS + shadcn/ui | Tailwind 3.x | shadcn/ui のコンポーネントは `src/components/ui/` 配下にコピーして管理 |
| フォーム | react-hook-form + zodResolver | 7.x | `packages/shared` の Zod スキーマで入力検証 |
| ドラッグ&ドロップ | dnd-kit | 6.x | R-05 のカンバン D&D |
| 日時 | date-fns + date-fns-tz | 3.x | JST 固定、`+09:00` のフォーマット |
| アイコン | lucide-react | 最新 | UI アイコン全般 |
| ユニットテスト | Vitest + React Testing Library | Vitest 1.x | コンポーネント・フック・ストアのテスト |
| E2E | Playwright | 1.x | クロスブラウザの主要シナリオテスト |

### 7.2 バックエンド

| 領域 | 採用技術 | バージョン目安 | 用途 |
|---|---|---|---|
| 言語 | TypeScript | 5.x | `strict: true` で運用 |
| フレームワーク | NestJS | 10.x | DI、モジュール構成、Guard / Interceptor / Filter |
| HTTP プラットフォーム | Express | 4.x | NestJS の既定アダプタ |
| ORM | Prisma | 5.x | スキーマ管理、マイグレーション、型生成 |
| バリデーション | nestjs-zod | 最新 | `packages/shared` の Zod スキーマを `ZodValidationPipe` で検証 |
| API ドキュメント | @nestjs/swagger | 7.x | `/api/docs` で OpenAPI を公開、CI で `openapi.json` 出力 |
| セッション | express-session + connect-pg-simple | 1.x / 9.x | PostgreSQL の `session` テーブルで永続化 |
| 認可 | カスタム Guard | - | `SessionAuthGuard` / `CsrfGuard` / `RolesGuard` / `ProjectMemberGuard`（§2.2） |
| CSRF | csurf 等は採用せず自前実装 | - | Double Submit Cookie（§6.17） |
| 非同期ジョブ | pg-boss | 9.x | PostgreSQL 駆動のジョブキュー、Outbox の送信側 |
| メール（送信ライブラリ） | @nestjs-modules/mailer + Nodemailer + Handlebars | 最新 | テンプレートエンジン Handlebars でメール本文生成 |
| メール送信基盤 | Amazon SES（本番）／ Mailpit（開発） | - | 本番は SES、開発は Mailpit のローカル SMTP（§9.4） |
| レートリミット | @nestjs/throttler | 5.x | 認証系エンドポイントに適用（§6.17） |
| パスワードハッシュ | bcrypt | 5.x | コスト係数 12 |
| ロギング | pino + nestjs-pino | 最新 | 構造化 JSON ログ |
| ヘッダ強化 | helmet | 最新 | CSP、X-Frame-Options 等 |
| テスト | Jest + Supertest | Jest 29.x | ユニット / E2E（API レベル） |

### 7.3 データベース・インフラ

| 領域 | 採用技術 | バージョン目安 | 用途 |
|---|---|---|---|
| RDBMS | PostgreSQL | 16.x | アプリ DB（`timezone='Asia/Tokyo'`、`TIMESTAMPTZ`） |
| セッションストア | PostgreSQL（同居）／ `connect-pg-simple` 管理の `session` テーブル | - | §5.5.1 |
| ジョブキュー基盤 | PostgreSQL（同居）／ `pgboss` スキーマ | - | §5.5.1 |
| メール送信（本番） | Amazon SES | - | Region は要件次第（東京リージョン想定） |
| メール送信（開発） | Mailpit | 最新 | SMTP `1025`、UI `8025`（§9.4） |
| Redis | **不採用** | - | キャッシュもセッションも PostgreSQL 同居で十分とする方針 |

### 7.4 開発支援

| 領域 | 採用技術 | バージョン目安 | 用途 |
|---|---|---|---|
| Monorepo | pnpm workspaces + Turborepo | pnpm 9.x / Turborepo 2.x | タスク並列実行・キャッシュ |
| Lint / Format | ESLint + Prettier | ESLint 9.x | `eslint-config-prettier`・`@typescript-eslint`・React Hooks ルール適用 |
| 型 / スキーマ共有 | TypeScript + Zod（`packages/shared`） | Zod 3.x | BE / FE 双方で参照する Single Source of Truth |
| Git フック | husky + lint-staged | 最新 | コミット前 Lint / 型チェック |
| コミットメッセージ | Conventional Commits | - | 規約準拠（任意） |
| 開発コンテナ | Docker / docker compose | 最新 | PostgreSQL 16、Mailpit を提供 |
| OpenAPI 整合 | CI ジョブ | - | `openapi.json` と `packages/shared` の Zod スキーマを照合（§6.16） |

---

## 8. ディレクトリ構成

機能別（feature-based）構成を採用する。`apps/{web, api}` と `packages/shared` の 3 ワークスペースを基本とする。`features/`（FE）と `modules/`（BE）配下は、§3 の機能と §6 の API リソースに対応する単位で分割する。

```
.
├── apps/
│   ├── web/                              # フロントエンド (React + Vite)
│   │   ├── src/
│   │   │   ├── features/                 # 機能単位
│   │   │   │   ├── auth/                 # ログイン、招待受諾、CSRF 取得
│   │   │   │   │   ├── api/              #   TanStack Query フック
│   │   │   │   │   ├── components/
│   │   │   │   │   ├── hooks/
│   │   │   │   │   └── pages/            #   /login, /invite/:token
│   │   │   │   ├── me/                   # /me：プロフィール、パスワード変更
│   │   │   │   ├── projects/             # /projects, /projects/:id, /projects/:id/settings
│   │   │   │   ├── tasks/                # カンバン、タスク詳細モーダル、D&D
│   │   │   │   │   ├── api/
│   │   │   │   │   ├── components/       #   KanbanBoard, TaskCard, TaskDetailModal
│   │   │   │   │   ├── hooks/            #   useTaskDragAndDrop など
│   │   │   │   │   └── pages/
│   │   │   │   ├── comments/             # タスク詳細モーダル内のコメント
│   │   │   │   ├── notifications/        # 将来の通知センター用の場所取り
│   │   │   │   └── admin/                # /admin/dashboard, /admin/users, /admin/invitations
│   │   │   ├── components/               # 横断 UI (shadcn/ui ラッパー、Toast、Modal 等)
│   │   │   ├── lib/                      # 横断ユーティリティ (apiClient, dateFns ラッパー)
│   │   │   ├── routes/                   # React Router 定義、RequireAuth / RequireRole
│   │   │   ├── stores/                   # Zustand ストア
│   │   │   └── main.tsx
│   │   ├── public/
│   │   ├── index.html
│   │   ├── tailwind.config.ts
│   │   └── vite.config.ts
│   │
│   └── api/                              # バックエンド (NestJS)
│       ├── src/
│       │   ├── modules/                  # 機能モジュール
│       │   │   ├── auth/                 #   §6.5：ログイン、招待検証・受諾
│       │   │   ├── me/                   #   §6.6：自プロフィール、パスワード変更
│       │   │   ├── users/                #   §6.7：ユーザー管理 (ADMIN)
│       │   │   ├── invitations/          #   §6.8：招待管理 (ADMIN)
│       │   │   ├── projects/             #   §6.9：プロジェクト
│       │   │   ├── project-members/      #   §6.10：メンバー管理
│       │   │   ├── tasks/                #   §6.11：タスク、状態機械、position
│       │   │   │   ├── tasks.controller.ts
│       │   │   │   ├── tasks.service.ts
│       │   │   │   ├── tasks.module.ts
│       │   │   │   └── dto/              #     Zod スキーマからの型エイリアス
│       │   │   ├── comments/             #   §6.12：コメント
│       │   │   ├── notifications/        #   §3.6 / §5.3.7：Notification + pg-boss ワーカ
│       │   │   └── admin/                #   §6.13：ダッシュボード集計
│       │   ├── common/                   # 横断
│       │   │   ├── guards/               #   SessionAuthGuard, CsrfGuard, RolesGuard, ProjectMemberGuard
│       │   │   ├── interceptors/         #   レスポンスエンベロープ整形
│       │   │   ├── filters/              #   全例外を §6.2 のエラーエンベロープに変換
│       │   │   └── decorators/           #   @Roles, @CurrentUser, @ProjectId 等
│       │   ├── infra/                    # 外部 I/O
│       │   │   ├── prisma/               #   PrismaService、トランザクションヘルパ
│       │   │   ├── mailer/               #   メーラ設定、Handlebars テンプレート
│       │   │   ├── pg-boss/              #   ジョブキュー設定、ワーカ登録
│       │   │   └── session/              #   express-session + connect-pg-simple 設定
│       │   ├── config/                   # 環境変数バインディング
│       │   └── main.ts                   # bootstrap、Swagger 設定、ミドルウェア
│       ├── prisma/
│       │   ├── schema.prisma             # §5.5 の最終版
│       │   ├── migrations/               # マイグレーション履歴
│       │   └── seed.ts                   # 初期 ADMIN・サンプルデータ投入
│       └── test/                         # Jest + Supertest による E2E
│
├── packages/
│   └── shared/                           # BE/FE 共有 Zod スキーマ・型・定数
│       ├── src/
│       │   ├── schemas/                  # Zod スキーマ (機能別)
│       │   │   ├── auth.ts
│       │   │   ├── users.ts
│       │   │   ├── invitations.ts
│       │   │   ├── projects.ts
│       │   │   ├── project-members.ts
│       │   │   ├── tasks.ts
│       │   │   ├── comments.ts
│       │   │   └── admin.ts
│       │   ├── types/                    # Zod から派生する TS 型エクスポート
│       │   ├── constants/                # enum、エラーコード、バリデーション上限値
│       │   └── index.ts
│       └── package.json
│
├── docker-compose.yml                    # PostgreSQL 16、Mailpit
├── turbo.json                            # Turborepo パイプライン
├── pnpm-workspace.yaml
├── package.json
├── .env.example                          # ルートに集約された環境変数の例
└── MANIFEST.md
```

---

## 9. 開発環境構築

### 9.1 前提

| 項目 | 推奨 |
|---|---|
| Node.js | LTS（v20 系以上） |
| pnpm | v9 系 |
| Docker / Docker Compose | 最新の安定版 |
| Git | 任意 |
| OS | macOS / Linux / WSL2 のいずれか |

### 9.2 初回セットアップ手順

1. リポジトリをクローンする。
2. 依存関係をインストールする: `pnpm install`
3. 環境変数ファイルを配置する: `cp .env.example .env`（`apps/api` と `apps/web` に同様に複製、または Turborepo の env パススルー設定で集中管理）
4. ローカル PostgreSQL / Mailpit を起動する: `docker compose up -d`
5. Prisma マイグレーションを適用する: `pnpm --filter @app/api prisma migrate dev`
6. シードデータを投入する: `pnpm --filter @app/api prisma db seed`
7. 開発サーバを起動する: `pnpm dev`（Turborepo で `web` と `api` を並列起動）
8. ブラウザで `http://localhost:5173` を開き、シードで作成された初期 ADMIN でログインする。

### 9.3 環境変数

> 機密値（`SESSION_SECRET`、`CSRF_SECRET`、`SES_*` 等）はリポジトリにコミットせず、本番では Secrets Manager 等から注入する。

#### 共通

| 変数 | 用途 | 既定 / 例 |
|---|---|---|
| `NODE_ENV` | 環境種別 | `development` / `production` |
| `TZ` | プロセスタイムゾーン | `Asia/Tokyo` |

#### `apps/api`

| 変数 | 用途 | 既定 / 例 |
|---|---|---|
| `PORT` | API リッスンポート | `3000` |
| `APP_BASE_URL` | 招待 URL 等の生成に使う公開 URL | `http://localhost:5173`（開発）／ 本番 URL |
| `DATABASE_URL` | PostgreSQL 接続文字列 | `postgresql://app:app@localhost:5432/app?schema=public` |
| `SESSION_SECRET` | セッション署名鍵（十分長いランダム値） | （秘匿） |
| `SESSION_NAME` | セッション Cookie 名 | `tb_session` |
| `SESSION_MAX_AGE_HOURS` | セッション有効期間 | `24` |
| `CSRF_SECRET` | CSRF トークン生成鍵 | （秘匿） |
| `COOKIE_DOMAIN` | Cookie ドメイン | `localhost`（開発）／ 本番ドメイン |
| `COOKIE_SECURE` | `Secure` 属性 | `false`（開発）／ `true`（本番） |
| `BCRYPT_COST` | bcrypt コスト係数 | `12` |
| `INVITATION_TTL_DAYS` | 招待トークン有効期限 | `7` |
| `MAIL_TRANSPORT` | メール送信トランスポート | `smtp`（開発）／ `ses`（本番） |
| `MAIL_HOST` | SMTP ホスト | `localhost` |
| `MAIL_PORT` | SMTP ポート | `1025` |
| `MAIL_USER` | SMTP ユーザー（必要時） | -（Mailpit は不要） |
| `MAIL_PASS` | SMTP パスワード（必要時） | -（Mailpit は不要） |
| `MAIL_FROM` | 送信元アドレス | `noreply@example.com` |
| `SES_REGION` | SES リージョン（本番） | `ap-northeast-1` |
| `SES_ACCESS_KEY_ID` | SES 認証 | -（本番） |
| `SES_SECRET_ACCESS_KEY` | SES 認証 | -（本番） |
| `PGBOSS_SCHEMA` | pg-boss 用スキーマ名 | `pgboss` |
| `LOG_LEVEL` | pino ログレベル | `info` |
| `THROTTLE_*` | 認証系レートリミットの調整 | §6.17 の既定に従う |

#### `apps/web`

| 変数 | 用途 | 既定 / 例 |
|---|---|---|
| `VITE_API_BASE_URL` | 開発時のみ。本番は同一オリジン運用のため未設定 | 開発: `http://localhost:3000` |

### 9.4 起動ポート（既定）

| サービス | ポート |
|---|---|
| Web（Vite） | `5173` |
| API（NestJS） | `3000` |
| PostgreSQL | `5432` |
| Mailpit UI | `8025` |
| Mailpit SMTP | `1025` |

### 9.5 シードデータ（`prisma/seed.ts`）

| 種別 | 内容 |
|---|---|
| 初期 ADMIN | `admin@example.com` / 開発用パスワードを bcrypt ハッシュ化して投入 |
| サンプル MEMBER | 数名分の `MEMBER` ロールユーザー |
| サンプルプロジェクト | 1〜2 件、初期 ADMIN とサンプル MEMBER を `ProjectMember` として登録 |
| サンプルタスク | 各列（`TODO` / `IN_PROGRESS` / `DONE`）に数件、`assignment_status` の全状態を網羅 |
| サンプルコメント | 数件 |

> 本番環境ではシードを実行しない（マイグレーションのみ）。本番の初期 ADMIN は手動 SQL または専用の管理スクリプトで投入する。

### 9.6 主要スクリプト（Turborepo タスク）

| コマンド | 役割 |
|---|---|
| `pnpm dev` | 全アプリ並列起動（`web` と `api`） |
| `pnpm build` | 全アプリビルド |
| `pnpm lint` | 全ワークスペース Lint（ESLint） |
| `pnpm format` | Prettier フォーマット適用 |
| `pnpm typecheck` | 全ワークスペース `tsc --noEmit` による型チェック |
| `pnpm test` | 全ワークスペーステスト（Vitest / Jest） |
| `pnpm e2e` | Playwright による FE E2E |
| `pnpm --filter @app/api prisma migrate dev` | DB マイグレーション適用 |
| `pnpm --filter @app/api prisma db seed` | シード投入 |
| `pnpm --filter @app/api openapi:export` | `/api/docs` から `openapi.json` を出力（CI で `packages/shared` と整合性検証） |

### 9.7 メール送信の確認（開発時）

開発環境では Mailpit を使用する。アプリから送信されたメールは外部に出ず、`http://localhost:8025` の UI で受信箱として確認できる。これにより、招待メール（§6.7.3）や承諾通知メール（§6.11.9）の動作確認を SES なしで実施できる。

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

> 本書はアプリケーション設計の最終版である。実装段階で本書に記載のない判断が必要となった場合は、当該判断の決定後に本書を更新し、整合性を保つ運用とする。
