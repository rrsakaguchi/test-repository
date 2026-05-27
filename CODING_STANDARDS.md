# コーディング規約（CODING STANDARDS）

> 本書は `MANIFEST.md` で定義された技術スタックとディレクトリ構成に基づく、開発チーム向けのコーディング規約である。実装中に判断に迷ったときの第一参照点とし、規約に反する PR はレビューで差し戻す運用とする。例外を許容する場合は、コード上のコメントまたは PR コメントで根拠（MANIFEST §X.Y との対応や、明示的なトレードオフ）を必ず示すこと。

---

## 目次

1. [全般的な言語規約](#1-全般的な言語規約)
2. [フロントエンド規約](#2-フロントエンド規約)
3. [バックエンド規約](#3-バックエンド規約)
4. [ORM / データベース規約](#4-orm--データベース規約)
5. [API 設計規約](#5-api-設計規約)
6. [テスト規約](#6-テスト規約)
7. [Git 規約](#7-git-規約)
8. [環境変数管理](#8-環境変数管理)
9. [セキュリティガイドライン](#9-セキュリティガイドライン)
10. [コードレビューチェックリスト](#10-コードレビューチェックリスト)

---

## 1. 全般的な言語規約

### 1.1 TypeScript コンパイラ設定

すべてのワークスペース（`apps/web`、`apps/api`、`packages/shared`）で **`strict: true` を必須** とする。共通ベースは `tsconfig.base.json` に集約し、各ワークスペースは `extends` で取り込む。

`tsconfig.base.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "forceConsistentCasingInFileNames": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "skipLibCheck": true
  }
}
```

- `noUncheckedIndexedAccess` により `array[i]` は `T | undefined` 型になる。インデックスアクセス後は必ず undefined チェックを行うこと。
- `apps/api` は `module: "NodeNext"` / `moduleResolution: "NodeNext"`、`apps/web` は Vite の既定（`Bundler`）でオーバライドする。

### 1.2 ESLint / Prettier 設定

ESLint は **フラットコンフィグ**（`eslint.config.js`）を使用する。必須プラグイン:

- `@typescript-eslint/eslint-plugin`
- `eslint-plugin-import` または `eslint-plugin-simple-import-sort`
- `eslint-plugin-react`・`eslint-plugin-react-hooks`・`eslint-plugin-jsx-a11y`（`apps/web` のみ）
- `eslint-config-prettier`（Prettier と競合するルールを無効化）

Prettier 設定（`.prettierrc.json`）:

```json
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

**厳格化したいルール（一例）:**

```js
// eslint.config.js から抜粋
{
  rules: {
    '@typescript-eslint/no-explicit-any': 'error',
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
    '@typescript-eslint/consistent-type-imports': ['error', { prefer: 'type-imports' }],
    '@typescript-eslint/no-floating-promises': 'error',
    '@typescript-eslint/await-thenable': 'error',
    '@typescript-eslint/no-misused-promises': 'error',
    'no-console': ['warn', { allow: ['warn', 'error'] }],
    'eqeqeq': ['error', 'always'],
    'curly': ['error', 'all'],
  },
}
```

### 1.3 型定義ルール

#### `any` は禁止、`unknown` を使う

```ts
// ❌ Bad
function parse(data: any) {
  return data.name;
}

// ✅ Good — unknown を narrowing して使う
function parse(data: unknown): string {
  if (typeof data === 'object' && data !== null && 'name' in data && typeof data.name === 'string') {
    return data.name;
  }
  throw new Error('Invalid data');
}
```

外部入力（API レスポンス、ユーザー入力、JSON.parse 結果）は **`packages/shared` の Zod スキーマで `parse`** すること。`as` キャストで型を偽装しない。

#### `interface` と `type` の使い分け

- オブジェクト形状の **公開 API**（コンポーネントの Props、サービスの公開メソッド引数）は `interface`。
- ユニオン型、交差型、ユーティリティ型、Zod から派生する型は `type`。

```ts
// ✅ Good
interface TaskCardProps {
  task: Task;
  onClick: (id: string) => void;
}

type TaskStatus = 'TODO' | 'IN_PROGRESS' | 'DONE';
type TaskWithAssignee = Task & { assignee: User };
```

#### `enum` は使わず Zod / Union を使う

```ts
// ❌ Bad — 数値 enum はランタイム表現が予期しづらく、tree-shake にも不利
enum TaskStatus { TODO, IN_PROGRESS, DONE }

// ✅ Good — Zod スキーマ（packages/shared）を Single Source of Truth とする
import { z } from 'zod';
export const TaskStatusSchema = z.enum(['TODO', 'IN_PROGRESS', 'DONE']);
export type TaskStatus = z.infer<typeof TaskStatusSchema>;
```

Prisma の `enum` はそのまま使ってよい（DB と TS で型整合が取れる）。

#### `satisfies` を活用する

```ts
// ❌ Bad — as キャストで型を偽装
const config = {
  retries: 3,
  delay: 100,
} as RetryConfig;

// ✅ Good — satisfies で型チェックを残しつつリテラル型を保持
const config = {
  retries: 3,
  delay: 100,
} satisfies RetryConfig;
```

#### Null と Undefined

API 境界では **null** を使う（`User.avatarUrl: string | null`）。関数の引数省略やオブジェクトのプロパティ未指定では **undefined** を使う。`null` と `undefined` を混在させない。

Prisma の Nullable カラムは TS では `string | null` として現れる。FE / BE の境界でもこの形を維持する（Zod スキーマで `.nullable()`、`.optional()` は使い分ける）。

### 1.4 命名規則

| 対象 | 規則 | 例 |
|---|---|---|
| 変数・関数 | `camelCase` | `userId`、`fetchTasks` |
| 真偽値変数 | `is` / `has` / `should` / `can` 接頭辞 | `isLoading`、`hasPermission`、`canEdit` |
| 型・インタフェース・クラス | `PascalCase` | `Task`、`TasksService`、`UserSummary` |
| ジェネリクス型パラメータ | 大文字 1 字または `T` 接頭辞 | `T`、`TData`、`TResult` |
| トップレベル定数（モジュール公開） | `UPPER_SNAKE_CASE` | `MAX_RETRIES`、`DEFAULT_PAGE_SIZE` |
| ローカル定数 | `camelCase` | `const pageSize = 20;` |
| React コンポーネント | `PascalCase` | `TaskCard`、`KanbanBoard` |
| Custom Hook | `use` 接頭辞・`camelCase` | `useCurrentUser`、`useTaskMutation` |
| Zod スキーマ | `XxxSchema` サフィックス | `TaskSchema`、`CreateTaskInputSchema` |
| 列挙的ユニオン値 | `UPPER_SNAKE_CASE` | `'TODO'`、`'IN_PROGRESS'` |
| エラーコード | `UPPER_SNAKE_CASE`（§6.3） | `VALIDATION_ERROR`、`PROJECT_MEMBER_REQUIRED` |
| ファイル名（BE） | `kebab-case.ts` | `tasks.service.ts`、`project-member.guard.ts` |
| ファイル名（FE コンポーネント） | `PascalCase.tsx` | `TaskCard.tsx`、`KanbanBoard.tsx` |
| ファイル名（FE その他） | `kebab-case.ts` | `use-current-user.ts`、`api-client.ts` |
| ディレクトリ名 | `kebab-case` | `project-members/`、`tasks/`、`admin/` |
| DB テーブル・カラム | `snake_case` | `users`、`assignment_status` |
| API パスセグメント | `kebab-case` 複数形（§6.1） | `/project-members`、`/tasks` |

#### 命名のアンチパターン

```ts
// ❌ Bad
const d = new Date();           // 略語不明
const userdata = ...;            // 単語境界なし
const HandleClick = () => {};    // 関数なのに PascalCase
const Task_card_props = {};      // ケース混在

// ✅ Good
const now = new Date();
const userData = { ... };
const handleClick = () => {};
const taskCardProps = { ... };
```

### 1.5 インポート順序

`eslint-plugin-import` または `eslint-plugin-simple-import-sort` で自動整形する。グループ間は **空行で区切る**。

順序:

1. Node.js 組み込みモジュール（`node:fs`、`node:path` など）
2. 外部 npm パッケージ（`react`、`@nestjs/common` など）
3. ワークスペース内エイリアス（`@app/shared`、`@/features/...` など）
4. 親ディレクトリからの相対インポート（`../`）
5. 同一ディレクトリからの相対インポート（`./`）
6. **型のみ** のインポート（`import type ...`）は最後にまとめる

```ts
// ✅ Good
import { readFile } from 'node:fs/promises';

import { Module } from '@nestjs/common';
import { z } from 'zod';

import { TaskSchema } from '@app/shared';

import { PrismaModule } from '../infra/prisma/prisma.module';

import { TasksController } from './tasks.controller';
import { TasksService } from './tasks.service';

import type { Task } from '@app/shared';
```

### 1.6 コメント方針

- **何をやっているか** ではなく、**なぜそうしたか** をコメントに書く。
- 自明な日本語訳コメント（`// ユーザーを取得する` for `const user = await getUser()`）は不要。
- TODO / FIXME には GitHub Issue 番号を必ず付ける: `// TODO(#123): position 再採番のしきい値を環境変数化する`
- 公開 API（NestJS Controller / 公開 hook / Service の public method）には JSDoc を付与し、Swagger UI と Hover ヒントに反映させる。

```ts
// ❌ Bad
// タスクを承諾する
async acceptTask(id: string) { ... }

// ✅ Good — 副作用と前提条件を明示
/**
 * 依頼中タスクの承諾を行う。被依頼者本人のみ実行可能 (§6.11.9)。
 * 単一トランザクションで Task 更新 + Notification 作成 + pg-boss ジョブ投入を行う (Outbox)。
 *
 * @throws NotRequesteeException — 自分宛の依頼でない場合
 * @throws InvalidStateTransitionException — assignment_status が REQUESTED でない場合
 */
async acceptTask(id: string, currentUserId: string): Promise<Task> { ... }
```

---

## 2. フロントエンド規約

### 2.1 ディレクトリ構成

`apps/web/src/` 配下は機能別（feature-based）構成（MANIFEST §8）。

```
apps/web/src/
├── features/
│   ├── tasks/
│   │   ├── api/                  # TanStack Query フック (useTasks, useUpdateTask, ...)
│   │   ├── components/           # PascalCase.tsx (TaskCard, KanbanBoard, TaskDetailModal)
│   │   ├── hooks/                # use-task-drag-and-drop.ts などコンポーネント横断 hook
│   │   ├── pages/                # ルートに紐づくページコンポーネント
│   │   └── index.ts              # feature の公開エクスポート
│   ├── auth/
│   ├── projects/
│   └── ...
├── components/                   # 横断 UI (Button, Modal, Toast, ...) + shadcn/ui ラッパー
│   └── ui/                       # shadcn/ui のコピー先
├── lib/                          # 横断ユーティリティ (api-client.ts, cn.ts, date.ts)
├── routes/                       # React Router 定義、RequireAuth / RequireRole
├── stores/                       # Zustand ストア
└── main.tsx
```

- **feature をまたぐインポートは原則禁止**。`features/tasks` から `features/projects` の内部実装を直接 import しない。共有が必要なものは `packages/shared`（型・スキーマ）または `apps/web/src/components`・`lib`・`stores`（UI / ユーティリティ）に昇格させる。
- ルーティング先となるページコンポーネントは `features/{name}/pages/` に置き、ルート定義は `src/routes/` から import する。

### 2.2 コンポーネント設計

#### 関数コンポーネントのみ・`React.FC` は使わない

```tsx
// ❌ Bad
const TaskCard: React.FC<TaskCardProps> = ({ task }) => { ... };

// ✅ Good — Props を引数で受ける普通の関数
interface TaskCardProps {
  task: Task;
  onClick: (id: string) => void;
}

export function TaskCard({ task, onClick }: TaskCardProps) {
  return (
    <button onClick={() => onClick(task.id)} className="...">
      {task.title}
    </button>
  );
}
```

#### Named Export を使う

ページコンポーネント等で React Router の lazy ロードと両立させたい場合のみ default export を許容する。それ以外は named export で統一する（IDE のリネーム支援・grep がしやすい）。

#### コンポーネントは単一責任に保つ

- 100 行を超えたら分割を検討する。
- データ取得（`useQuery`）と表示を 1 つのコンポーネントに同居させてよいのは **ページコンポーネント** のみ。子コンポーネントは Props 経由でデータを受け取る純粋表示にする。
- `useEffect` を多用するコンポーネントは設計が誤っている可能性が高い。サーバ状態は TanStack Query に、派生値は `useMemo` に逃がす。

```tsx
// ❌ Bad — useEffect でフェッチしている
export function TasksPage({ projectId }: Props) {
  const [tasks, setTasks] = useState<Task[]>([]);
  useEffect(() => {
    fetch(`/api/v1/projects/${projectId}/tasks`).then(r => r.json()).then(setTasks);
  }, [projectId]);
  return <KanbanBoard tasks={tasks} />;
}

// ✅ Good — TanStack Query を使う
export function TasksPage({ projectId }: Props) {
  const { data: tasks, isLoading, error } = useTasks(projectId);
  if (isLoading) return <Skeleton />;
  if (error) return <ErrorBoundary error={error} />;
  return <KanbanBoard tasks={tasks} />;
}
```

#### Props にロジックを混ぜない

```tsx
// ❌ Bad — Props でハンドラを組み立てている
<TaskCard onClick={() => { mutate({ id: task.id, status: 'DONE' }); toast.success('...'); }} />

// ✅ Good — コンポーネント内で関数を定義
function TasksPage() {
  const updateTask = useUpdateTask();
  const handleComplete = (id: string) => {
    updateTask.mutate({ id, status: 'DONE' }, {
      onSuccess: () => toast.success('完了しました'),
    });
  };
  return <TaskCard onClick={handleComplete} />;
}
```

### 2.3 カスタムフック

- **`use` 接頭辞**を必ず付ける（React の Rules of Hooks のため）。
- 戻り値は **2 つ以下なら配列、3 つ以上ならオブジェクト** を返す（`useState` の慣習に倣う／プロパティ名で意味が明確になる）。
- 1 つのフックは 1 つの責務に絞る。「タスクの取得と更新と削除」を 1 つのフックに詰め込まない。

```ts
// ✅ Good — TanStack Query の hook を feature 内で薄くラップする
// apps/web/src/features/tasks/api/use-tasks.ts
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '@/lib/api-client';
import { TaskSchema } from '@app/shared';
import { z } from 'zod';

const TasksResponseSchema = z.array(TaskSchema);

export function useTasks(projectId: string) {
  return useQuery({
    queryKey: ['projects', projectId, 'tasks'],
    queryFn: async () => {
      const res = await apiClient.get(`/projects/${projectId}/tasks`);
      return TasksResponseSchema.parse(res.data);
    },
    enabled: Boolean(projectId),
  });
}
```

```ts
// ✅ Good — Mutation hook は副作用（キャッシュ無効化）も内包する
export function useUpdateTask() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (input: UpdateTaskInput) =>
      apiClient.patch(`/tasks/${input.id}`, input).then(r => TaskSchema.parse(r.data)),
    onSuccess: (task) => {
      queryClient.invalidateQueries({ queryKey: ['projects', task.projectId, 'tasks'] });
    },
  });
}
```

### 2.4 スタイリング（Tailwind CSS + shadcn/ui）

#### Tailwind ユーティリティを優先する

カスタム CSS（`*.module.css`、`styled-components` 等）は原則禁止。Tailwind の utility だけで表現する。複雑なアニメーションのみ `tailwind.config.ts` の `keyframes` に追加する。

#### 条件付きクラスは `cn()` を使う

`clsx` + `tailwind-merge` をラップした `cn` ユーティリティを `lib/cn.ts` に用意する。

```ts
// apps/web/src/lib/cn.ts
import { clsx, type ClassValue } from 'clsx';
import { twMerge } from 'tailwind-merge';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

```tsx
// ❌ Bad — テンプレートリテラルで条件分岐
<button className={`btn ${isActive ? 'btn-active' : ''} ${isDisabled ? 'opacity-50' : ''}`} />

// ✅ Good — cn() を使い、tailwind-merge で重複ユーティリティを統合
<button
  className={cn(
    'rounded-md px-4 py-2',
    isActive && 'bg-blue-500 text-white',
    isDisabled && 'opacity-50 cursor-not-allowed',
  )}
/>
```

#### shadcn/ui コンポーネント

- `src/components/ui/` に **コピーして** 配置（npm パッケージとして取り込まない）。
- カスタマイズは直接編集で行う。リネームしない（公式ドキュメントと突き合わせやすくするため）。
- 業務固有のコンポーネント（`TaskCard` 等）は `src/components/ui/` ではなく `features/{name}/components/` に置く。

#### デザイントークン

`tailwind.config.ts` の `theme.extend.colors` で意味付き名（`primary`、`destructive` 等）を定義し、生のカラーパレット（`blue-500` 等）を直接使う頻度を下げる。

### 2.5 状態管理

#### サーバ状態は TanStack Query

API から取得するすべてのデータは TanStack Query 経由とする。`useState` でサーバ状態を保持しない。

#### `queryKey` の規約

階層的に組み立て、不要な再フェッチを避ける。

```ts
// ✅ Good — リソース階層をキー配列で表現
['projects']                                  // 一覧
['projects', projectId]                       // 詳細
['projects', projectId, 'tasks']              // 配下のタスク一覧
['projects', projectId, 'tasks', taskId]      // 個別タスク
['projects', projectId, 'members']            // メンバー一覧
['me']                                        // 自分の情報
['admin', 'dashboard', 'summary']             // 管理者ダッシュボード
```

`invalidateQueries({ queryKey: ['projects', projectId] })` で配下すべてを無効化できるため、階層設計を最初から崩さない。

#### Optimistic Update

R-05 のドラッグ&ドロップ（**dnd-kit** で実装）など即時反映が必要な箇所では Optimistic Update を使う。失敗時は必ず `onError` でロールバックすること。

```ts
useMutation({
  mutationFn: updateTaskPosition,
  onMutate: async (input) => {
    await queryClient.cancelQueries({ queryKey: ['projects', input.projectId, 'tasks'] });
    const previous = queryClient.getQueryData(['projects', input.projectId, 'tasks']);
    queryClient.setQueryData(['projects', input.projectId, 'tasks'], (old: Task[]) =>
      reorderTasksLocally(old, input),
    );
    return { previous };
  },
  onError: (_err, input, context) => {
    if (context?.previous) {
      queryClient.setQueryData(['projects', input.projectId, 'tasks'], context.previous);
    }
    toast.error('並び替えに失敗しました');
  },
  onSettled: (_data, _err, input) => {
    queryClient.invalidateQueries({ queryKey: ['projects', input.projectId, 'tasks'] });
  },
});
```

#### クライアント状態は Zustand

サーバから取得しない一時的な UI 状態のみに使う。**ドラッグ中のタスク ID、開いているモーダル、トーストキュー** など。

```ts
// ✅ Good — 用途特化のストアを複数に分ける
// apps/web/src/stores/use-modal-store.ts
import { create } from 'zustand';

interface ModalState {
  activeTaskId: string | null;
  openTaskModal: (id: string) => void;
  closeTaskModal: () => void;
}

export const useModalStore = create<ModalState>((set) => ({
  activeTaskId: null,
  openTaskModal: (id) => set({ activeTaskId: id }),
  closeTaskModal: () => set({ activeTaskId: null }),
}));
```

#### 何を Zustand に置かないか

- ✗ ログインユーザー情報 → TanStack Query（`useCurrentUser` で `GET /auth/me` を keepalive）
- ✗ タスク一覧 → TanStack Query
- ✗ フォーム入力中の値 → `react-hook-form`

### 2.6 フォーム（react-hook-form + zodResolver）

`packages/shared` の Zod スキーマを `zodResolver` でそのまま使う。`packages/shared` に無い「フォーム専用入力スキーマ」は feature 配下に置く。

```tsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { CreateTaskInputSchema, type CreateTaskInput } from '@app/shared';

export function CreateTaskForm({ projectId }: { projectId: string }) {
  const { register, handleSubmit, formState: { errors } } = useForm<CreateTaskInput>({
    resolver: zodResolver(CreateTaskInputSchema),
    defaultValues: { title: '', description: null, status: 'TODO', dueDate: null },
  });
  const createTask = useCreateTask(projectId);

  const onSubmit = handleSubmit((data) => createTask.mutate(data));

  return (
    <form onSubmit={onSubmit}>
      <input {...register('title')} />
      {errors.title && <p className="text-destructive">{errors.title.message}</p>}
      <button type="submit" disabled={createTask.isPending}>作成</button>
    </form>
  );
}
```

### 2.7 日時の扱い

すべて `date-fns` + `date-fns-tz` を使い、表示は JST、API 入出力は ISO 8601 + `+09:00`。ネイティブ `Date.prototype.toLocaleString()` で表示しない（実行環境のロケールに依存するため）。

```ts
import { format, parseISO } from 'date-fns';
import { ja } from 'date-fns/locale';

// API から受け取った ISO 文字列をそのまま表示
export function formatDateTime(iso: string): string {
  return format(parseISO(iso), 'yyyy年M月d日 HH:mm', { locale: ja });
}
```

---

## 3. バックエンド規約

### 3.1 ディレクトリ構成

`apps/api/src/` 配下は NestJS のモジュール単位（MANIFEST §8）。

```
apps/api/src/
├── modules/                        # 機能モジュール（§6 の API リソースに対応）
│   └── tasks/
│       ├── tasks.controller.ts
│       ├── tasks.service.ts
│       ├── tasks.module.ts
│       ├── dto/                    # Zod スキーマから派生する DTO 型
│       └── tasks.service.spec.ts   # ユニットテストを共置（§6 テスト規約）
├── common/                         # 横断
│   ├── guards/                     # SessionAuthGuard, CsrfGuard, RolesGuard, ProjectMemberGuard
│   ├── interceptors/               # レスポンスエンベロープ整形
│   ├── filters/                    # 例外 → §6.2 のエラーエンベロープへ
│   ├── decorators/                 # @CurrentUser, @Roles, @PublicEndpoint 等
│   └── pipes/                      # ZodValidationPipe（nestjs-zod ラッパー）
├── infra/                          # 外部 I/O
│   ├── prisma/
│   ├── mailer/
│   ├── pg-boss/
│   └── session/
├── config/                         # 環境変数バインディング（§8）
├── app.module.ts
└── main.ts
```

### 3.2 レイヤードアーキテクチャ

```
Controller   → HTTP 入出力のみ。リクエスト解釈・レスポンス整形。業務ロジック禁止。
   ↓
Service      → 業務ロジック・トランザクション境界。Prisma を直接呼ぶ。
   ↓
PrismaClient → DB アクセス。Service からのみ呼ぶ。
```

#### Controller の責務（業務ロジック禁止）

```ts
// ❌ Bad — Controller に業務ロジックがある
@Controller('tasks')
export class TasksController {
  constructor(private readonly prisma: PrismaService) {}

  @Post(':id/accept')
  async accept(@Param('id') id: string, @CurrentUser() user: User) {
    const task = await this.prisma.task.findUnique({ where: { id } });
    if (task?.requestedToId !== user.id) throw new ForbiddenException();
    return this.prisma.task.update({ where: { id }, data: { ... } });
  }
}

// ✅ Good — Controller は薄く、ロジックは Service に委譲
@Controller('tasks')
@UseGuards(SessionAuthGuard, CsrfGuard, ProjectMemberGuard)
export class TasksController {
  constructor(private readonly tasksService: TasksService) {}

  @Post(':id/accept')
  async accept(@Param('id') id: string, @CurrentUser() user: User): Promise<Task> {
    return this.tasksService.acceptTask(id, user.id);
  }
}
```

#### Service は単一トランザクションを意識する

複数テーブルにまたがる更新は **必ず `prisma.$transaction` で囲む**（MANIFEST §3.4 / §5.4 / §6.11.9）。

```ts
// ✅ Good — Outbox パターン: タスク更新 + 通知 + ジョブ投入を 1 トランザクションで
async acceptTask(taskId: string, currentUserId: string): Promise<Task> {
  return this.prisma.$transaction(async (tx) => {
    const task = await tx.task.findUnique({
      where: { id: taskId },
      include: { project: { include: { members: true } } },
    });
    if (!task) throw new TaskNotFoundException();
    if (task.requestedToId !== currentUserId) throw new NotRequesteeException();
    if (task.assignmentStatus !== 'REQUESTED') throw new InvalidStateTransitionException();

    const updated = await tx.task.update({
      where: { id: taskId },
      data: {
        assignmentStatus: 'ACCEPTED',
        assigneeId: currentUserId,
        requestedToId: null,
        requestedById: null,
      },
    });

    const notification = await tx.notification.create({
      data: {
        recipientId: task.requestedById!,
        type: 'TASK_ACCEPTED',
        channel: 'EMAIL',
        status: 'PENDING',
        taskId: task.id,
        projectId: task.projectId,
        actorId: currentUserId,
        payload: { taskTitle: task.title },
      },
    });

    await this.pgBoss.send('send-email', { notificationId: notification.id });
    return updated;
  });
}
```

### 3.3 ルーティング

#### Controller の URL は MANIFEST §6 と一致させる

```ts
// ✅ Good — /api/v1 はグローバルプレフィックスで設定（main.ts）
// app.setGlobalPrefix('api/v1');

@Controller('projects/:projectId/tasks')
export class ProjectTasksController { /* ... */ }

@Controller('tasks')
export class TasksController { /* ... */ }

@Controller('admin/dashboard')
@UseGuards(SessionAuthGuard, CsrfGuard, RolesGuard)
@Roles('ADMIN')
export class AdminDashboardController { /* ... */ }
```

#### Guard の適用順（§2.2）

`SessionAuthGuard` → `CsrfGuard` → `RolesGuard` → `ProjectMemberGuard` の順で **`@UseGuards()` に列挙する**（NestJS は左から評価する）。グローバル適用は `main.ts` でも可能。

```ts
@Controller()
@UseGuards(SessionAuthGuard, CsrfGuard) // すべてのエンドポイントに適用
export class TasksController {
  @Post(':id')
  @UseGuards(ProjectMemberGuard) // このエンドポイントだけ追加
  async update(...) { ... }
}
```

#### 状態遷移は専用エンドポイント（動詞パス）

```ts
// ❌ Bad — PATCH /tasks/:id で assigneeId を直接更新できてしまう
@Patch(':id')
async update(@Param('id') id: string, @Body() body: any) { ... }

// ✅ Good — 状態遷移はそれぞれ専用エンドポイント（§6.14）
@Post(':id/request')
async request(@Param('id') id: string, @Body() body: RequestTaskInput) { ... }

@Post(':id/accept')
async accept(@Param('id') id: string) { ... }

@Post(':id/decline')
async decline(@Param('id') id: string, @Body() body: DeclineTaskInput) { ... }
```

### 3.4 エラーハンドリング

すべての例外を `AllExceptionsFilter` で捕捉し、§6.2 のエラーエンベロープに統一する。`code` は §6.3 の `UPPER_SNAKE_CASE` で固定。

#### カスタム例外クラス

```ts
// apps/api/src/common/exceptions/app.exception.ts
export class AppException extends Error {
  constructor(
    public readonly code: string,            // §6.3 のエラーコード
    public readonly httpStatus: number,
    message: string,
    public readonly details?: Record<string, unknown>,
  ) {
    super(message);
    this.name = 'AppException';
  }
}

// 個別例外（§6.3 と 1:1 対応）
export class TaskNotFoundException extends AppException {
  constructor() {
    super('TASK_NOT_FOUND', 404, '指定されたタスクは存在しません');
  }
}

export class InvalidStateTransitionException extends AppException {
  constructor(details?: Record<string, unknown>) {
    super('INVALID_STATE_TRANSITION', 422, '状態機械上不正な遷移です', details);
  }
}

export class OptimisticLockFailedException extends AppException {
  constructor() {
    super('OPTIMISTIC_LOCK_FAILED', 409, 'リソースが他者により更新されています');
  }
}
```

#### 例外フィルタ

```ts
// apps/api/src/common/filters/all-exceptions.filter.ts
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const res = ctx.getResponse<Response>();

    let httpStatus = 500;
    let code = 'INTERNAL_ERROR';
    let message = 'サーバ内部エラー';
    let details: Record<string, unknown> | undefined;

    if (exception instanceof AppException) {
      httpStatus = exception.httpStatus;
      code = exception.code;
      message = exception.message;
      details = exception.details;
    } else if (exception instanceof ZodValidationException) {
      httpStatus = 400;
      code = 'VALIDATION_ERROR';
      message = 'リクエストが不正です';
      details = exception.getZodError().flatten().fieldErrors;
    } else if (exception instanceof Prisma.PrismaClientKnownRequestError) {
      // P2002 unique violation 等の Prisma エラーを §6.3 にマッピング
      ({ httpStatus, code, message } = mapPrismaError(exception));
    }
    // ... ロギング

    res.status(httpStatus).json({
      success: false,
      error: {
        code,
        message,
        details,
        timestamp: new Date().toISOString(),
      },
    });
  }
}
```

#### Service では型ヒントある例外を投げる

```ts
// ❌ Bad — 汎用 Error を投げると filter がマッピングできない
throw new Error('Task not found');

// ❌ Bad — NestJS 組み込み例外をそのまま投げると code が一貫しない
throw new NotFoundException('Task not found');

// ✅ Good — §6.3 のエラーコードと 1:1 対応した AppException を投げる
throw new TaskNotFoundException();
```

### 3.5 バリデーション（nestjs-zod）

`packages/shared` の Zod スキーマを `ZodValidationPipe` で適用する。`class-validator` は使わない（Zod に統一）。

```ts
// packages/shared/src/schemas/tasks.ts
import { z } from 'zod';

export const CreateTaskInputSchema = z.object({
  title: z.string().min(1).max(200),
  description: z.string().max(10000).nullable().default(null),
  status: z.enum(['TODO', 'IN_PROGRESS', 'DONE']).default('TODO'),
  dueDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).nullable().default(null),
});
export type CreateTaskInput = z.infer<typeof CreateTaskInputSchema>;
```

```ts
// apps/api/src/modules/tasks/tasks.controller.ts
import { CreateTaskInputSchema, type CreateTaskInput } from '@app/shared';
import { createZodDto } from 'nestjs-zod';

class CreateTaskDto extends createZodDto(CreateTaskInputSchema) {}

@Controller('projects/:projectId/tasks')
export class ProjectTasksController {
  @Post()
  async create(
    @Param('projectId') projectId: string,
    @Body() body: CreateTaskDto,
    @CurrentUser() user: User,
  ): Promise<Task> {
    return this.tasksService.createTask(projectId, body, user.id);
  }
}
```

`ZodValidationPipe` をグローバル適用すると全エンドポイントで自動検証される（`main.ts` で `app.useGlobalPipes(new ZodValidationPipe())`）。

### 3.6 ロギング（pino + nestjs-pino）

- すべて **構造化 JSON ログ**。`console.log` は使わない（ESLint で警告）。
- 機密情報（パスワード、トークン、`Set-Cookie`、招待トークン原文）はログに **絶対に出さない**。リクエスト本文をロギングする場合は pino の `redact` 機能でマスクする。
- ログレベル: `error`（要対応）／`warn`（想定内の異常）／`info`（重要イベント）／`debug`（詳細トレース・開発時のみ）。

```ts
// ✅ Good
this.logger.info({ taskId, userId, fromStatus, toStatus }, 'task assignment status changed');

// ❌ Bad — テンプレートリテラルで埋め込むと検索性が下がる
this.logger.info(`Task ${taskId} accepted by ${userId}`);
```

---

## 4. ORM / データベース規約

### 4.1 スキーマ命名規則

MANIFEST §5.1 に従い、DB は **snake_case**、TS は **camelCase**。`@map` / `@@map` で必ず橋渡しする。

```prisma
// ✅ Good
model ProjectMember {
  projectId String   @map("project_id")
  userId    String   @map("user_id")
  joinedAt  DateTime @default(now()) @map("joined_at") @db.Timestamptz(3)

  @@id([projectId, userId])
  @@map("project_members")
}
```

- テーブル名は **複数形 snake_case**（`users`、`project_members`）。
- カラム名は **snake_case**。
- インデックス名・制約名は Prisma の自動生成名で原則 OK。例外的に CHECK 制約等を手動 SQL で書く場合は `chk_<table>_<purpose>` / `idx_<table>_<columns>` の命名にする。

### 4.2 主キー・タイムスタンプ

```prisma
// ✅ Good — id を持つ全モデルで cuid()、TIMESTAMPTZ(3) で統一
model Task {
  id        String   @id @default(cuid())
  // ...
  createdAt DateTime @default(now()) @map("created_at") @db.Timestamptz(3)
  updatedAt DateTime @updatedAt        @map("updated_at") @db.Timestamptz(3)
}
```

- `updatedAt` は楽観ロック用（§5.1）。`PATCH` 系エンドポイントの Body で `updatedAt` を必須にし、Service 層で `where: { id, updatedAt }` で更新。
- `dueDate` のような **日付のみ** は `DateTime @db.Date`。

### 4.3 リレーション

- 外部キーカラムは `<relation>Id` の camelCase（`assigneeId`、`projectId`）。
- `onDelete` の方針は MANIFEST §5.4 の表に従う。新規モデル追加時もこの方針に倣う:
  - User 参照: 原則 `Restrict`（物理削除を防ぐ）
  - Task 参照: `Cascade`（タスク物理削除に追随）
  - Project 参照: `Cascade`（または `SetNull` で履歴を残すか業務判断）

### 4.4 PrismaService（クライアント設定）

```ts
// apps/api/src/infra/prisma/prisma.service.ts
import { Injectable, OnModuleDestroy, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  constructor() {
    super({
      log: [
        { level: 'warn', emit: 'stdout' },
        { level: 'error', emit: 'stdout' },
        // 開発時のみ query ログを stdout、本番は無効
      ],
    });
  }

  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

- **Controller から直接 `PrismaService` を呼ばない**。必ず Service 経由（§3.2）。
- `PrismaService` は `PrismaModule` でグローバルに提供し、各 Feature モジュールから注入する。

### 4.5 トランザクション

#### 複数テーブルにまたがる更新は必ず `$transaction`

```ts
// ✅ Good — Interactive Transaction
async removeMember(projectId: string, userId: string): Promise<void> {
  await this.prisma.$transaction(async (tx) => {
    await tx.task.updateMany({
      where: { projectId, assigneeId: userId, assignmentStatus: 'ACCEPTED' },
      data: { assignmentStatus: 'UNASSIGNED', assigneeId: null },
    });
    await tx.task.updateMany({
      where: { projectId, requestedToId: userId, assignmentStatus: 'REQUESTED' },
      data: { assignmentStatus: 'UNASSIGNED', requestedToId: null, requestedById: null },
    });
    await tx.projectMember.delete({ where: { projectId_userId: { projectId, userId } } });
  });
}
```

#### 楽観ロック

```ts
// ✅ Good — where に updatedAt を含め、件数 0 なら競合
async updateTask(id: string, input: UpdateTaskInput, expectedUpdatedAt: Date): Promise<Task> {
  const updated = await this.prisma.task.updateMany({
    where: { id, updatedAt: expectedUpdatedAt },
    data: { ...input },
  });
  if (updated.count === 0) {
    throw new OptimisticLockFailedException();
  }
  return this.prisma.task.findUniqueOrThrow({ where: { id } });
}
```

R-05 の位置変更（`POST /tasks/:id/position`）のみ楽観ロックを **適用しない**（MANIFEST §5.1）。

#### Raw SQL は最後の手段

`$queryRaw` / `$executeRaw` を使う場合は **必ずタグ付きテンプレート版**（プレースホルダ自動エスケープ）を使う。`$queryRawUnsafe` は禁止。

```ts
// ❌ Bad — SQL インジェクションの温床
await this.prisma.$queryRawUnsafe(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Good — タグ付きテンプレート
await this.prisma.$queryRaw`SELECT * FROM users WHERE email = ${email}`;
```

### 4.6 マイグレーション

- 命名: `pnpm prisma migrate dev --name <snake_case_description>` で `<timestamp>_<description>/` が作成される。description は内容が分かるものに（`add_position_index_to_tasks` など）。
- `prisma migrate reset` は **開発環境のみ**。本番ではマイグレーションを順次適用するのみ。
- CHECK 制約や複雑なインデックス（部分インデックス等）は migration ファイルに **手動 SQL** を追記する（MANIFEST §5.5.2 の例を参照）。

---

## 5. API 設計規約

### 5.1 RESTful 命名

| 種別 | 規約 | 例 |
|---|---|---|
| ベースパス | `/api/v1/` | `/api/v1/tasks` |
| リソースパス | 複数形 kebab-case | `/projects`、`/project-members` |
| 識別子パラメータ | `:resourceId` または `:id` | `/projects/:projectId/tasks/:id` |
| 状態遷移 | 動詞パス（POST） | `/tasks/:id/accept`、`/projects/:id/archive` |
| 一括取得 | 親リソース配下にネスト | `/projects/:projectId/tasks` |
| クエリパラメータ | camelCase | `?includeArchived=true&pageSize=20` |

### 5.2 レスポンスエンベロープ

すべてのレスポンスを §6.2 のエンベロープに包む。**Interceptor で自動付与** し、Controller は素のオブジェクトを返す。

```ts
// ✅ Good — Controller は素の値を return、Interceptor が success/data に包む
@Get(':id')
async findOne(@Param('id') id: string): Promise<Task> {
  return this.tasksService.findOne(id);
}
```

```ts
// apps/api/src/common/interceptors/response-envelope.interceptor.ts
@Injectable()
export class ResponseEnvelopeInterceptor implements NestInterceptor {
  intercept(_ctx: ExecutionContext, next: CallHandler): Observable<unknown> {
    return next.handle().pipe(map((data) => ({ success: true, data })));
  }
}
```

エラー側は §3.4 の `AllExceptionsFilter` がエンベロープを付与する。

### 5.3 HTTP ステータスコード

| ステータス | 使うべき場合 |
|---|---|
| `200 OK` | 取得・更新成功（レスポンス本文あり） |
| `201 Created` | リソース新規作成成功 |
| `204 No Content` | 削除・パスワード変更等、本文不要な成功 |
| `400 Bad Request` | リクエストフォーマット不正（Zod 検証失敗） |
| `401 Unauthorized` | 未認証・認証失敗 |
| `403 Forbidden` | 認証済だが認可エラー |
| `404 Not Found` | リソース不存在 |
| `409 Conflict` | 楽観ロック失敗、重複、既存リソースとの競合 |
| `410 Gone` | 招待トークン期限切れ等、明示的に「もう存在しない」 |
| `422 Unprocessable Entity` | 業務ルール違反（状態機械、自己依頼禁止等） |
| `429 Too Many Requests` | レートリミット超過 |
| `500 Internal Server Error` | サーバ側の予期せぬ例外 |

「400 と 422 を曖昧に使う」のは禁止。**400 = フォーマット**、**422 = 業務ルール**で線引きする（§6.3 を参照）。

### 5.4 ページネーション

`GET` 一覧系は `page` / `pageSize` を Query で受け、レスポンスを `{ items, meta }` に統一する（§6.2）。`pageSize` は **最大 100** とする。

ただし MANIFEST で全件返却と定められたカンバンボード（§6.11.1）・プロジェクト一覧（§6.9.1）・メンバー一覧（§6.10.1）は **ページネーションしない**。これらは `{ items, meta }` ではなく配列をそのまま返す。

### 5.5 楽観ロック

更新系の Body には `updatedAt` を必須項目として含める。R-05 の位置変更のみ例外（§5.1）。

```ts
// ✅ Good
const UpdateTaskInputSchema = z.object({
  title: z.string().min(1).max(200).optional(),
  description: z.string().max(10000).nullable().optional(),
  status: z.enum(['TODO', 'IN_PROGRESS', 'DONE']).optional(),
  dueDate: z.string().regex(/^\d{4}-\d{2}-\d{2}$/).nullable().optional(),
  updatedAt: z.string().datetime({ offset: true }), // 必須
});
```

### 5.6 OpenAPI / Swagger

- `@nestjs/swagger` で `/api/docs` を公開。CI で `openapi.json` を出力し `packages/shared` の Zod スキーマと整合性検証する（MANIFEST §6.16 / §9.6）。
- Controller 各エンドポイントには `@ApiOperation`、`@ApiResponse` を付与。Zod スキーマから DTO を `createZodDto` で生成すれば `@ApiBody` は自動付与される。

---

## 6. テスト規約

### 6.1 何をテストするか

テスト実行は **Turborepo** のタスクパイプライン経由（`pnpm test`）。CI ではキャッシュを活かして変更ワークスペースのみ実行する。

| レイヤ | ツール | 主目的 |
|---|---|---|
| Service / Util（BE） | Jest | 業務ロジック、状態機械、エッジケース |
| Controller / E2E（BE） | Jest + Supertest | 認可、Guard 連携、エンドツーエンドのリクエスト/レスポンス |
| Component / Hook（FE） | Vitest + React Testing Library | UI の振る舞い、Custom Hook の戻り値 |
| Browser E2E（FE） | Playwright | クリティカルユーザーフロー（ログイン → カンバン操作 → 承諾） |

> 「カバレッジ X%」を絶対基準にはしない。**業務ロジックを持つ Service と状態機械は必ずテストする**ことを優先する。

### 6.2 ファイル構成

#### バックエンド

- ユニットテストは対象ファイルと **共置**: `tasks.service.ts` → `tasks.service.spec.ts`
- E2E テストは `apps/api/test/` 配下に集約: `tasks.e2e-spec.ts`

```
apps/api/
├── src/modules/tasks/
│   ├── tasks.service.ts
│   ├── tasks.service.spec.ts        # ユニット
│   └── tasks.controller.ts
└── test/
    └── tasks.e2e-spec.ts            # E2E
```

#### フロントエンド

- コンポーネント・hook と **共置**: `TaskCard.tsx` → `TaskCard.test.tsx`、`use-tasks.ts` → `use-tasks.test.ts`
- Playwright E2E は `apps/web/e2e/` 配下

### 6.3 命名規則

- `describe` には **対象のクラス・関数・コンポーネント名** を書く。
- `it` には **「何をすると / どうなる」** を書く（`should` 接頭辞は省略可）。

```ts
// ✅ Good
describe('TasksService.acceptTask', () => {
  it('REQUESTED 状態のタスクを ACCEPTED に遷移させる', async () => { /* ... */ });

  it('被依頼者本人でなければ NOT_REQUESTEE をスローする', async () => { /* ... */ });

  it('REQUESTED 以外の状態では INVALID_STATE_TRANSITION をスローする', async () => { /* ... */ });
});

// ❌ Bad — 曖昧
describe('TasksService', () => {
  it('works', () => { /* ... */ });
  it('test1', () => { /* ... */ });
});
```

### 6.4 テストパターン

#### AAA パターン（Arrange / Act / Assert）

```ts
it('REQUESTED 状態のタスクを ACCEPTED に遷移させる', async () => {
  // Arrange
  const requester = await createUser({ role: 'MEMBER' });
  const requestee = await createUser({ role: 'MEMBER' });
  const project = await createProject({ members: [requester, requestee] });
  const task = await createTask({
    projectId: project.id,
    assignmentStatus: 'REQUESTED',
    requestedById: requester.id,
    requestedToId: requestee.id,
  });

  // Act
  const updated = await service.acceptTask(task.id, requestee.id);

  // Assert
  expect(updated.assignmentStatus).toBe('ACCEPTED');
  expect(updated.assigneeId).toBe(requestee.id);
  expect(updated.requestedToId).toBeNull();
  expect(updated.requestedById).toBeNull();
});
```

#### テストデータはファクトリで作る

固定 ID やインラインオブジェクトを大量に書かない。`test/factories/` に **ファクトリ関数** を用意する。

```ts
// apps/api/test/factories/task.factory.ts
import cuid from 'cuid'; // 本番の Prisma @default(cuid()) と同じ cuid v1 を使用（MANIFEST §5.1）

export const buildTask = (overrides: Partial<Task> = {}): Task => ({
  id: cuid(),
  projectId: cuid(),
  title: 'Sample Task',
  description: null,
  status: 'TODO',
  assignmentStatus: 'UNASSIGNED',
  assigneeId: null,
  requestedToId: null,
  requestedById: null,
  requestMessage: null,
  dueDate: null,
  position: 0,
  createdAt: new Date(),
  updatedAt: new Date(),
  ...overrides,
});
```

#### モックは最小限に

- Service のユニットテストでは Prisma を実 DB（テスト用の独立 DB）に対して呼ぶ統合テスト寄りを推奨する。Mock を多用すると「実装をテストしているだけ」になりやすい。
- pg-boss・メール送信などの外部副作用は Mock 化する。

```ts
// ✅ Good — pg-boss は Mock、Prisma は実 DB
beforeAll(async () => {
  app = await createTestingApp({
    pgBoss: { send: jest.fn() }, // Mock
    // PrismaService はテスト用 DB に接続
  });
});
```

#### コンポーネントテスト（FE）

```tsx
// apps/web/src/features/tasks/components/TaskCard.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { TaskCard } from './TaskCard';
import { buildTask } from '@/test/factories';

describe('TaskCard', () => {
  it('クリックすると onClick がタスク ID 付きで呼ばれる', async () => {
    const onClick = vi.fn();
    const task = buildTask({ id: 'task_1', title: 'Buy milk' });

    render(<TaskCard task={task} onClick={onClick} />);
    await userEvent.click(screen.getByRole('button', { name: /buy milk/i }));

    expect(onClick).toHaveBeenCalledWith('task_1');
  });
});
```

- セレクタは `getByRole` > `getByLabelText` > `getByText` の順で優先（アクセシビリティと一致するため）。`data-testid` は最後の手段。
- ユーザー操作は `fireEvent` ではなく `userEvent` を使う。

#### E2E テスト（Playwright）

- クリティカルパスのみ。**全機能を E2E でカバーしない**（時間がかかるため、ユニット / 統合テストでカバー）。
- 例: ログイン → プロジェクト一覧 → カンバン表示 → タスク作成 → 依頼 → 承諾 → 通知メール受信（Mailpit で確認）。

---

## 7. Git 規約

### 7.1 ブランチ命名

`<type>/<issue-id>-<short-description>` の形式。`type` は Conventional Commits のタイプと揃える。

| プレフィックス | 用途 | 例 |
|---|---|---|
| `feature/` | 新機能 | `feature/TB-101-task-accept-flow` |
| `fix/` | バグ修正 | `fix/TB-205-position-race-condition` |
| `chore/` | ビルド・依存関係・雑務 | `chore/TB-310-upgrade-nestjs-to-10.4` |
| `docs/` | ドキュメントのみ | `docs/TB-401-update-manifest-tz` |
| `refactor/` | 振る舞いを変えない内部改善 | `refactor/TB-220-extract-task-state-machine` |
| `test/` | テスト追加・修正のみ | `test/TB-305-tasks-service-coverage` |
| `hotfix/` | 本番障害対応 | `hotfix/TB-999-session-cookie-domain` |

- `main`（または `develop`）への直接プッシュは禁止。必ず PR 経由。
- PR マージ後はブランチを削除する（リモート・ローカルとも）。

### 7.2 コミットメッセージ（Conventional Commits）

```
<type>(<scope>): <subject>

<body>

<footer>
```

| type | 用途 |
|---|---|
| `feat` | 新機能 |
| `fix` | バグ修正 |
| `docs` | ドキュメント |
| `style` | コードフォーマット（振る舞い変更なし） |
| `refactor` | 振る舞いを変えない内部改善 |
| `perf` | パフォーマンス改善 |
| `test` | テスト追加・修正 |
| `build` | ビルド・依存関係 |
| `ci` | CI 設定 |
| `chore` | その他雑務 |
| `revert` | 過去コミットの取り消し |

- **subject**: 50 文字以内、命令形（"add" / "fix" / "remove"）、末尾ピリオドなし。日本語可。
- **scope**: feature 名 or モジュール名（`tasks`、`auth`、`prisma`、`web` 等）。
- **body**: 必要なら 72 文字で折り返し、**なぜ** を中心に。
- **footer**: 関連 Issue や Breaking Change を明示。

```
✅ Good:
feat(tasks): タスク承諾フローを追加 (R-08)

POST /tasks/:id/accept で被依頼者本人のみ承諾可能。
Outbox パターンで Notification 作成 + pg-boss ジョブ投入を
単一トランザクションで実行する。

Closes #101

❌ Bad:
update
WIP
[fix] バグ修正しました。
```

### 7.3 PR ルール

- タイトルは Conventional Commits 形式（Squash マージ時のコミットメッセージになる）。
- 1 PR = 1 トピック。「タスク承諾フロー追加」と「依存関係アップグレード」を同じ PR に混ぜない。
- レビュアー 1 名以上の Approve が必須。
- CI（Lint / Typecheck / Test / OpenAPI 整合）がすべて Green になってからマージ。
- マージ方式は **Squash and merge** を既定（コミット履歴を線形に保つ）。

---

## 8. 環境変数管理

### 8.1 ファイル運用

- `.env.example` を **必ずリポジトリに含める**（実値はダミー）。実値の `.env` は `.gitignore` で除外。
- `.env` を更新したら `.env.example` も同時に更新する（変数名のみ追記）。
- 本番では **Secrets Manager / Parameter Store 等** から注入する。リポジトリにも CI にも実値を置かない。

`.gitignore`:
```
.env
.env.*
!.env.example
```

### 8.2 起動時に Zod で検証

NestJS の `ConfigModule` で `.env` を読み込み、Zod スキーマで検証する。**1 つでも欠けたら起動失敗** で運用上のミスを早期検知。

```ts
// apps/api/src/config/env.schema.ts
import { z } from 'zod';

export const EnvSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  TZ: z.literal('Asia/Tokyo'),
  PORT: z.coerce.number().int().positive().default(3000),
  APP_BASE_URL: z.string().url(),
  DATABASE_URL: z.string().url(),
  SESSION_SECRET: z.string().min(32),
  SESSION_NAME: z.string().default('tb_session'),
  SESSION_MAX_AGE_HOURS: z.coerce.number().int().positive().default(24),
  CSRF_SECRET: z.string().min(32),
  COOKIE_DOMAIN: z.string(),
  COOKIE_SECURE: z.coerce.boolean(),
  BCRYPT_COST: z.coerce.number().int().min(10).max(15).default(12),
  INVITATION_TTL_DAYS: z.coerce.number().int().positive().default(7),
  MAIL_TRANSPORT: z.enum(['smtp', 'ses']),
  MAIL_HOST: z.string().optional(),
  MAIL_PORT: z.coerce.number().int().optional(),
  MAIL_USER: z.string().optional(),
  MAIL_PASS: z.string().optional(),
  MAIL_FROM: z.string().email(),
  SES_REGION: z.string().optional(),
  PGBOSS_SCHEMA: z.string().default('pgboss'),
  LOG_LEVEL: z.enum(['fatal', 'error', 'warn', 'info', 'debug', 'trace']).default('info'),
});

export type Env = z.infer<typeof EnvSchema>;
```

```ts
// apps/api/src/config/config.module.ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      validate: (raw) => EnvSchema.parse(raw),
    }),
  ],
})
export class AppConfigModule {}
```

### 8.3 アクセスは型安全に

```ts
// ❌ Bad — process.env を直接参照
const port = process.env.PORT;

// ✅ Good — ConfigService 経由で型付き取得
@Injectable()
export class TasksService {
  constructor(private readonly config: ConfigService<Env, true>) {}

  someMethod() {
    const ttlDays = this.config.get('INVITATION_TTL_DAYS', { infer: true });
  }
}
```

### 8.4 フロントエンドの環境変数

- Vite では `VITE_` 接頭辞のみ公開される。**機密値は絶対にフロント側に置かない**（ビルド成果物に埋め込まれる）。
- 同一オリジン運用のため、本番では `VITE_API_BASE_URL` を **設定しない**（相対パスで `/api/v1/...` を叩く）。開発時のみ `http://localhost:3000` を設定。

---

## 9. セキュリティガイドライン

### 9.1 認証・セッション

- Cookie 属性: **`HttpOnly`、`Secure`、`SameSite=Lax`**（MANIFEST §6.17）。
- セッション署名鍵 `SESSION_SECRET` は **32 文字以上のランダム値**。Zod でも `.min(32)` を強制（§8.2）。
- ログアウト時はサーバ側でセッションを破棄し、`Set-Cookie` で失効させる。クライアント側だけで Cookie を消さない。

### 9.2 パスワード

- bcrypt コスト係数 **12**（MANIFEST §3.1 / §7.2）。
- 平文パスワードは **DB・ログ・エラーレスポンス・例外メッセージのいずれにも残さない**。
- パスワード長: 最小 12 文字、最大 128 文字。複雑性ルールは強制しない（NIST 800-63B）。

```ts
// ❌ Bad — エラーメッセージにパスワードを含めるな
throw new Error(`Login failed for user with password ${password}`);

// ❌ Bad — ログ
logger.info({ email, password }, 'login attempt');

// ✅ Good
logger.info({ email }, 'login attempt');
```

### 9.3 CSRF

- 状態変更系（POST / PATCH / DELETE）は **`X-CSRF-Token` ヘッダ** と `csrf_token` Cookie の一致を `CsrfGuard` で検証（§2.2）。
- FE では TanStack Query の mutation で **自動的にヘッダを付与** する共通ラッパを `lib/api-client.ts` に置く。

```ts
// apps/web/src/lib/api-client.ts
import axios from 'axios';

export const apiClient = axios.create({
  baseURL: '/api/v1',
  withCredentials: true,
});

apiClient.interceptors.request.use((config) => {
  if (['post', 'patch', 'delete', 'put'].includes(config.method ?? '')) {
    const csrfToken = getCookie('csrf_token');
    if (csrfToken) config.headers['X-CSRF-Token'] = csrfToken;
  }
  return config;
});
```

### 9.4 XSS

- React の自動エスケープを信頼する。**`dangerouslySetInnerHTML` は原則禁止**。やむを得ず使う場合は DOMPurify でサニタイズし、PR 説明に根拠を明示。
- Markdown レンダリングは初期スコープ外（MANIFEST §3.5）。

```tsx
// ❌ Bad
<div dangerouslySetInnerHTML={{ __html: comment.body }} />

// ✅ Good — プレーンテキストとして表示
<div className="whitespace-pre-wrap">{comment.body}</div>
```

### 9.5 SQL インジェクション

- Prisma の生成クエリは安全。`$queryRawUnsafe` / `$executeRawUnsafe` は **禁止**（§4.5）。タグ付きテンプレート版を使う。

### 9.6 認可境界

- **すべての** プロジェクトスコープエンドポイントに `ProjectMemberGuard` を付け、さらに Service 層でも `project: { members: { some: { userId } } }` を付与（多層防御、MANIFEST §2.2）。Guard 一つに依存しない。
- ADMIN ロールでも非メンバーであれば操作不可（§2.3）。

```ts
// ❌ Bad — Service 層で project membership を確認していない
async findTask(id: string): Promise<Task> {
  return this.prisma.task.findUniqueOrThrow({ where: { id } });
}

// ✅ Good — Service 層クエリにも所属チェックを必ず含める
async findTask(id: string, currentUserId: string): Promise<Task> {
  const task = await this.prisma.task.findFirst({
    where: {
      id,
      project: { members: { some: { userId: currentUserId } } },
    },
  });
  if (!task) throw new TaskNotFoundException();
  return task;
}
```

### 9.7 招待トークン

- 原文は **生成時とメール送信時のみ** 存在。DB には **SHA-256 ハッシュのみ**（MANIFEST §3.1 / §5.3.2）。
- ログに原文トークンを出さない（招待 URL もログ出力しない）。

### 9.8 レートリミット

- 認証系エンドポイント（ログイン、招待受諾、パスワード変更、ユーザー招待、招待再送）に `@nestjs/throttler` を適用（MANIFEST §6.17）。
- レートリミット超過時は `429 RATE_LIMIT_EXCEEDED`。

### 9.9 ログ・秘匿情報

- 機密項目は `pino` の `redact` でマスク:

```ts
// apps/api/src/main.ts
const logger = pino({
  redact: {
    paths: [
      'req.headers.cookie',
      'req.headers.authorization',
      'req.body.password',
      'req.body.currentPassword',
      'req.body.newPassword',
      'req.body.token',
      '*.password',
      '*.tokenHash',
    ],
    censor: '[REDACTED]',
  },
});
```

### 9.10 helmet / CORS

- 本番は同一オリジン運用のため CORS は **無効**。開発時のみ `http://localhost:5173` を許可（MANIFEST §6.17）。
- `helmet()` をデフォルト設定で適用し、CSP は段階的に厳格化する。

---

## 10. コードレビューチェックリスト

PR レビュー時に **レビュアー** がチェックする観点。**著者は PR 説明文に「該当項目を満たした」旨を簡潔に書く**。

### 10.1 設計・整合性

- [ ] 変更内容が `MANIFEST.md` の関連セクションと矛盾していないか
- [ ] API 仕様を変更した場合、`packages/shared` の Zod スキーマと OpenAPI（`/api/docs`）が同時に更新されているか
- [ ] エラーコードが `MANIFEST.md §6.3` の一覧に存在するか、または同表に追加されているか
- [ ] DB スキーマを変更した場合、`MANIFEST.md §5` と Prisma マイグレーションの両方が更新されているか

### 10.2 認可・セキュリティ

- [ ] 新規エンドポイントに適切な Guard（`SessionAuthGuard` / `CsrfGuard` / `RolesGuard` / `ProjectMemberGuard`）が付与されているか
- [ ] プロジェクトスコープのリソースに対し、Service 層クエリでも所属チェックが付与されているか（多層防御）
- [ ] パスワード・トークン・Cookie がログ・エラーメッセージに漏れていないか
- [ ] `dangerouslySetInnerHTML`・`$queryRawUnsafe` を使っていないか（やむを得ない場合は PR 説明に根拠）

### 10.3 業務ロジック

- [ ] タスクの担当者割り当てが状態機械（`UNASSIGNED → REQUESTED → ACCEPTED`）を経由しているか
- [ ] 複数テーブル更新が `prisma.$transaction` で囲まれているか
- [ ] 楽観ロックが必要な更新で `updatedAt` を Body から受け取り、where に含めているか
- [ ] 削除戦略が `MANIFEST.md §5.4` の方針と整合しているか（論理 / 物理 / ハード / CASCADE）

### 10.4 入力検証

- [ ] すべての入力が `packages/shared` の Zod スキーマで検証されているか
- [ ] `any` / `as` キャストを使っていないか（使う場合は理由をコメント）
- [ ] 文字列長・配列要素数の上限・下限が定義されているか

### 10.5 テスト

- [ ] Service の新規メソッドにユニットテストがあるか
- [ ] 状態機械の遷移ごとに「正常」「権限なし」「不正な現状態」の各テストがあるか
- [ ] FE 側で UI の振る舞いに対応するコンポーネントテストがあるか
- [ ] スナップショットテストに頼りすぎていないか（振る舞いを検証する形になっているか）

### 10.6 コード品質

- [ ] ESLint / Prettier / typecheck が CI で通っているか
- [ ] `console.log` が残っていないか
- [ ] TODO / FIXME に GitHub Issue 番号が付いているか
- [ ] 命名規則（§1.4）に準拠しているか
- [ ] Controller に業務ロジックが漏れていないか（§3.2）
- [ ] feature をまたぐ内部実装の import がないか（§2.1）

### 10.7 環境変数・運用

- [ ] 新規環境変数がある場合、`.env.example` と `EnvSchema`（§8.2）の両方が更新されているか
- [ ] マイグレーションが破壊的変更（カラム削除等）を含む場合、本番適用順序が PR 説明に書かれているか
- [ ] 機密値が `.env` や CI 設定にハードコードされていないか

### 10.8 ドキュメント

- [ ] 公開 API（Controller / Service の public method / 公開 hook）に JSDoc があるか
- [ ] PR 説明に **何を**・**なぜ**・**どう検証したか** が書かれているか
- [ ] スクリーンショット・動画が UI 変更時に添付されているか

---

> 本書はチームで運用するうちに改善されることを前提とする。改善提案は PR にて行い、`MANIFEST.md` と整合性を保ったうえで採用する。判断に迷う規約があれば「なぜこのルールが存在するのか」をテックリードと議論し、明文化した根拠を残すこと。
