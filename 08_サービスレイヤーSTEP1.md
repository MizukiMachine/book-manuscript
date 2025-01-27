# 第8章 TodoServiceの基本実装

## 8.1 サービスレイヤーの役割

サービスレイヤーの担当は以下です。

- ビジネスロジックの実装
- 入力値の詳細なバリデーション
- セキュリティ対策（XSS対策など）
- リポジトリレイヤーの抽象化

## 8.2 基本実装のテスト

```typescript
// src/services/todo.service.test.ts
import { TodoService } from './todo.service';
import { TodoRepository } from '../repositories/todo.repository';
import { CreateTodoDTO } from '../types';

describe('TodoService', () => {
    let service: TodoService;
    let repository: TodoRepository;

    beforeEach(() => {
        repository = new TodoRepository();
        service = new TodoService(repository);
    });

    describe('createTodo', () => {
        it('creates a todo with valid input', async () => {
            const dto: CreateTodoDTO = {
                title: 'Test Todo',
                description: 'Test Description'
            };

            const todo = await service.createTodo(dto);
            
            expect(todo.title).toBe(dto.title);
            expect(todo.description).toBe(dto.description);
            expect(todo.completed).toBe(false);
        });

        it('sanitizes HTML content from input', async () => {
            const dto: CreateTodoDTO = {
                title: '<script>alert("xss")</script>Test Todo',
                description: '<img src="x" onerror="alert()">Description'
            };

            const todo = await service.createTodo(dto);
            
            expect(todo.title).toBe('Test Todo');
            expect(todo.description).toBe('Description');
        });
    });
});
```

**テストのポイント：**
1. サービスとリポジトリの依存関係の設定
2. 基本的なTodo作成機能の確認
3. セキュリティ対策（XSS対策）の検証

## 8.3 依存パッケージのインストール

```bash
npm install sanitize-html
npm install --save-dev @types/sanitize-html
```

## 8.4 サービスの実装

```typescript
// src/services/todo.service.ts
import { TodoRepository } from '../repositories/todo.repository';
import { CreateTodoDTO, Todo } from '../types';
import sanitizeHtml from 'sanitize-html';

export class TodoService {
    constructor(private repository: TodoRepository) {}

    async createTodo(dto: CreateTodoDTO): Promise<Todo> {
        // 入力値のサニタイズ
        const sanitizedTitle = this.sanitizeText(dto.title);
        const sanitizedDescription = dto.description 
            ? this.sanitizeText(dto.description)
            : undefined;

        // タイトルの長さチェック
        if (sanitizedTitle.length > 100) {
            throw new Error('Title cannot exceed 100 characters');
        }

        // 説明文の長さチェック（存在する場合）
        if (sanitizedDescription && 
            sanitizedDescription.length > 500) {
            throw new Error('Description cannot exceed 500 characters');
        }

        // リポジトリを使用してTodoを作成
        return this.repository.create({
            title: sanitizedTitle,
            description: sanitizedDescription
        });
    }

    private sanitizeText(text: string): string {
        return sanitizeHtml(text, {
            allowedTags: [],       // HTMLタグを全て除去
            allowedAttributes: {}, // 属性を全て除去
        }).trim();
    }
}
```

**実装のポイント：**

1. **依存性注入**
   ```typescript
   constructor(private repository: TodoRepository) {}
   ```
   - リポジトリをコンストラクタで注入
   - テストやメンテナンスを容易に

2. **入力値のサニタイズ**
   ```typescript
   private sanitizeText(text: string): string {
       return sanitizeHtml(text, {
           allowedTags: [],
           allowedAttributes: {},
       }).trim();
   }
   ```
   - XSS攻撃の防止
   - HTMLタグと属性の完全な除去

3. **バリデーションルール**
   ```typescript
   if (sanitizedTitle.length > 100) {
       throw new Error('Title cannot exceed 100 characters');
   }
   ```
   - 明確な長さ制限
   - エラーメッセージの具体化

## 8.5 テスト実行結果

```bash
PASS  src/services/todo.service.test.ts
  TodoService
    createTodo
      ✓ creates a todo with valid input (3ms)
      ✓ sanitizes HTML content from input (1ms)
```

## 8.6 リポジトリレイヤーとの違い

サービスレイヤーでは、リポジトリレイヤーにはない以下の処理を追加しています：

1. **セキュリティ対策**
   - HTMLコンテンツのサニタイズ
   - 悪意のあるスクリプトの除去

2. **より厳密なバリデーション**
   - 文字数制限
   - コンテンツの品質チェック

3. **ビジネスルールの適用**
   - より具体的なエラーメッセージ
   - アプリケーション固有のルール

次章では、より高度なバリデーションとビジネスルールの実装に進みます。
