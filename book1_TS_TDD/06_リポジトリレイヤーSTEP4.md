# 第6章 TodoRepositoryの更新機能実装

## 6.1 更新機能の設計思想

更新機能は、既存のデータを変更するための重要な機能です。  
リポジトリレイヤーでの更新機能には、以下の要件があります。

1. データの整合性確保
   - 存在するデータの検証
   - 更新日時の管理
   - 部分更新への対応

2. エラー処理
   - 存在しないデータへの対応
   - バリデーションエラーの処理
   - 適切なエラーメッセージの提供

3. データの永続性
   - 更新内容の保存
   - 更新前データの保持
   - トランザクション的な動作の確保

## 6.2 更新機能のテスト実装

まず、共通のテストユーティリティファイルを作成します。  
これは更新日時の検証のために必要です。

```typescript
// src/test-utils/helpers.ts

// テストヘルパー関数：時間差を作るため
export const wait = (ms: number = 1) => new Promise(resolve => setTimeout(resolve, ms));
```

次に、更新機能のテストケースを実装します。

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
            expect(updated.updatedAt.getTime()).toBeGreaterThan(todo.updatedAt.getTime());
            expect(updated.id).toBe(todo.id);// IDは変更されないことを確認
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
            expect(updated.updatedAt.getTime()).toBeGreaterThan(todo.updatedAt.getTime());
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

**更新機能テストのポイント：**

1. **時間差の制御**
   - `wait`関数による意図的な遅延
   - 更新日時の確実な変更を検証

2. **部分更新の検証**
   - 一部のフィールドのみを更新
   - 未更新フィールドの値維持を確認

3. **エラーケースの網羅**
   - 存在しないTodoの更新試行
   - バリデーションルールの適用確認

## 6.3 更新機能の実装

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
   - `??`演算子で`undefined`の場合のみ既存の値を使用
   - `null`や空文字列は新しい値として扱う

2. **スプレッド構文による不変性の維持**
   ```typescript
   const updatedTodo: Todo = {
       ...existingTodo,
       // 更新フィールド
   };
   ```
   - オブジェクトの安全なコピー
   - 意図しない変更の防止

3. **部分更新の実現**
   - 更新されないフィールドは既存の値を維持
   - 明示的に指定されたフィールドのみを更新

## 6.4 テスト実行結果

```bash
PASS  src/repositories/todo.repository.test.ts
  TodoRepository
    update
      ✓ updates todo fields correctly (3ms)
      ✓ maintains unchanged fields (2ms)
      ✓ throws error when todo does not exist (1ms)
      ✓ applies validation rules to updated fields (2ms)
```

## 6.5 更新機能の特徴とまとめ

### 1. データの整合性確保
- 存在チェックによる安全な更新
- バリデーションルールの適用
- 更新日時の自動管理

### 2. 柔軟な更新機能
- 部分更新のサポート
- 未指定フィールドの既存値維持
- 型安全な更新処理

### 3. エラー処理の充実
- 存在しないデータへの対応
- バリデーションエラーの処理
- 明確なエラーメッセージ

### 4. 実装上の工夫
1. **オプショナルチェーンの活用**
   ```typescript
   dto.title?.trim()
   ```
   - 安全なプロパティアクセス
   - `undefined`の適切な処理

2. **不変性を意識した実装**
   - 新しいオブジェクトの作成
   - 副作用の防止
   - デバッグのしやすさ

3. **型システムの活用**
   - UpdateTodoDTOによる更新可能フィールドの制限
   - コンパイル時の型チェック
   - 実行時エラーの防止

この章では、TodoRepositoryの更新機能を実装しました。  
次章では、Todoの削除機能の実装に進みます。
