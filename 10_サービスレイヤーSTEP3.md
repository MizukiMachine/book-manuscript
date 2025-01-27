# 第10章 TodoServiceの検索機能実装

## 10.1 検索機能のテスト実装

```typescript
// src/services/todo.service.test.ts
describe('TodoService', () => {
    let service: TodoService;
    let repository: TodoRepository;

    beforeEach(async () => {
        repository = new TodoRepository();
        service = new TodoService(repository);

        // テストデータを準備
        await service.createTodo({ title: 'Shopping', description: 'Buy groceries' });
        await service.createTodo({ title: 'Coding', description: 'Implement search' });
        await service.createTodo({ title: 'Exercise', description: 'Go to gym' });
        const completedTodo = await service.createTodo({ 
            title: 'Reading', 
            description: 'Read book' 
        });
        await service.updateTodo(completedTodo.id, { completed: true });
    });

    describe('findTodos', () => {
        it('finds todos by title search', async () => {
            const todos = await service.findTodos({ title: 'ing' });
            
            expect(todos).toHaveLength(3);
            expect(todos.map(t => t.title)).toEqual(
                expect.arrayContaining(['Shopping', 'Coding', 'Reading'])
            );
        });

        it('finds todos by completion status', async () => {
            const completed = await service.findTodos({ completed: true });
            expect(completed).toHaveLength(1);
            expect(completed[0].title).toBe('Reading');

            const incomplete = await service.findTodos({ completed: false });
            expect(incomplete).toHaveLength(3);
        });

        it('combines search criteria', async () => {
            const todos = await service.findTodos({
                title: 'ing',
                completed: false
            });

            expect(todos).toHaveLength(2);
            expect(todos.map(t => t.title)).toEqual(
                expect.arrayContaining(['Shopping', 'Coding'])
            );
        });

        it('returns empty array when no matches found', async () => {
            const todos = await service.findTodos({ 
                title: 'nonexistent' 
            });
            expect(todos).toHaveLength(0);
        });

        it('performs case-insensitive search', async () => {
            const upperCase = await service.findTodos({ title: 'SHOPPING' });
            const lowerCase = await service.findTodos({ title: 'shopping' });
            
            expect(upperCase).toHaveLength(1);
            expect(lowerCase).toHaveLength(1);
            expect(upperCase[0]).toEqual(lowerCase[0]);
        });
    });
});
```

## 10.2 検索パラメータの型定義

```typescript
// src/types.ts に追加
export interface TodoSearchParams {
    title?: string;      // タイトルでの部分一致検索
    description?: string;// 説明文での部分一致検索
    completed?: boolean; // 完了状態での検索
}
```

## 10.3 検索機能の実装

```typescript
// src/services/todo.service.ts
export class TodoService {
    constructor(private repository: TodoRepository) {}

    async findTodos(params: TodoSearchParams): Promise<Todo[]> {
        // まず全てのTodoを取得
        const allTodos = await this.repository.findAll();

        // 検索条件に基づいてフィルタリング
        return allTodos.filter(todo => {
            // タイトルで検索
            if (params.title && !todo.title.toLowerCase()
                .includes(params.title.toLowerCase())) {
                return false;
            }

            // 説明文で検索
            if (params.description && !todo.description?.toLowerCase()
                .includes(params.description.toLowerCase())) {
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

## 10.4 実装のポイント解説

1. **大文字小文字を区別しない検索**
   ```typescript
   todo.title.toLowerCase().includes(params.title.toLowerCase())
   ```
   - `toLowerCase()`で大文字小文字の違いを無視
   - より使いやすい検索機能を実現

2. **オプショナルチェーンの活用**
   ```typescript
   todo.description?.toLowerCase()
   ```
   - `description`が`undefined`の場合を安全に処理
   - TypeScriptの型安全性を活用

3. **複数条件の組み合わせ**
   ```typescript
   if (params.completed !== undefined && 
       todo.completed !== params.completed) {
       return false;
   }
   ```
   - 各条件を`AND`条件として扱う
   - `undefined`の場合はその条件をスキップ

## 10.5 テスト実行結果

```bash
PASS  src/services/todo.service.test.ts
  TodoService
    findTodos
      ✓ finds todos by title search (2ms)
      ✓ finds todos by completion status (1ms)
      ✓ combines search criteria (1ms)
      ✓ returns empty array when no matches found (1ms)
      ✓ performs case-insensitive search (1ms)
```

次章では、コントローラーレイヤーの実装に進みます。

