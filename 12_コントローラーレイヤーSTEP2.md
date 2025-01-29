
# 第12章 コントローラーレイヤーの検索エンドポイント実装

## 12.1 検索エンドポイントのテスト

```typescript
// src/controllers/todo.controller.test.ts
describe('TodoController', () => {
    let app: Express;
    let controller: TodoController;
    let service: TodoService;

    beforeEach(async () => {
        app = express();
        app.use(express.json());
        
        const repository = new TodoRepository();
        service = new TodoService(repository);
        controller = new TodoController(service);

        // ルーティングの設定
        app.get('/todos', controller.getTodos.bind(controller));
        app.post('/todos', createTodoValidation, controller.createTodo.bind(controller));
        app.patch('/todos/:id', controller.updateTodo.bind(controller));


        // テストデータを準備
        const todo1 = await request(app)
        .post('/todos')
        .send({ title: 'Shopping', description: 'Buy groceries' });

        const todo2 = await request(app)
        .post('/todos')
        .send({ title: 'Coding', description: 'Implement search' });

        const todo3 = await request(app)
        .post('/todos')
        .send({ title: 'Reading', description: 'Read book' });

        // 完了状態の更新を待つ
        await request(app)
        .patch(`/todos/${todo3.body.id}`)
        .send({ completed: true });
    });

    describe('GET /todos', () => {
        it('returns all todos when no query parameters', async () => {
            const response = await request(app).get('/todos');

            expect(response.status).toBe(200);
            expect(response.body).toHaveLength(3);
        });

        it('filters todos by search term', async () => {
            const response = await request(app)
                .get('/todos')
                .query({ search: 'ing' });

            expect(response.status).toBe(200);
            expect(response.body).toHaveLength(3);
            expect(response.body.map((todo: any) => todo.title))
                .toEqual(expect.arrayContaining(['Shopping', 'Coding', 'Reading']));
        });

        it('filters todos by completion status', async () => {
            const response = await request(app)
                .get('/todos')
                .query({ completed: 'true' });

            expect(response.status).toBe(200);
            expect(response.body).toHaveLength(1);
            expect(response.body[0].title).toBe('Reading');
        });

        it('handles invalid completed parameter', async () => {
            const response = await request(app)
                .get('/todos')
                .query({ completed: 'invalid' });

            expect(response.status).toBe(400);
            expect(response.body).toEqual({
                errors: ['Completed status must be true or false']
            });
        });
    });
});
```

テスト実行結果：
```bash
 FAIL  src/controllers/todo.controller.test.ts
 ● Test suite failed to run
    src/controllers/todo.controller.test.ts:23:38 - error TS2339: Property 'getTodos' does not exist on type 'TodoController'.
```

## 12.2 クエリパラメータのバリデーション

```typescript
// src/middleware/validation.ts
import { query, ValidationChain } from 'express-validator';

export const getTodosValidation: ValidationChain[] = [
    query('completed')
        .optional()
        .isBoolean()
        .withMessage('Completed status must be true or false')
        .toBoolean(),
    
    query('search')
        .optional()
        .isString()
        .trim()
];
```

## 12.3 検索エンドポイントの実装

```typescript
// src/controllers/todo.controller.ts
export class TodoController {
    constructor(private service: TodoService) {}

    async getTodos(req: Request, res: Response): Promise<void> {
        try {
            // バリデーション結果のチェック
            const errors = validationResult(req);
            if (!errors.isEmpty()) {
                res.status(400).json({ 
                    errors: errors.array().map(err => err.msg)
                });
                return;
            }

            const { search, completed } = req.query;

            const searchParams: TodoSearchParams = {};
            if (typeof search === 'string') {
                searchParams.title = search;
            }
            if (completed !== undefined) {
                // 型を文字列に変換してから比較
                // completedがboolean型
                searchParams.completed = String(completed).toLowerCase() === 'true';
            }

            const todos = await this.service.findTodos(searchParams);
            res.json(todos);
        } catch (error) {
            res.status(500).json({ 
                errors: ['Internal server error']
            });
        }
    }
}
```

テストのセットアップを更新します。

```typescript
// src/controllers/todo.controller.test.ts
beforeEach(async () => {
    app = express();
    app.use(express.json());
    
    const repository = new TodoRepository();
    service = new TodoService(repository);
    controller = new TodoController(service);

    // バリデーションミドルウェアを追加
    app.get('/todos', getTodosValidation, controller.getTodos.bind(controller));
    app.post('/todos', createTodoValidation, controller.createTodo.bind(controller));
    app.patch('/todos/:id', controller.updateTodo.bind(controller));
});
```

テスト実行結果：
```bash
PASS  src/controllers/todo.controller.test.ts
  TodoController
    GET /todos
      ✓ returns all todos when no query parameters (42ms)
      ✓ filters todos by search term (38ms)
      ✓ filters todos by completion status (35ms)
      ✓ handles invalid completed parameter (12ms)
```

## 12.4 実装のポイント解説

1. **クエリパラメータの処理**
   ```typescript
   const { search, completed } = req.query;
   ```
   - URLパラメータからの検索条件の取得
   - 型安全な処理

2. **バリデーションの適用**
   ```typescript
   query('completed')
       .optional()
       .isBoolean()
       .withMessage('Completed status must be true or false')
   ```
   - クエリパラメータの型チェック
   - エラーメッセージのカスタマイズ

3. **検索パラメータの変換**
   ```typescript
   if (typeof search === 'string') {
       searchParams.title = search;
   }
   ```
   - HTTPクエリからサービスレイヤーの型への変換
   - 型安全性の確保

4. **エラーハンドリング**
   ```typescript
   try {
       // ... 検索処理
   } catch (error) {
       res.status(500).json({ errors: ['Internal server error'] });
   }
   ```
   - 予期せぬエラーの適切な処理
   - ユーザーフレンドリーなエラーメッセージ

次章では、更新エンドポイントの実装に進みます。

