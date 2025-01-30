# 第11章 コントローラーレイヤーの基本実装

## 11.1 コントローラーレイヤーの役割と設計思想

「レイヤー」の考え方をさらに発展させ、コントローラーレイヤー（Controller Layer）は、アプリケーションの最も外側に位置し、以下の重要な役割を担います。

1. **HTTPインターフェースの提供**
   - RESTful APIエンドポイントの定義
   - リクエストの受付と解析
   - レスポンスの形成と返却

2. **入力値の検証（バリデーション）**
   - HTTPリクエストの形式確認
   - 必須パラメータの検証
   - 入力値の型チェック

3. **レイヤー間の変換**
   - HTTPリクエストからDTOへの変換
   - サービスレイヤーの戻り値からHTTPレスポンスへの変換
   - エラー情報の適切な形式への変換

### コントローラーレイヤーの設計指針

1. **シンプルな責務**
   - ビジネスロジックを含まない
   - データアクセスを直接行わない
   - HTTPインターフェースとしての機能に集中

2. **適切なエラーハンドリング**
   - バリデーションエラー → 400 Bad Request
   - リソース未検出 → 404 Not Found
   - サーバーエラー → 500 Internal Server Error

3. **一貫性のある応答形式**
   - 成功時の統一されたレスポンス形式
   - エラー時の統一された形式
   - 適切なHTTPステータスコードの使用

## 11.2 必要なパッケージのインストール

Express.jsでのAPIエンドポイント実装に必要なパッケージをインストールします。

```bash
npm install express-validator
npm install --save-dev supertest @types/supertest
```

**パッケージの役割：**
- express-validator: HTTPリクエストの検証
- supertest: APIエンドポイントのテスト実行
- @types/supertest: supertestの型定義

## 11.3 基本実装のテスト

```typescript
// src/controllers/todo.controller.test.ts
import { TodoController } from './todo.controller';
import { TodoService } from '../services/todo.service';
import { TodoRepository } from '../repositories/todo.repository';
import request from 'supertest';
import express, { Express } from 'express';

describe('TodoController', () => {
    let app: Express;
    let controller: TodoController;
    let service: TodoService;

    beforeEach(() => {
        app = express();
        app.use(express.json());
        
        const repository = new TodoRepository();
        service = new TodoService(repository);
        controller = new TodoController(service);

        // ルートの設定
        app.post('/todos', controller.createTodo.bind(controller));
    });

    describe('POST /todos', () => {
        it('creates a new todo with valid input', async () => {
            const response = await request(app)
                .post('/todos')
                .send({
                    title: 'Test Todo',
                    description: 'Test Description'
                });

            expect(response.status).toBe(201);
            expect(response.body).toMatchObject({
                title: 'Test Todo',
                description: 'Test Description',
                completed: false
            });
            expect(response.body.id).toBeDefined();
        });
    });
});
```
テスト実行結果：
```bash
FAIL  src/controllers/todo.controller.test.ts
  ● Test suite failed to run
    Cannot find module './todo.controller'
```
## 11.4 コントローラーの実装

テストに対応する形でコントローラーを実装します。

```typescript
// src/controllers/todo.controller.ts
import { Request, Response } from 'express';
import { TodoService } from '../services/todo.service';
import { CreateTodoDTO } from '../types';

export class TodoController {
    constructor(private service: TodoService) {}

    async createTodo(req: Request, res: Response): Promise<void> {
        try {
            const todo = await this.service.createTodo(req.body as CreateTodoDTO);
            res.status(201).json(todo);
        } catch (error) {
            if (error instanceof Error) {
                res.status(400).json({ error: error.message });
            } else {
                res.status(500).json({ error: 'Internal server error' });
            }
        }
    }
}
```

**実装のポイント：**

1. **Expressの型定義の活用**
   ```typescript
   async createTodo(req: Request, res: Response): Promise<void>
   ```
   - Express.jsの型システムを利用
   - リクエスト/レスポンスの型安全な処理

2. **HTTPステータスコードの使い分け**
   - 201: リソース作成成功
   - 400: クライアントエラー
   - 500: サーバーエラー

