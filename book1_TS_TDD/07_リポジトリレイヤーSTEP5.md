# 第7章 TodoRepositoryの削除機能実装

## 7.1 削除機能のテスト実装

まず、Todoの削除機能に関するテストケースを実装します。

```typescript
// src/repositories/todo.repository.test.ts
describe('TodoRepository', () => {
    let repository: TodoRepository;

    beforeEach(() => {
        repository = new TodoRepository();
    });

    describe('delete', () => {
        it('deletes existing todo', async () => {
            // 削除対象のTodoを作成
            const todo = await repository.create({
                title: 'Delete Me'
            });

            // Todoを削除
            await repository.delete(todo.id);

            // 削除したTodoが取得できないことを確認
            const found = await repository.findById(todo.id);
            expect(found).toBeNull();
        });

        it('throws error when todo does not exist', async () => {
            // 存在しないIDでの削除を試みる
            await expect(
                repository.delete('non-existent-id')
            ).rejects.toThrow('Todo not found');
        });

        it('does not affect other todos', async () => {
            // 2つのTodoを作成
            const todo1 = await repository.create({ title: 'Todo 1' });
            const todo2 = await repository.create({ title: 'Todo 2' });

            // todo1を削除
            await repository.delete(todo1.id);

            // todo2は残っていることを確認
            const remainingTodos = await repository.findAll();
            expect(remainingTodos).toHaveLength(1);
            expect(remainingTodos[0]).toEqual(todo2);
        });

        it('allows creation after deletion with same title', async () => {
            // 最初のTodoを作成して削除
            const todo1 = await repository.create({ title: 'Recycled Title' });
            await repository.delete(todo1.id);

            // 同じタイトルで新しいTodoを作成
            const todo2 = await repository.create({ title: 'Recycled Title' });

            // 新しいTodoが正しく作成されていることを確認
            const found = await repository.findById(todo2.id);
            expect(found).toEqual(todo2);
            expect(found?.id).not.toBe(todo1.id);
        });
    });
});
```

今までと同様にこの段階ではテストは失敗します。  
通らないことを確認の上、実装を行います。

## 7.2 削除機能の実装

```typescript
// src/repositories/todo.repository.ts
export class TodoRepository {
    private todos: Map<string, Todo> = new Map();

    // 既存のメソッド...

    async delete(id: string): Promise<void> {
        // 削除対象のTodoが存在するか確認
        const todo = await this.findById(id);
        if (!todo) {
            throw new TodoValidationError('Todo not found');
        }

        // Todoを削除
        this.todos.delete(id);
    }
}
```

**実装のポイント：**

1. **存在確認の重要性**
   ```typescript
   const todo = await this.findById(id);
   if (!todo) {
       throw new TodoValidationError('Todo not found');
   }
   ```
   - 削除前に対象の存在を確認
   - 存在しない場合は明示的にエラーをスロー

2. **戻り値の型定義**
   ```typescript
   async delete(id: string): Promise<void>
   ```
   - 削除操作は値を返さない（`void`）
   - 非同期処理として定義（将来的なDB操作を想定）

3. **Map.delete()の利用**
   ```typescript
   this.todos.delete(id);
   ```
   - JavaScriptのMapオブジェクトの機能を活用
   - 存在しないキーの削除は無害

## 7.3 テスト実行結果

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
      ✓ updates todo fields correctly (1ms)
      ✓ maintains unchanged fields (1ms)
      ✓ throws error when todo does not exist (1ms)
    delete
      ✓ deletes existing todo (1ms)
      ✓ throws error when todo does not exist (1ms)
      ✓ does not affect other todos (1ms)
      ✓ allows creation after deletion with same title (1ms)
```

## 7.4 リポジトリレイヤーの完成

これでTodoRepositoryの基本的なCRUD操作（Create, Read, Update, Delete）が全て実装できました。  
実装したメソッドは、

1. `create`: Todoの作成
2. `findById`: IDによるTodoの検索
3. `findAll`: 全Todoの取得
4. `update`: Todoの更新
5. `delete`: Todoの削除

です。

次章からは、このTodoRepositoryを使用するサービスレイヤーの実装に進みます。
