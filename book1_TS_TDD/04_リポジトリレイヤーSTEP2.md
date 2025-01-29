# 第4章 TodoRepositoryのバリデーション実装

## 4.1 バリデーション機能の追加

前章で基本的なTodo作成機能を実装しましたが、ここではデータの整合性を保つためのバリデーション機能を追加していきます。

### 1. バリデーションのテスト実装

```typescript
// src/repositories/todo.repository.test.ts
describe('TodoRepository', () => {
    let repository: TodoRepository;

    beforeEach(() => {
        repository = new TodoRepository();
    });

    describe('create', () => {
        // 既存のテストは維持したまま、バリデーション用のテストを追加
        it('throws error for empty title', async () => {
            const dto: CreateTodoDTO = {
                title: ''
            };

            await expect(repository.create(dto))
                .rejects
                .toThrow('Title cannot be empty');
        });

        it('throws error for title with only whitespace', async () => {
            const dto: CreateTodoDTO = {
                title: '   '
            };

            await expect(repository.create(dto))
                .rejects
                .toThrow('Title cannot be empty');
        });

        it('trims whitespace from title', async () => {
            const dto: CreateTodoDTO = {
                title: '  Test Todo  '
            };

            const todo = await repository.create(dto);
            expect(todo.title).toBe('Test Todo');
        });

        it('trims whitespace from description when provided', async () => {
            const dto: CreateTodoDTO = {
                title: 'Test Todo',
                description: '  Test Description  '
            };

            const todo = await repository.create(dto);
            expect(todo.description).toBe('Test Description');
        });
    });
});
```

**テストケースの解説：**
- 空のタイトルを検出
- スペースのみのタイトルを検出
- タイトルの前後の空白を除去
- 説明文の前後の空白を除去（オプショナルフィールド）

先程同様にテストを実行してみましょう。

```bash
npm test
FAIL  src/repositories/todo.repository.test.ts
  TodoRepository
    create
      ✓ creates a new todo with required fields (3ms)
      ✕ throws error for empty title (4ms)
      ✕ throws error for title with only whitespace (1ms)
      ✕ trims whitespace from title (1ms)

  ● TodoRepository › create › throws error for empty title
    expect(received).toThrow()
    Received function did not throw

  ● TodoRepository › create › throws error for title with only whitespace
    expect(received).toThrow()
    Received function did not throw
    
  ● TodoRepository › create › trims whitespace from title
    expect(received).toBe(expected)
    Expected: "Test Todo"
    Received: "  Test Todo  "
```

テストが失敗したので、これらのテストを通すように実装を修正します。

### 2. バリデーション機能の実装

```typescript
// src/repositories/todo.repository.ts
export class TodoRepository {
    private todos: Map<string, Todo> = new Map();

    async create(dto: CreateTodoDTO): Promise<Todo> {
        // バリデーションを追加
        const title = dto.title.trim();
        if (!title) {
            throw new Error('Title cannot be empty');
        }

        const todo: Todo = {
            id: uuidv4(),
            title: title,  // トリム済みのタイトルを使用
            description: dto.description?.trim(),  // 説明文もトリム
            completed: false,
            createdAt: new Date(),
            updatedAt: new Date()
        };

        this.todos.set(todo.id, todo);
        return todo;
    }
}
```

**実装のポイント：**
1. **入力値の正規化**
   - `trim()`メソッドで前後の空白を除去
   - 一貫性のあるデータ形式を保証

2. **バリデーションチェック**
   - 空文字列のチェック
   - エラーメッセージの明確な定義

3. **オプショナルチェーン演算子の活用**
   ```typescript
   description: dto.description?.trim()
   ```
   - `undefined`の場合でもエラーにならない安全な実装
   - TypeScriptの型システムと連携

### 3. テスト実行結果

```bash
npm test
PASS  src/repositories/todo.repository.test.ts
  TodoRepository
    create
      ✓ creates a new todo with required fields (3ms)
      ✓ throws error for empty title (1ms)
      ✓ throws error for title with only whitespace (1ms)
      ✓ trims whitespace from title (1ms)
      ✓ trims whitespace from description when provided (1ms)
```

## 4.2 エラーハンドリングの改善


### 1. テストケースの更新

```typescript
// src/repositories/todo.repository.test.ts
import { TodoValidationError } from '../errors/todo.errors';

describe('TodoRepository', () => {
    describe('create', () => {
        it('creates a new todo with required fields', async () => {
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
        it('throws TodoValidationError for empty title', async () => {
          const dto: CreateTodoDTO = {
              title: ''
          };

          await expect(repository.create(dto))
              .rejects
              .toThrow(TodoValidationError);
          
          await expect(repository.create(dto))
              .rejects
              .toHaveProperty('name', 'TodoValidationError');
        });

        // ... 残りの実装
    });
});
```

**エラーハンドリングの利点：**
1. エラーの種類を明確に区別可能
2. エラーに応じた適切な処理の実装が容易
3. デバッグ時のエラー追跡が容易

### 2. カスタムエラークラスの実装

より明確なエラーハンドリングのため、カスタムエラークラスを実装します：

```typescript
// src/errors/todo.errors.ts
export class TodoValidationError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'TodoValidationError';
    }
}
```

### 3. リポジトリの更新

```typescript
// src/repositories/todo.repository.ts
import { TodoValidationError } from '../errors/todo.errors';

export class TodoRepository {
    async create(dto: CreateTodoDTO): Promise<Todo> {
        const title = dto.title.trim();
        if (!title) {
            throw new TodoValidationError('Title cannot be empty');
        }

        // ... 残りの実装
    }
}
```


## 4.3 ポイント解説

1. **バリデーションの一貫性**
   - 同じ種類のチェックは同じ方法で実装
   - エラーメッセージの統一

2. **早期リターン**
   ```typescript
   if (!title) {
       throw new TodoValidationError('Title cannot be empty');
   }
   ```
   - バリデーションは処理の最初に実行
   - コードの可読性向上

3. **型安全性の維持**
   ```typescript
   description: dto.description?.trim()
   ```
   - オプショナルな値の安全な処理
   - コンパイル時のエラー検出

次章では、Todoの検索機能の実装に進みます。