テスト実行結果：
```bash
PASS  src/controllers/todo.controller.test.ts
  TodoController
    POST /todos
      ✓ creates a new todo with valid input (45ms)
```

## 11.5 バリデーションの追加
### 1. バリデーションのテスト

```typescript
// src/controllers/todo.controller.test.ts
describe('POST /todos', () => {
    // 既存のテスト...

    it('returns 400 when title is missing', async () => {
        const response = await request(app)
            .post('/todos')
            .send({
                description: 'Test Description'
            });

        expect(response.status).toBe(400);
        expect(response.body).toEqual({
            errors: ['Title is required']
        });
    });

    it('returns 400 when title is empty', async () => {
        const response = await request(app)
            .post('/todos')
            .send({
                title: '',
                description: 'Test Description'
            });

        expect(response.status).toBe(400);
        expect(response.body).toEqual({
            errors: ['Title must not be empty']
        });
    });
});
```

この段階のテスト実行結果：
```bash
FAIL  src/controllers/todo.controller.test.ts
  TodoController
    POST /todos
      ✓ creates a new todo with valid input (42ms)
      ✕ returns 400 when title is missing (12ms)
      ✕ returns 400 when title is empty (11ms)
```

### 2. バリデーションミドルウェアの実装

```typescript
// src/middleware/validation.ts
import { body, ValidationChain } from 'express-validator';

export const createTodoValidation: ValidationChain[] = [
    // titleのバリデーション
    body('title')
        .exists()
        .withMessage('Title is required')
        .bail()
        .notEmpty()
        .withMessage('Title must not be empty')
        .trim(),
    
    // descriptionのバリデーション
    body('description')
        .optional()
        .trim(),
];
```

**バリデーションルールのポイント：**

1. **必須項目の検証**
   - `.exists()`: フィールドの存在確認
   - `.notEmpty()`: 空値のチェック

2. **オプショナル項目の処理**
   - `.optional()`: 任意フィールドの指定
   - `.trim()`: 入力値の正規化

### 3. バリデーション付きコントローラーの実装

```typescript
// src/controllers/todo.controller.ts
import { validationResult } from 'express-validator';

export class TodoController {
    constructor(private service: TodoService) {}
    async createTodo(req: Request, res: Response): Promise<void> {
        try {
            // バリデーション結果のチェック
            const errors = validationResult(req);
            if (!errors.isEmpty()) {
                res.status(400).json({ 
                    errors: errors.array().map(err => err.msg)
                });
                return;
            }

            const todo = await this.service.createTodo(req.body as CreateTodoDTO);
            res.status(201).json(todo);
        } catch (error) {
            if (error instanceof Error) {
                res.status(400).json({ 
                    errors: [error.message]
                });
            } else {
                res.status(500).json({ 
                    errors: ['Internal server error']
                });
            }
        }
    }
}
```


テストのセットアップを更新します。（ `createTodoValidation` を追加）

```typescript
// src/controllers/todo.controller.test.ts
import { createTodoValidation } from '../middleware/validation';

beforeEach(() => {
    app = express();
    app.use(express.json());
    
    const repository = new TodoRepository();
    service = new TodoService(repository);
    controller = new TodoController(service);

    // バリデーションミドルウェアを追加
    app.post('/todos', createTodoValidation, controller.createTodo.bind(controller));
```


最終的なテスト実行結果：

```bash
PASS  src/controllers/todo.controller.test.ts
  TodoController
    POST /todos
      ✓ creates a new todo with valid input (45ms)
      ✓ returns 400 when title is missing (12ms)
      ✓ returns 400 when title is empty (11ms)
```


## 11.6 レイヤー間の関係性

コントローラーレイヤーは以下のような関係性を持ちます。

1. **サービスレイヤーとの関係**
   - サービスレイヤーのメソッドを呼び出し
   - ビジネスロジックの実行を委譲
   - エラーハンドリングの統合

2. **HTTPインターフェースとしての役割**
   - クライアントとの通信窓口
   - データの受け渡し形式の定義
   - エラー情報の適切な変換

この章では、HTTPインターフェースとしてのコントローラーレイヤーの基本実装を行いました。  
次章では、より高度な検索機能の実装に進みます。
