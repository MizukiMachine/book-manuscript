# 第9章 TodoServiceの更新機能実装

## 9.1 更新機能のテスト実装

```typescript
// src/services/todo.service.test.ts
import { wait } from '../test-utils/helpers';

describe('TodoService', () => {
    let service: TodoService;
    let repository: TodoRepository;


    beforeEach(() => {
        repository = new TodoRepository();
        service = new TodoService(repository);
    });

    describe('updateTodo', () => {
        it('updates todo with valid input', async () => {
            // まず新しいTodoを作成
            const created = await service.createTodo({
                title: 'Original Title',
                description: 'Original Description'
            });

            await wait();

            const updateDto: UpdateTodoDTO = {
                title: 'Updated Title',
                description: 'Updated Description',
                completed: true
            };

            const updated = await service.updateTodo(created.id, updateDto);

            // 更新結果の検証
            expect(updated.title).toBe(updateDto.title);
            expect(updated.description).toBe(updateDto.description);
            expect(updated.completed).toBe(true);
            expect(updated.updatedAt.getTime()).toBeGreaterThan(created.updatedAt.getTime());
        });

        it('sanitizes and validates updated fields', async () => {
            const created = await service.createTodo({
                title: 'Original Title'
            });

            const updateDto: UpdateTodoDTO = {
                title: '<script>alert("xss")</script>Updated Title  ',
                description: '  <b>New</b> Description  '
            };

            const updated = await service.updateTodo(created.id, updateDto);

            expect(updated.title).toBe('Updated Title');
            expect(updated.description).toBe('New Description');
        });

        it('prevents updating completed todo', async () => {
            // 完了済みのTodoを作成
            const todo = await service.createTodo({ title: 'Test Todo' });
            await service.updateTodo(todo.id, { completed: true });

            // 完了済みTodoの更新を試みる
            await expect(
                service.updateTodo(todo.id, { title: 'New Title' })
            ).rejects.toThrow('Cannot update completed todo');
        });

        it('enforces maximum length constraints', async () => {
            const todo = await service.createTodo({ title: 'Test Todo' });

            await expect(
                service.updateTodo(todo.id, { 
                    title: 'a'.repeat(101) 
                })
            ).rejects.toThrow('Title cannot exceed 100 characters');

            await expect(
                service.updateTodo(todo.id, { 
                    description: 'a'.repeat(501) 
                })
            ).rejects.toThrow('Description cannot exceed 500 characters');
        });
    });
});
```

## 9.2 更新機能の実装

```typescript
// src/services/todo.service.ts
export class TodoService {
    constructor(private repository: TodoRepository) {}

    async updateTodo(id: string, dto: UpdateTodoDTO): Promise<Todo> {
        // 既存のTodoを取得
        const existingTodo = await this.repository.findById(id);
        if (!existingTodo) {
            throw new Error('Todo not found');
        }

        // 完了済みTodoの更新を防止
        if (existingTodo.completed) {
            throw new Error('Cannot update completed todo');
        }

        // 更新用のDTOを準備
        const updateData: UpdateTodoDTO = {};

        // タイトルの更新処理
        if (dto.title !== undefined) {
            const sanitizedTitle = this.sanitizeText(dto.title);
            if (sanitizedTitle.length > 100) {
                throw new Error('Title cannot exceed 100 characters');
            }
            updateData.title = sanitizedTitle;
        }

        // 説明文の更新処理
        if (dto.description !== undefined) {
            const sanitizedDescription = this.sanitizeText(dto.description);
            if (sanitizedDescription.length > 500) {
                throw new Error('Description cannot exceed 500 characters');
            }
            updateData.description = sanitizedDescription;
        }

        // 完了状態の更新
        if (dto.completed !== undefined) {
            updateData.completed = dto.completed;
        }

        // リポジトリを使用してTodoを更新
        return this.repository.update(id, updateData);
    }
}
```

## 9.3 実装のポイント解説

1. **入力値の検証とサニタイズ**
   ```typescript
   const sanitizedTitle = this.sanitizeText(dto.title);
   if (sanitizedTitle.length > 100) {
       throw new Error('Title cannot exceed 100 characters');
   }
   ```
   - HTMLインジェクション対策
   - 文字数制限の適用

2. **ビジネスルールの適用**
   ```typescript
   if (existingTodo.completed) {
       throw new Error('Cannot update completed todo');
   }
   ```
   - 完了済みTodoの更新を禁止
   - アプリケーション固有のルール実装

3. **部分更新の処理**
   ```typescript
       // タイトルの更新処理
   if (dto.title !== undefined) {
   }
   ```
   - 指定されたフィールドのみを更新
   - undefined チェックによる安全な更新

## 9.4 テスト実行結果

```bash
PASS  src/services/todo.service.test.ts
  TodoService
    createTodo
      ✓ creates a todo with valid input (3ms)
      ✓ sanitizes HTML content from input (1ms)
    updateTodo
      ✓ updates todo with valid input (2ms)
      ✓ sanitizes and validates updated fields (1ms)
      ✓ prevents updating completed todo (1ms)
      ✓ enforces maximum length constraints (2ms)
```

## 9.5 リポジトリレイヤーとの違いの確認

1. **バリデーションの違い**
   - リポジトリ：基本的なデータ整合性のチェック
   - サービス：ビジネスルールに基づく詳細なバリデーション

2. **エラー処理の違い**
   - リポジトリ：データ操作に関する基本的なエラー
   - サービス：ビジネスルールに関連する具体的なエラー

3. **責務の違い**
   - リポジトリ：データの永続化と取得
   - サービス：ビジネスロジックとバリデーション

次章では、検索機能の実装に進みます。
