# 第3章 リポジトリレイヤーの基礎実装

## 3.1 基本的な型定義

まず、Todoアプリケーションで使用する基本的な型を定義します。これらの型定義は、アプリケーション全体で共通して使用します。

```typescript
// src/types.ts
export interface Todo {
    id: string;          // Todoの一意識別子
    title: string;       // Todoのタイトル
    description?: string;// 詳細説明（オプショナル）
    completed: boolean;  // 完了状態
    createdAt: Date;    // 作成日時
    updatedAt: Date;    // 更新日時
}

// 新規Todo作成時に使用するDTO（Data Transfer Object）
export interface CreateTodoDTO {
    title: string;       // 必須項目
    description?: string;// オプショナル
}

// Todo更新時に使用するDTO
export interface UpdateTodoDTO {
    title?: string;      // 全てオプショナル
    description?: string;// 更新したいフィールドのみ
    completed?: boolean; // 指定可能
}
```

**型定義の解説：**
- `Todo`: データベースに保存される完全なTodoの形
- `CreateTodoDTO`: APIがクライアントから受け取る作成用データの形
- `UpdateTodoDTO`: 更新時に必要なフィールドのみを含む形

## 3.2 TodoRepositoryの初期実装

まず、TodoRepositoryのテストファイルを作成します。

```typescript
// src/repositories/todo.repository.test.ts
import { TodoRepository } from './todo.repository';
import { CreateTodoDTO, Todo } from '../types';

describe('TodoRepository', () => {
    let repository: TodoRepository;

    beforeEach(() => {
        // 各テストケース実行前にリポジトリを初期化
        repository = new TodoRepository();
    });

    describe('create', () => {
        it('creates a new todo with required fields', async () => {
            const dto: CreateTodoDTO = {
                title: 'Test Todo'
            };

            const todo = await repository.create(dto);

            // 新しいTodoが正しく作成されていることを確認
            expect(todo.id).toBeDefined();
            expect(todo.title).toBe(dto.title);
            expect(todo.completed).toBe(false);
            expect(todo.createdAt).toBeInstanceOf(Date);
            expect(todo.updatedAt).toBeInstanceOf(Date);
        });
    });
});
```

**テストコードの解説：**
- `beforeEach`: 各テストケース実行前にリポジトリを初期化し、テスト間の独立性を確保
- 非同期処理の`async/await`を使用：実際のデータベース操作を想定
- `expect`文で各プロパティを個別に検証：より詳細なテスト

テストを実行すると、以下のエラーが発生します。

```bash
FAIL  src/repositories/todo.repository.test.ts
  ● Test suite failed to run
    Cannot find module './todo.repository' from 'src/repositories/todo.repository.test.ts'
```
これは当然の結果です。
まだ`todo.repository.ts`ファイルを作成していないためエラーになります。  
では、このエラーを解消するため、TodoRepositoryの実装を行います。

```typescript
// src/repositories/todo.repository.ts
import { Todo, CreateTodoDTO } from '../types';
import { v4 as uuidv4 } from 'uuid';  // IDの生成に使用

export class TodoRepository {
    // TodoをIDで管理するためのMap
    private todos: Map<string, Todo> = new Map();

    async create(dto: CreateTodoDTO): Promise<Todo> {
        const todo: Todo = {
            id: uuidv4(),          // ユニークなIDを生成
            title: dto.title,      // DTOからタイトルを取得
            description: dto.description,
            completed: false,      // 新規作成時は未完了
            createdAt: new Date(), // 現在の日時
            updatedAt: new Date()  // 現在の日時
        };

        this.todos.set(todo.id, todo);
        return todo;
    }
}
```

**実装の解説：**
- `Map`を使用してTodoをメモリ内で管理
  - キー：TodoのID
  - 値：Todoオブジェクト
- `uuidv4()`で一意のIDを生成
- メソッドは非同期（`async`）として定義
  - 将来的なデータベース統合を見据えた設計

必要なパッケージをインストール：

```bash
npm install uuid
npm install --save-dev @types/uuid
```

テストを再実行：

```bash
npm test

PASS  src/repositories/todo.repository.test.ts
  TodoRepository
    create
      ✓ creates a new todo with required fields (3ms)
```

## 3.3 ポイント解説

1. **インメモリデータストア**
   - `Map`を使用することで、キーと値の関係を明確に管理
   - 開発初期段階では、データベースの複雑さを避けつつ機能実装に集中可能

2. **非同期処理**
   ```typescript
   async create(dto: CreateTodoDTO): Promise<Todo>
   ```
   - データベース統合を見据えた設計
   - `Promise`を返すことで、将来的な非同期操作への対応を容易に

3. **型安全性**
   ```typescript
   private todos: Map<string, Todo> = new Map();
   ```
   - TypeScriptの型システムを活用
   - コンパイル時のエラー検出を可能に

4. **DTOパターン**
   ```typescript
   async create(dto: CreateTodoDTO): Promise<Todo>
   ```
   - 入力データと永続化データを明確に分離
   - APIの入力とデータモデルの間の変換を明示的に実装

この章では、TDDの基本的なサイクルに従いながら、TodoRepositoryの基本機能を実装しました。  
次章では、より高度な機能とバリデーションの実装に進みます。
