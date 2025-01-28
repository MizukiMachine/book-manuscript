# 第11章 コントローラーレイヤーの基本実装

## 11.1 準備

まず、Express用の追加パッケージをインストールします：

```bash
npm install express-validator
npm install --save-dev supertest @types/supertest
```

## 11.2 コントローラーの基本実装

### 11.2.1 初期テストの作成

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

### 11.2.2 コントローラーの実装

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

テスト実行結果：
```bash
PASS  src/controllers/todo.controller.test.ts
  TodoController
    POST /todos
      ✓ creates a new todo with valid input (45ms)
```

## 11.3 バリデーションの追加

### 11.3.1 バリデーションのテスト

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

テスト実行結果：
```bash
FAIL  src/controllers/todo.controller.test.ts
  TodoController
    POST /todos
      ✓ creates a new todo with valid input (42ms)
      ✕ returns 400 when title is missing (12ms)
      ✕ returns 400 when title is empty (11ms)
```

### 11.3.2 バリデーションミドルウェアの実装

```typescript
// src/middleware/validation.ts
import { body, ValidationChain } from 'express-validator';

export const createTodoValidation: ValidationChain[] = [
    // titleのバリデーション
    body('title')
        .exists()        // titleフィールドが存在するか
        .withMessage('Title is required')  // エラーメッセージ
        .bail()          // 以降のバリデーションをスキップ
        .notEmpty()      // 空文字でないか
        .withMessage('Title must not be empty')
        .trim(),         // 前後の空白を削除
    
    // descriptionのバリデーション
    body('description')
        .optional()      // 任意フィールド
        .trim(),         // 前後の空白を削除
];
```

### 11.3.3 コントローラーの更新

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

テストのセットアップを更新：

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
});
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

## 11.4 実装のポイント解説

1. **レイヤー間の責務分離**
   - コントローラー：HTTPリクエスト/レスポンスの処理
   - サービス：ビジネスロジック
   - リポジトリ：データアクセス

2. **エラーハンドリング**
   - バリデーションエラー：400 Bad Request
   - サービスエラー：400 Bad Request
   - 予期せぬエラー：500 Internal Server Error

3. **express-validatorの活用**
   - リクエストの検証
   - エラーメッセージのカスタマイズ
   - 入力値の正規化（trim等）

次章では、検索・更新エンドポイントの実装に進みます。
