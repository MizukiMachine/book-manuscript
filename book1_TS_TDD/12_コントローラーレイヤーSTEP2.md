# 第12章 コントローラーレイヤーの検索エンドポイント実装

## 12.1 検索エンドポイントの設計思想

RESTful APIにおける検索エンドポイントは、以下の要件を満たす必要があります。

1. **クエリパラメータの設計**
   - URLパラメータの適切な利用
   - 検索条件の柔軟な指定
   - パラメータの型安全な処理

2. **レスポンスの設計**
   - 一貫性のある結果形式
   - 検索結果の適切な表現
   - エラー時の明確な応答

3. **バリデーションの考慮**
   - パラメータの形式確認
   - 不正な値の検出
   - エラーメッセージの提供

## 12.2 検索エンドポイントのテスト

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

        // 完了状態の更新
        await request(app)
            .patch(`/todos/${todo3.body.id}`)
            .send({ completed: true });
    });
```

```typescript
// src/controllers/todo.controller.test.ts
    describe('POST /todos', () => {
    // 既存のテスト...
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

**検索エンドポイントテストのポイント：**

1. **テストデータの準備**
   - 複数のサンプルデータを作成
   - 異なる状態のデータを用意
   - 検索条件に合致するデータの確保

2. **検索シナリオの網羅**
   - パラメータなしの全件取得
   - キーワードによる検索
   - 完了状態でのフィルタリング

3. **エラー処理の確認**
   - 不正なパラメータ値の処理
   - 適切なエラーレスポンス
   - HTTPステータスコードの確認

テスト実行結果：
```bash
 FAIL  src/controllers/todo.controller.test.ts
 ● Test suite failed to run
    src/controllers/todo.controller.test.ts:23:38 - error TS2339: Property 'getTodos' does not exist on type 'TodoController'.
```

## 12.3 クエリパラメータのバリデーション

検索用のクエリパラメータに対するバリデーションを実装します。

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

**クエリパラメータバリデーションのポイント：**

1. **パラメータの型変換**
   - `.isBoolean()`: 真偽値としての妥当性を確認
   - `.toBoolean()`: 文字列から真偽値への変換
   - `.trim()`: 文字列の正規化

2. **オプショナルパラメータ**
   - `.optional()`: 必須でないパラメータの指定
   - 存在する場合のみ検証を実行

## 12.4 検索エンドポイントの実装

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
                // 型を確実に string型に変換してから比較
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
テストのセットアップを更新します。（ `getTodosValidation` を追加）

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
**実装のポイント：**

1. **クエリパラメータの抽出**
   ```typescript
   const { search, completed } = req.query;
   ```
   - URLパラメータからの値の取得
   - 型安全な処理

2. **検索パラメータの構築**
   ```typescript
   if (typeof search === 'string') {
       searchParams.title = search;
   }
   ```
   - 型チェックの実施
   - 適切な形式への変換

3. **エラーハンドリング**
   - バリデーションエラー: 400 Bad Request
   - サーバーエラー: 500 Internal Server Error

## 12.5 RESTful APIのベストプラクティス

検索エンドポイントの実装では、以下のRESTful APIデザインの原則に従っています。

1. **URLの設計**
   - GETメソッドの使用
   - クエリパラメータによる条件指定
   - 意図の明確な命名

2. **レスポンス形式**
   - 一貫性のあるJSON形式
   - 配列形式での結果返却
   - エラー情報の統一的な提供

3. **HTTPステータスコード**
   - 200: 検索成功
   - 400: パラメータ不正
   - 500: サーバーエラー

## 12.6 サービスレイヤーとの連携

上記はコントローラーレイヤーとサービスレイヤーの明確な役割分担になっています。

1. **コントローラーの責務**
   - HTTPパラメータの解析
   - 基本的なバリデーション
   - レスポンスの形成

2. **サービスへの委譲**
   - 実際の検索処理
   - ビジネスロジックの適用
   - データの加工

この章では、検索機能のエンドポイントを実装し、URLパラメータを使用した柔軟な検索機能を実現しました。  
次章では、更新エンドポイントの実装に進みます。
