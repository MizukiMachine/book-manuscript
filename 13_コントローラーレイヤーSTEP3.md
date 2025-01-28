# 第13章 コントローラーレイヤーの更新エンドポイント実装

## 13.1 更新エンドポイントのテストの拡充

```typescript
// src/controllers/todo.controller.test.ts
describe('TodoController', () => {
    // 既存のテストコード...

    describe('PATCH /todos/:id', () => {
        let todoId: string;

        beforeEach(async () => {
            // テスト用のTodoを作成
            const response = await request(app)
                .post('/todos')
                .send({ title: 'Original Title' });
            todoId = response.body.id;
        });

        it('updates todo with valid input', async () => {
            const response = await request(app)
                .patch(`/todos/${todoId}`)
                .send({
                    title: 'Updated Title',
                    description: 'Updated Description',
                    completed: true
                });

            expect(response.status).toBe(200);
            expect(response.body).toMatchObject({
                title: 'Updated Title',
                description: 'Updated Description',
                completed: true
            });
        });

        it('returns 404 when todo not found', async () => {
            const response = await request(app)
                .patch('/todos/non-existent-id')
                .send({ title: 'Updated Title' });

            expect(response.status).toBe(404);
            expect(response.body).toEqual({
                errors: ['Todo not found']
            });
        });

        it('handles validation errors', async () => {
            const response = await request(app)
                .patch(`/todos/${todoId}`)
                .send({ title: '' });

            expect(response.status).toBe(400);
            expect(response.body).toEqual({
                errors: ['Title must not be empty']
            });
        });

        it('allows partial updates', async () => {
            // まず説明文を追加
            await request(app)
                .patch(`/todos/${todoId}`)
                .send({ description: 'First Description' });

            // タイトルのみを更新
            const response = await request(app)
                .patch(`/todos/${todoId}`)
                .send({ title: 'Updated Title' });

            expect(response.status).toBe(200);
            expect(response.body).toMatchObject({
                title: 'Updated Title',
                description: 'First Description'  // 説明文が維持されていることを確認
            });
        });

        it('prevents updating completed todo', async () => {
            // Todoを完了状態に更新
            await request(app)
                .patch(`/todos/${todoId}`)
                .send({ completed: true });

            // 完了済みTodoの更新を試みる
            const response = await request(app)
                .patch(`/todos/${todoId}`)
                .send({ title: 'New Title' });

            expect(response.status).toBe(400);
            expect(response.body).toEqual({
                errors: ['Cannot update completed todo']
            });
        });
    });
});
```

## 13.2 更新用のバリデーションミドルウェアの追加

```typescript
// src/middleware/validation.ts
export const updateTodoValidation: ValidationChain[] = [
    body('title')
        .optional()
        .notEmpty()
        .withMessage('Title must not be empty')
        .trim(),
    
    body('description')
        .optional()
        .trim(),

    body('completed')
        .optional()
        .isBoolean()
        .withMessage('Completed must be a boolean value')
];
```

## 13.3 コントローラーの更新

```typescript
// src/controllers/todo.controller.ts
export class TodoController {
    constructor(private service: TodoService) {}

    async updateTodo(req: Request, res: Response): Promise<void> {
        try {
            // バリデーション結果のチェック
            const errors = validationResult(req);
            if (!errors.isEmpty()) {
                res.status(400).json({ 
                    errors: errors.array().map(err => err.msg)
                });
                return;
            }

            try {
                const todo = await this.service.updateTodo(
                    req.params.id,
                    req.body
                );
                res.json(todo);
            } catch (error) {
                if (error instanceof Error) {
                    if (error.message === 'Todo not found') {
                        res.status(404).json({ errors: [error.message] });
                        return;
                    }
                    res.status(400).json({ errors: [error.message] });
                } else {
                    throw error; // 予期せぬエラーは外側のcatchで処理
                }
            }
        } catch (error) {
            res.status(500).json({ errors: ['Internal server error'] });
        }
    }
}
```

ルーティングの更新：

```typescript
// src/controllers/todo.controller.test.ts
beforeEach(async () => {
    // ... 既存のセットアップコード

    // 更新エンドポイントにバリデーションを追加
    app.patch(
        '/todos/:id', 
        updateTodoValidation, 
        controller.updateTodo.bind(controller)
    );
});
```

## 13.4 実装のポイント解説

1. **エラーハンドリングの階層化**
   ```typescript
   try {
       // バリデーションエラー処理
       try {
           // ビジネスロジックエラー処理
       } catch (error) {
           // エラータイプに応じた処理
       }
   } catch (error) {
       // 予期せぬエラーの処理
   }
   ```
   - バリデーションエラー: 400 Bad Request
   - 存在しないTodo: 404 Not Found
   - ビジネスルールエラー: 400 Bad Request
   - 予期せぬエラー: 500 Internal Server Error

2. **部分更新のサポート**
   - バリデーションでフィールドを`optional()`に設定
   - 未指定フィールドの値を維持

3. **HTTPステータスコードの適切な使用**
   - 201: リソース作成成功
   - 200: 更新成功
   - 400: クライアントエラー
   - 404: リソース未検出
   - 500: サーバーエラー

これでTodoアプリケーションの基本的なAPIエンドポイントが完成しました。
