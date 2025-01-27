## 3. バリデーションとエラーハンドリングの実装

まず、バリデーション用のテストケースを追加します

```typescript
// src/controllers/todo.controller.test.ts
describe('POST /todos', () => {
    // 既存のテスト...

    test('returns 400 when title is missing', async () => {
        const response = await request(app)
            .post('/todos')
            .send({
                description: 'Test Description'
            });
        // タイトルがないリクエストを送信→400エラーとエラーメッセージを確認

        expect(response.status).toBe(400);
        expect(response.body).toEqual({
            errors: ['Title is required']
        });
    });

    test('returns 400 when title is empty', async () => {
        const response = await request(app)
            .post('/todos')
            .send({
                title: '',
                description: 'Test Description'
            });
        // 空のタイトルでリクエスト→400エラーとエラーメッセージを確認

        expect(response.status).toBe(400);
        expect(response.body).toEqual({
            errors: ['Title must not be empty']
        });
    });

    test('handles service errors gracefully', async () => {
        const response = await request(app)
            .post('/todos')
            .send({
                title: 'a'.repeat(101),  // 最大長を超過
                description: 'Test Description'
            });
        // 長すぎるタイトルでリクエスト→400エラーとエラーメッセージを確認

        expect(response.status).toBe(400);
        expect(response.body).toEqual({
            errors: ['Title cannot exceed 100 characters']
        });
    });
});
```
これらのテストにより、APIが異常系の入力に対して適切に対応できることを確認


続いて、express-validatorを使用してバリデーションを実装します。
```typescript
// src/middleware/validation.ts
import { body, ValidationChain } from 'express-validator';

export const createTodoValidation: ValidationChain[] = [
    body('title')
        .exists()
        .withMessage('Title is required')
        .bail()
        .notEmpty()
        .withMessage('Title must not be empty')
        .trim(),
    
    body('description')
        .optional()
        .trim(),
];
```
- `.bail()` を追加して、最初のバリデーションエラーで処理を停止
- これにより、titleが存在しない場合は "Title is required" のみが返される
- titleが空文字の場合は "Title must not be empty" が返される

コントローラーの実装を更新します。

```typescript
// src/controllers/todo.controller.ts
import { Request, Response } from 'express';
import { validationResult } from 'express-validator';
import { TodoService } from '../services/todo.service';
import { CreateTodoDTO } from '../types';

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

テストのセットアップも更新します

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
- express-validatorを使用した入力値の検証
- エラーメッセージの統一フォーマット（errorsプロパティの配列）
- リクエストボディのトリミング
- サービス層のエラーも同じフォーマットで返却


テスト結果
```
PASS  src/controllers/todo.controller.test.ts
  TodoController
    POST /todos
      ✓ creates a new todo with valid input (45ms)
      ✓ returns 400 when title is missing (12ms)
      ✓ returns 400 when title is empty (11ms)
      ✓ handles service errors gracefully (11ms)
```

[続いて検索・更新・削除のエンドポイント実装に進みましょうか？]
