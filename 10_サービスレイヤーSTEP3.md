## 4. 検索機能の実装

検索機能のテストを追加します

```typescript
// src/services/todo.service.test.ts
describe('findTodos', () => {
    beforeEach(async () => {
        // テストデータを準備
        await service.createTodo({ title: 'Shopping', description: 'Buy groceries' });
        await service.createTodo({ title: 'Coding', description: 'Implement search' });
        await service.createTodo({ title: 'Exercise', description: 'Go to gym' });
        const completedTodo = await service.createTodo({ title: 'Reading', description: 'Read book' });
        await service.updateTodo(completedTodo.id, { completed: true });
    });

    test('finds todos by title search', async () => {
        const todos = await service.findTodos({ title: 'ing' });
        expect(todos).toHaveLength(3);
        expect(todos.map(t => t.title)).toEqual(
            expect.arrayContaining(['Shopping', 'Coding', 'Reading'])
        );
    });

    test('finds todos by completion status', async () => {
        const completed = await service.findTodos({ completed: true });
        expect(completed).toHaveLength(1);
        expect(completed[0].title).toBe('Reading');

        const incomplete = await service.findTodos({ completed: false });
        expect(incomplete).toHaveLength(3);
    });
    // 複合条件での検索テスト
    test('combines search criteria', async () => {
        const todos = await service.findTodos({
            title: 'ing',
            completed: false
        });
        expect(todos).toHaveLength(2);
        expect(todos.map(t => t.title)).toEqual(
        expect.arrayContaining(['Shopping', 'Coding'])
        );
    });
    // 該当なしケースのテスト
    test('returns empty array when no matches found', async () => {
        const todos = await service.findTodos({ title: 'nonexistent' });
        expect(todos).toHaveLength(0);
    });
});
```
- `arrayContaining` を使用して順序に依存しないテスト


検索用の型を定義します

```typescript
// src/types.ts に追加
export interface TodoSearchParams {
    title?: string;
    description?: string;
    completed?: boolean;
}
```

検索機能を実装します

```typescript
// src/services/todo.service.ts
import { CreateTodoDTO, Todo, UpdateTodoDTO, TodoSearchParams } from '../types';

export class TodoService {
    // 既存のコードは維持...

    async findTodos(params: TodoSearchParams): Promise<Todo[]> {
        // まず全てのTodoを取得
        const allTodos = await this.repository.findAll();

        // 検索条件に基づいてフィルタリング
        return allTodos.filter(todo => {
            // タイトルで検索
            if (params.title && !todo.title.toLowerCase().includes(
                params.title.toLowerCase()
            )) {
                return false;
            }

            // 説明文で検索
            if (params.description && !todo.description?.toLowerCase().includes(
                params.description.toLowerCase()
            )) {
                return false;
            }

            // 完了状態で検索
            if (params.completed !== undefined && 
                todo.completed !== params.completed) {
                return false;
            }

            return true;
        });
    }
}
```

実装のポイント
- 大文字小文字を区別しない検索
- 部分一致での検索をサポート
- 複数の検索条件を組み合わせ可能
- 未指定の条件は無視（AND条件として扱う）
- 説明文は`undefined`の可能性を考慮

実行結果
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
    findTodos
      ✓ finds todos by title search (2ms)
      ✓ finds todos by completion status (1ms)
      ✓ combines search criteria (1ms)
      ✓ returns empty array when no matches found (1ms)
```

これでTodoServiceの基本的な機能が実装できました。
次回は、このサービスを使用するコントローラーレイヤーの実装に進みます。
