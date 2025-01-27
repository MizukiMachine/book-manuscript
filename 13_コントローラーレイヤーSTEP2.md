## 4. 検索エンドポイントの実装

まず、検索機能のテストを追加します

```typescript
// src/controllers/todo.controller.test.ts
describe('GET /todos', () => {
    beforeEach(async () => {
        // テストデータを準備
        await request(app)
            .post('/todos')
            .send({ title: 'Shopping', description: 'Buy groceries' });
        await request(app)
            .post('/todos')
            .send({ title: 'Coding', description: 'Implement search' });
        const completedTodo = await request(app)
            .post('/todos')
            .send({ title: 'Reading', description: 'Read book' });
        await request(app)
            .patch(`/todos/${completedTodo.body.id}`)
            .send({ completed: true });
    });

    test('returns all todos when no query parameters', async () => {
        const response = await request(app).get('/todos');

        expect(response.status).toBe(200);
        expect(response.body).toHaveLength(3);
    });

    test('filters todos by search term', async () => {
        const response = await request(app)
            .get('/todos')
            .query({ search: 'ing' });

        expect(response.status).toBe(200);
        expect(response.body).toHaveLength(2);
        expect(response.body.map((todo: any) => todo.title))
            .toEqual(expect.arrayContaining(['Shopping', 'Reading']));
    });

    test('filters todos by completion status', async () => {
        const response = await request(app)
            .get('/todos')
            .query({ completed: 'true' });

        expect(response.status).toBe(200);
        expect(response.body).toHaveLength(1);
        expect(response.body[0].title).toBe('Reading');
    });
});
```

コントローラーに検索機能を追加します。

```typescript
// src/controllers/todo.controller.ts
export class TodoController {
    // 既存のコード...

    async getTodos(req: Request, res: Response): Promise<void> {
        try {
            const { search, completed } = req.query;

            const searchParams: TodoSearchParams = {};
            if (typeof search === 'string') {
                // タイトルまたは説明文で検索
                searchParams.title = search;
            }
            if (completed !== undefined) {
                searchParams.completed = completed === 'true';
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
使用例
1. 全件取得: `/todos`
2. タイトルで検索: `/todos?search=買い物`
3. 完了済みのみ: `/todos?completed=true`
4. 未完了のみ: `/todos?completed=false`
5. 条件組み合わせ: `/todos?search=買い物&completed=true`

ルーティングを更新します。

```typescript
// src/controllers/todo.controller.test.ts
beforeEach(() => {
    // 既存のセットアップコード...

    // 検索エンドポイントを追加
    app.get('/todos', controller.getTodos.bind(controller));
    app.patch('/todos/:id', controller.updateTodo.bind(controller));
});
```
- クエリパラメータを使用した柔軟な検索
- 文字列の`completed`値を適切にブール値に変換
- 検索パラメータが指定されない場合は全件取得

## 5. 更新エンドポイントの実装

更新機能のテストを追加します

```typescript
// src/controllers/todo.controller.test.ts
describe('PATCH /todos/:id', () => {
    let todoId: string;

    beforeEach(async () => {
        const response = await request(app)
            .post('/todos')
            .send({ title: 'Original Title' });
        todoId = response.body.id;
    });

    test('updates todo with valid input', async () => {
        const response = await request(app)
            .patch(`/todos/${todoId}`)
            .send({
                title: 'Updated Title',
                completed: true
            });

        expect(response.status).toBe(200);
        expect(response.body).toMatchObject({
            title: 'Updated Title',
            completed: true
        });
    });

    test('returns 404 when todo not found', async () => {
        const response = await request(app)
            .patch('/todos/non-existent-id')
            .send({ title: 'Updated Title' });

        expect(response.status).toBe(404);
        expect(response.body).toEqual({
            errors: ['Todo not found']
        });
    });
});
```
- 共通で使用する `todoId` を外部スコープで定義し、すべてのテストケースで参照できるようにしています
- `toMatchObject` を使用して、部分的なオブジェクトマッチングを行っています
- `toEqual` を使用して、完全一致での比較を行っています


更新機能を実装します。

```typescript
// src/controllers/todo.controller.ts
export class TodoController {
    // 既存のコード...

    async updateTodo(req: Request, res: Response): Promise<void> {
        try {
            const todo = await this.service.updateTodo(
                req.params.id,
                req.body as UpdateTodoDTO
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
                res.status(500).json({ 
                    errors: ['Internal server error']
                });
            }
        }
    }
}
```
