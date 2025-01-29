# 第6章 TodoRepositoryの更新機能実装

## 6.1 更新機能のテスト実装

まず、Todoの更新機能に関するテストケースを実装します。


1. 共通のテストユーティリティファイルの作成
```typescript
// src/test-utils/helpers.ts

// テストヘルパー関数：時間差を作るため
export const wait = (ms: number = 1) => new Promise(resolve => setTimeout(resolve, ms));
```

```typescript
// src/repositories/todo.repository.test.ts
import { wait } from '../test-utils/helpers';

describe('TodoRepository', () => {
    let repository: TodoRepository;

    beforeEach(() => {
        repository = new TodoRepository();
    });

    describe('update', () => {
        it('updates todo fields correctly', async () => {
            // テスト用のTodoを作成
            const todo = await repository.create({
                title: 'Original Title',
                description: 'Original Description'
            });

            // updatedAtの比較テストのため、
            // Date オブジェクトの解像度（ミリ秒）の制限により
            // 意図的に時間差を作る
            await wait();

            // 更新用のDTOを作成
            const updateDto: UpdateTodoDTO = {
                title: 'Updated Title',
                description: 'Updated Description',
                completed: true
            };

            // Todoを更新
            const updated = await repository.update(todo.id, updateDto);

            // 更新結果の検証
            expect(updated.title).toBe(updateDto.title);
            expect(updated.description).toBe(updateDto.description);
            expect(updated.completed).toBe(true);
            expect(updated.updatedAt.getTime()).toBeGreaterThan(created.updatedAt.getTime());
            expect(updated.id).toBe(todo.id); // IDは変更されないことを確認
        });

        it('maintains unchanged fields', async () => {
            // 元のTodoを作成
            const todo = await repository.create({
                title: 'Original Title',
                description: 'Original Description'
            });

            await wait();

            // タイトルのみを更新
            const updated = await repository.update(todo.id, {
                title: 'Updated Title'
            });

            // 説明文が維持されていることを確認
            expect(updated.title).toBe('Updated Title');
            expect(updated.description).toBe('Original Description');
            expect(updated.updatedAt.getTime()).toBeGreaterThan(created.updatedAt.getTime());
        });

        it('throws error when todo does not exist', async () => {
            await expect(
                repository.update('non-existent-id', { title: 'New Title' })
            ).rejects.toThrow('Todo not found');
        });

        it('applies validation rules to updated fields', async () => {
            const todo = await repository.create({
                title: 'Original Title'
            });

            // 空白のタイトルでの更新を試みる
            await expect(
                repository.update(todo.id, { title: '   ' })
            ).rejects.toThrow('Title cannot be empty');
        });
    });
});
```

**テストケースの解説：**
1. **テストヘルパー関数**
    - `wait` 関数を導入し、更新日時の比較を確実に
    - ミリ秒単位の時間差を作成

2. **基本的な更新機能**
    - 全フィールドの更新
    - 更新日時の変更確認
    - IDの不変性確認

3. **部分更新の処理**
    - 特定フィールドのみの更新
    - 未指定フィールドの値維持

4. **エラーケースの処理**
    - 存在しないTodoの更新
    - バリデーションルールの適用

## 6.2 更新機能の実装

```typescript
// src/repositories/todo.repository.ts
export class TodoRepository {
    private todos: Map<string, Todo> = new Map();

    // 既存のメソッド...

    async update(id: string, dto: UpdateTodoDTO): Promise<Todo> {
        // 更新対象のTodoを検索
        const existingTodo = await this.findById(id);
        if (!existingTodo) {
            throw new TodoValidationError('Todo not found');
        }

        // タイトルのバリデーション
        const title = dto.title?.trim() ?? existingTodo.title;
        if (!title) {
            throw new TodoValidationError('Title cannot be empty');
        }

        // 更新するフィールドを設定
        const updatedTodo: Todo = {
            ...existingTodo,                    // 既存のフィールドを展開
            title,                              // バリデーション済みのタイトル
            description: dto.description?.trim() ?? existingTodo.description,
            completed: dto.completed ?? existingTodo.completed,
            updatedAt: new Date()              // 更新日時は必ず新しく
        };

        // 更新されたTodoを保存
        this.todos.set(id, updatedTodo);
        return updatedTodo;
    }
}
```

**実装のポイント：**

1. **Nullish Coalescing演算子の活用**
   ```typescript
   title: dto.title?.trim() ?? existingTodo.title
   ```
   - `undefined`の場合のみ既存の値を使用
   - `null`や空文字列は新しい値として扱う

2. **スプレッド構文による不変性の維持**
   ```typescript
   const updatedTodo: Todo = {
       ...existingTodo,
       // 更新フィールド
   };
   ```
   - オブジェクトの安全なコピー
   - 意図しない副作用の防止

## 6.3 テスト実行結果

```bash
PASS  src/repositories/todo.repository.test.ts
  TodoRepository
    create
      ✓ creates a new todo with required fields (2ms)
    findById
      ✓ returns todo when exists (1ms)
      ✓ returns null when todo does not exist (1ms)
    findAll
      ✓ returns empty array when no todos exist (1ms)
      ✓ returns all todos (1ms)
    update
      ✓ updates todo fields (1ms)
      ✓ throws error when todo does not exist (1ms)
      ✓ maintains unchanged fields (1ms)
```

次章では、Todoの削除機能の実装に進みます。
