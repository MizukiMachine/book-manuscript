# 第8章 TodoServiceの基本実装

## 8.1 サービスレイヤーの役割と設計思想

サービスレイヤー（Service Layer）は、アプリケーションのビジネスロジックを担当するレイヤーです。  
以下のような重要な役割を果たします。

1. **ビジネスロジックの実装**
   - アプリケーション固有のルール適用
   - 複数の操作の調整
   - データの加工や変換

2. **入力値の詳細なバリデーション**
   - ドメインルールに基づく検証
   - 複雑な条件のチェック
   - カスタムバリデーションの実装

3. **セキュリティ対策**
   - XSS対策などのセキュリティ処理
   - 入力値のサニタイズ
   - アクセス制御の実装

4. **リポジトリレイヤーの抽象化**
   - データアクセスの統合
   - トランザクション管理
   - エラーハンドリングの統一

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

**テストコードのポイント：**

1. **リポジトリとの連携**
   - リポジトリの注入
   - リポジトリ経由のデータ操作

2. **セキュリティ機能の検証**
   - XSS対策の確認
   - HTMLタグの除去
   - 安全なテキストの生成

## 8.3 依存パッケージのインストール

XSS対策のために必要なパッケージをインストールします。

```bash
npm install sanitize-html
npm install --save-dev @types/sanitize-html
```

`sanitize-html`は、HTML文字列から不要なタグや属性を除去するためのライブラリです。  
XSS（クロスサイトスクリプティング）攻撃への対策として利用します。

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
   - 前後の空白除去

3. **バリデーションルール**
   ```typescript
   if (sanitizedTitle.length > 100) {
       throw new Error('Title cannot exceed 100 characters');
   }
   ```
   - 文字数制限の実装
   - 明確なエラーメッセージ
   - ビジネスルールの適用

## 8.5 リポジトリレイヤーとの違いの解説

サービスレイヤーでは、リポジトリレイヤーにはない以下の処理を追加しています。

### 1. セキュリティ対策
- HTMLコンテンツのサニタイズ
- 悪意のあるスクリプトの除去
- 安全なデータ形式の確保

### 2. より厳密なバリデーション
- 文字数制限の実装
- 業務ルールに基づく検証
- コンテンツの品質チェック

### 3. ビジネスルールの適用
- アプリケーション固有のルール
- より具体的なエラーメッセージ
- 複雑な条件の検証

テスト実行結果：

```bash
PASS  src/services/todo.service.test.ts
  TodoService
    createTodo
      ✓ creates a todo with valid input (3ms)
      ✓ sanitizes HTML content from input (1ms)
```

この章では、ビジネスロジックを担当するサービスレイヤーの基本実装を行いました。  
特に、セキュリティとバリデーションに焦点を当てた実装を行いました。  
次章では、より高度なバリデーションとビジネスルールの実装に進みます。
