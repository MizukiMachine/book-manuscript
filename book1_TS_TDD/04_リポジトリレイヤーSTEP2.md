# 第4章 TodoRepositoryのバリデーション実装

## 4.1 バリデーションの役割と設計思想

バリデーション（検証）とは、データの整合性と品質を確保するための重要な処理です。  
リポジトリレイヤーでのバリデーションは、以下の目的で実施します。

1. データの整合性確保
   - 必須項目の存在確認
   - データ形式の検証
   - ビジネスルールの適用

2. データベースの保護
   - 不正なデータの混入防止
   - データ構造の一貫性維持

3. エラーの早期検出
   - データ保存前の問題検出
   - 適切なエラーメッセージの提供

## 4.2 バリデーションのテスト実装

まず、バリデーション機能のテストケースを追加します。

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

**バリデーションテストのポイント：**

1. **エラーケースのテスト**
   ```typescript
   await expect(repository.create(dto)).rejects.toThrow()
   ```
   - 非同期処理のエラーを検証する場合は`.rejects`を使用
   - エラーメッセージの内容まで検証することで、適切なエラー処理を確認

2. **データの正規化テスト**
   - 入力値の前後の空白が適切に除去されることを確認
   - オプショナルフィールドの処理が正しく行われることを確認

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

## 4.3 バリデーション機能の実装

テストの実装に応じて、リポジトリの実装を修正します。

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
   ```typescript
   const title = dto.title.trim();
   ```
   - `trim()`メソッドで文字列の前後の空白を除去
   - データの一貫性を確保

2. **バリデーションチェック**
   ```typescript
   if (!title) {
       throw new Error('Title cannot be empty');
   }
   ```
   - 空文字列のチェック
   - 具体的なエラーメッセージの提供

3. **オプショナルチェーン演算子の活用**
   ```typescript
   description: dto.description?.trim()
   ```
   - `?.`演算子で安全にオプショナルな値を処理
   - `undefined`の場合でもエラーにならない実装

テスト実行結果：

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

## 4.4 エラーハンドリングの改善

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

より明確なエラー処理のため、カスタムエラークラスを実装します。

```typescript
// src/errors/todo.errors.ts
export class TodoValidationError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'TodoValidationError';
    }
}
```

**カスタムエラーのポイント：**

1. **エラーの種類を明確化**
   - 標準の`Error`クラスを継承
   - エラーの名前を明示的に設定
   - エラーの発生源を特定しやすく

2. **エラーメッセージの統一**
   - 一貫性のあるエラーメッセージ
   - エラーの原因を具体的に説明
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


**ポイント解説**

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

この章では、データの整合性を確保するためのバリデーション機能を実装しました。  
次章では、Todoの検索機能の実装に進みます。
