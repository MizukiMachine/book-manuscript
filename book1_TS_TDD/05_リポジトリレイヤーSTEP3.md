# 第5章 TodoRepositoryの検索機能実装

## 5.1 IDによる検索機能

### 1. 検索機能のテスト実装

```typescript
// src/repositories/todo.repository.test.ts
describe('TodoRepository', () => {
    let repository: TodoRepository;

    beforeEach(() => {
        repository = new TodoRepository();
    });

    describe('findById', () => {
        it('returns todo when exists', async () => {
            // テスト用のTodoを作成
            const created = await repository.create({
                title: 'Find Me'
            });

            // 作成したTodoをIDで検索
            const found = await repository.findById(created.id);
            
            // 検索結果が作成したTodoと一致することを確認
            expect(found).toEqual(created);
        });

        it('returns null when todo does not exist', async () => {
            const found = await repository.findById('non-existent-id');
            expect(found).toBeNull();
        });
    });
});
```
  


テスト結果
```
FAIL  src/repositories/todo.repository.test.ts
  TodoRepository
    create
      ✓ creates a new todo with required fields (3ms)
    findById
      ✕ returns todo when exists
      ✕ returns null when todo does not exist

  ● TodoRepository › findById › returns todo when exists
    TypeError: repository.findById is not a function
```

まだ実装していない`findById`メソッドがないためテストが失敗します。  
実装を追加しましょう。

### 5.1.2 検索機能の実装

```typescript
// src/repositories/todo.repository.ts
export class TodoRepository {
    private todos: Map<string, Todo> = new Map();

    // 既存のcreateメソッド...

    async findById(id: string): Promise<Todo | null> {
        // Mapから指定されたIDのTodoを取得
        // 存在しない場合はnullを返す
        return this.todos.get(id) || null;
    }
}
```

テスト結果
```
PASS  src/repositories/todo.repository.test.ts
  TodoRepository
    create
      ✓ creates a new todo with required fields (2ms)
    findById
      ✓ returns todo when exists (1ms)
      ✓ returns null when todo does not exist (1ms)
```

## 5.2 全件取得機能

### 1. 全件取得のテスト

```typescript
// src/repositories/todo.repository.test.ts
describe('findAll', () => {
    it('returns empty array when no todos exist', async () => {
        const todos = await repository.findAll();
        expect(todos).toEqual([]);
    });

    it('returns all todos', async () => {
        // テスト用のTodoを2件作成
        const todo1 = await repository.create({ title: 'Todo 1' });
        const todo2 = await repository.create({ title: 'Todo 2' });

        const todos = await repository.findAll();
        
        // 件数の確認
        expect(todos).toHaveLength(2);
        // 作成したTodoが含まれていることを確認
        expect(todos).toEqual(expect.arrayContaining([todo1, todo2]));
    });
});
```


テスト結果
```
FAIL  src/repositories/todo.repository.test.ts
  ● TodoRepository › findAll › returns empty array when no todos exist
    TypeError: repository.findAll is not a function
```

`findAll`メソッドを実装しましょう。

### 5.2.2 全件取得機能の実装

```typescript
// src/repositories/todo.repository.ts
export class TodoRepository {
    // 既存のコード...

    async findAll(): Promise<Todo[]> {
        // Mapの値（Todo）を配列として返す
        return Array.from(this.todos.values());
    }
}
```

**実装のポイント：**
- `Map.values()`でTodoのイテレータを取得
- `Array.from()`でイテレータを配列に変換

## 5.3 テスト実行結果

```bash
PASS  src/repositories/todo.repository.test.ts
  TodoRepository
    findById
      ✓ returns todo when exists (2ms)
      ✓ returns null when todo does not exist (1ms)
    findAll
      ✓ returns empty array when no todos exist (1ms)
      ✓ returns all todos (1ms)
```

これで検索機能の基本的な実装は完了です。  
次章では、Todoの更新機能の実装に進みます。
