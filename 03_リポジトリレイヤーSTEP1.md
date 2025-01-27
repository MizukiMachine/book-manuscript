## 基本的な型定義

Step2までで、TDDの考え方とプロジェクトのセットアップについて学びました。
今回は、実際にTodoリストAPIを実装していきながら、
リポジトリレイヤーのテスト駆動開発を行っていきます。

まず、Todoに関連する型定義から始めます。
これらの型定義は、アプリケーション全体で使用する共通の型となります。

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

説明
- `Todo`インターフェースは、システムで管理するTodoの完全な形を定義します
- `CreateTodoDTO`は新規作成時に必要な項目のみを含みます
- `UpdateTodoDTO`は全てのフィールドをオプショナルにすることで、部分的な更新を可能にします

## 4. TodoRepositoryの初期テスト

最初のテストファイルを作成してみます

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
        test('creates a new todo with required fields', async () => {
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

この時点でテストを実行してみます

```bash
npm test
```

実行結果
```
FAIL  src/repositories/todo.repository.test.ts
  ● Test suite failed to run
    Cannot find module './todo.repository' from 'src/repositories/todo.repository.test.ts'

    > 1 | import { TodoRepository } from './todo.repository';
```

これは当然の結果です。
まだ`todo.repository.ts`ファイルを作成していないためエラーになります。
では、このテストを通すための最小限の実装を行ってみましょう

```typescript
// src/repositories/todo.repository.ts
import { Todo, CreateTodoDTO } from '../types';
import { v4 as uuidv4 } from 'uuid';  // uuidパッケージが必要です

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

実装のポイント
- `Map`を使用してTodoをメモリ内で管理します
- `uuidv4()`で一意のIDを生成します
- 非同期メソッドとして定義し、将来的なデータベース統合に備えます

まず、必要なパッケージをインストールします

```bash
npm install uuid
npm install --save-dev @types/uuid
```

テスト結果
```
PASS  src/repositories/todo.repository.test.ts
  TodoRepository
    create
      ✓ creates a new todo with required fields (3ms)
```

テストが通りました！これで最初の機能が実装できました。
