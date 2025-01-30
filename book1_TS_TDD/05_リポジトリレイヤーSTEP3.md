# 第5章 TodoRepositoryの検索機能実装

## 5.1 検索機能の設計思想

データの検索機能は、アプリケーションの重要な機能の一つです。  
リポジトリレイヤーでの検索機能には、以下の要件があります。

1. データの取得
   - 単一のデータの取得（IDによる検索）
   - データ一覧の取得（全件取得）

2. 検索結果の整合性
   - 存在しないデータの適切な処理
   - 型安全な結果の返却

3. 非同期処理への対応
   - データベースアクセスを想定した設計
   - Promise型を活用した実装

## 5.2 IDによる検索機能

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

**検索機能テストのポイント：**

1. **正常系のテスト**
   - 存在するデータの検索
   - 検索結果の完全一致確認

2. **異常系のテスト**
   - 存在しないIDでの検索
   - `null`が返却されることの確認
  
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

### 2. 検索機能の実装

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

**実装のポイント：**

1. **戻り値の型定義**
   ```typescript
   Promise<Todo | null>
   ```
   - 検索結果が存在しない可能性を型で表現
   - `null`を明示的に返却することで、未検出を示す

2. **Map.get()メソッドの活用**
   - JavaScriptのMapオブジェクトの機能を利用
   - 存在しない場合は`undefined`が返却されるため、`null`に変換

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

## 5.3 全件取得機能

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

**全件取得テストのポイント：**

1. **初期状態のテスト**
   - データが存在しない場合の挙動確認
   - 空配列が返却されることを検証

2. **複数データの取得確認**
   - 複数のテストデータを作成
   - 件数の確認
   - データの内容確認


テスト結果
```
FAIL  src/repositories/todo.repository.test.ts
  ● TodoRepository › findAll › returns empty array when no todos exist
    TypeError: repository.findAll is not a function
```

`findAll`メソッドを実装しましょう。

### 2. 全件取得機能の実装

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

1. **配列への変換**
   - `Map.values()`でTodoのイテレータを取得
   - `Array.from()`でイテレータを配列に変換

2. **型定義の明確化**
   ```typescript
   Promise<Todo[]>
   ```
   - 戻り値の型を明示的に定義
   - 常に配列を返却することを保証

## 5.4 テスト実行結果

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

## 5.5 検索機能の特徴とまとめ

### 1. 型安全性の確保
- `null`や空配列の適切な取り扱い
- 戻り値の型を明確に定義
- コンパイル時の型チェックによるエラー防止

### 2. 将来の拡張性
- 非同期処理を前提とした設計
- データベース実装への移行を考慮
- クエリの拡張に対応可能な構造

### 3. テストの網羅性
- 正常系・異常系の両方をカバー
- データ不在時の動作確認
- 複数データの取り扱いを確認

この章では、TodoRepositoryの基本的な検索機能を実装しました。  
次章では、Todoの更新機能の実装に進みます。
