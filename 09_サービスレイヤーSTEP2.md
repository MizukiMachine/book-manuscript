## 3. 更新機能の実装

更新機能のテストを追加します

```typescript
// src/services/todo.service.test.ts
import { CreateTodoDTO, UpdateTodoDTO } from '../types';

describe('updateTodo', () => {
    test('updates todo with valid input', async () => {
        // まず新しいTodoを作成
        const created = await service.createTodo({
            title: 'Original Title',
            description: 'Original Description'
        });

        const updateDto: UpdateTodoDTO = {
            title: 'Updated Title',
            description: 'Updated Description',
            completed: true
        };

        const updated = await service.updateTodo(created.id, updateDto);

        expect(updated.title).toBe(updateDto.title);
        expect(updated.description).toBe(updateDto.description);
        expect(updated.completed).toBe(true);
    });

    test('sanitizes and validates updated fields', async () => {
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

    test('prevents updating completed todo', async () => {
        // 完了済みのTodoを作成
        const todo = await service.createTodo({ title: 'Test Todo' });
        await service.updateTodo(todo.id, { completed: true });

        // 完了済みTodoの更新を試みる
        await expect(
            service.updateTodo(todo.id, { title: 'New Title' })
        ).rejects.toThrow('Cannot update completed todo');
    });

    test('throws error when todo not found', async () => {
        await expect(
            service.updateTodo('non-existent-id', { title: 'New Title' })
        ).rejects.toThrow('Todo not found');
    });
});
```
- `rejects.toThrow()` を使用して、適切なエラーメッセージが投げられることを確認しています


更新機能の実装の方を行います。
```typescript
// src/services/todo.service.ts
import { CreateTodoDTO, Todo, UpdateTodoDTO } from '../types';

export class TodoService {
    // 既存のコードは維持...

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
            if (sanitizedTitle.length > this.MAX_TITLE_LENGTH) {
                throw new Error('Title cannot exceed 100 characters');
            }
            updateData.title = sanitizedTitle;
        }

        // 説明文の更新処理
        if (dto.description !== undefined) {
            const sanitizedDescription = this.sanitizeText(dto.description);
            if (sanitizedDescription.length > this.MAX_DESCRIPTION_LENGTH) {
                throw new Error('Description cannot exceed 500 characters');
            }
            updateData.description = sanitizedDescription;
        }

        // 完了状態の更新
        if (dto.completed !== undefined) {
            updateData.completed = dto.completed;
        }

        return this.repository.update(id, updateData);
    }
}
```
- 各フィールドを個別に検証・サニタイズ
- 完了済みTodoの更新を禁止
- 部分的な更新をサポート（指定されたフィールドのみ更新）
- 入力値の検証は作成時と同じルールを適用
- 空のフィールドは更新しない（undefined時はスキップ）
- `updateTodo()` メソッド (TodoService クラス内)
    - **ビジネスロジックを担当**
- `update()` メソッド (repository クラス内)
    - **データアクセス処理を担当**


テスト結果
```
PASS  src/services/todo.service.test.ts
  TodoService
    createTodo
      ✓ creates a todo with valid input (3ms)
      ✓ throws error for title exceeding maximum length (1ms)
      ✓ throws error for description exceeding maximum length (1ms)
      ✓ sanitizes malicious content in title and description (2ms)
      ✓ trims whitespace from title and description (1ms)
    updateTodo
      ✓ updates todo with valid input (2ms)
      ✓ sanitizes and validates updated fields (1ms)
      ✓ prevents updating completed todo (1ms)
      ✓ throws error when todo not found (1ms)
```
