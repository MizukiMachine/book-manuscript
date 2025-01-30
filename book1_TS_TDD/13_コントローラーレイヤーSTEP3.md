# 第13章 コントローラーレイヤーの更新エンドポイント実装

## 13.1 更新エンドポイントの設計思想

RESTful APIの更新エンドポイントには、以下の重要な設計要素があります。

1. **HTTPメソッドの選択**
   - PATCH: 部分的な更新
   - PUT: リソースの完全な置き換え
   - 今回はPATCHを採用し、柔軟な更新を実現

2. **パスパラメータの利用**
   - URLに含まれるID指定
   - リソースの一意な特定
   - RESTfulな設計の実現

3. **エラー応答の設計**
   - 404: リソース未検出
   - 400: バリデーションエラー
   - 500: サーバーエラー

## 13.2 更新エンドポイントのテスト

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

**更新エンドポイントテストのポイント：**

追加したテストケースが多いので、詳しく説明します。

各テストではTodoの更新機能における様々なシナリオを検証します。  
`beforeEach`では、各テストケース実行前に、必ず一つのTodoを作成し、そのIDを`todoId`として保持します。

**1. 基本的な更新の検証**
```typescript
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
```
このテストでは、タイトル・説明文・完了状態 の3つのフィールドすべての更新が正しく行われることを確認しています。  
HTTPステータス200が返され、更新後のTodoオブジェクトが期待通りの値を持っていることを検証します。

**2. 存在しないTodoへの更新要求の検証**
```typescript
it('returns 404 when todo not found', async () => {
    const response = await request(app)
        .patch('/todos/non-existent-id')
        .send({ title: 'Updated Title' });

    expect(response.status).toBe(404);
    expect(response.body).toEqual({
        errors: ['Todo not found']
    });
});
```
存在しないIDに対する更新リクエストでは、適切な404エラーと、  
ユーザーフレンドリーなエラーメッセージが返されることを確認します。

**3. バリデーションエラーの検証**
```typescript
it('handles validation errors', async () => {
    const response = await request(app)
        .patch(`/todos/${todoId}`)
        .send({ title: '' });

    expect(response.status).toBe(400);
    expect(response.body).toEqual({
        errors: ['Title must not be empty']
    });
});
```
タイトルを空文字列に更新しようとした場合、バリデーションエラーが発生し、  
400エラーと適切なエラーメッセージが返されることを確認します。

**4. 部分更新の検証**
```typescript
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
        description: 'First Description'
    });
});
```
このテストは2段階の更新を行い、PATCHメソッドによる部分更新が正しく機能することを確認します。
1. まず説明文のみを更新
2. 次にタイトルのみを更新
3. 最後に、タイトルが新しい値に、説明文が以前の値のままであることを確認

**5. 完了済みTodoの更新防止の検証**
```typescript
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
```
このテストは、ビジネスルールの一つである「完了済みTodoは更新不可」を検証します。
1. まずTodoを完了状態に更新
2. その後、完了済みTodoのタイトル更新を試みる
3. 適切な400エラーとエラーメッセージが返されることを確認


## 13.3 バリデーションミドルウェアの実装

更新用のバリデーションルールを定義します。

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

**バリデーションの設計ポイント：**

1. **部分更新への対応**
   - すべてのフィールドを`.optional()`に設定
   - 更新時の柔軟性を確保

2. **フィールドごとの検証**
   - タイトル：空文字列のチェック
   - 完了状態：真偽値の検証

## 13.4 更新エンドポイントの実装

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

こちらもルーティングのセットアップを更新しましょう。（ `updateTodoValidation` を追加）

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
**実装のポイント：**
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


## 13.5 RESTful更新処理のベストプラクティス

更新エンドポイントの実装では、以下のRESTful原則に従っています。
改めて整理すると、

1. **リソース指向の設計**
   - URLでリソースを特定
   - HTTPメソッドで操作を表現
   - 一貫性のあるインターフェース

2. **部分更新のサポート**
   - PATCHメソッドの適切な使用
   - 必要なフィールドのみの更新
   - 既存値の保持

3. **エラー処理の一貫性**
   - 明確なステータスコード
   - エラーメッセージの統一
   - クライアントへの適切なフィードバック

3. **HTTPステータスコードの適切な使用**
   - 200: 更新成功
   - 400: クライアントエラー
   - 404: リソース未検出
   - 500: サーバーエラー

この章ではRESTfulなTodo更新エンドポイントを実装し、部分更新や適切なエラーハンドリングを実現しました。  

これでTodoアプリケーションの基本的なAPIエンドポイントが完成しました!

次章では、アプリケーション全体の動作確認に進みます。
