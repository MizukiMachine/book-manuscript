# 第12章 コントローラーレイヤーの検索エンドポイント実装

## 12.1 検索エンドポイントの設計思想

APIの検索機能を実装する際、単にデータを返すだけでなく、使いやすく柔軟な検索インターフェースを提供することが重要です。
ここでは、RESTful APIのベストプラクティスに基づいて、以下の3つの観点から検索エンドポイントの設計を考えていきます。

1. **クエリパラメータの設計**
   - URLパラメータの適切な利用（例：`/todos?search=keyword&completed=true`）
   - 検索条件の柔軟な指定（複数の条件の組み合わせ）
   - パラメータの型安全な処理（TypeScriptの型システムの活用）

2. **レスポンスの設計**
   - 一貫性のある結果形式（JSON形式での返却）
   - 検索結果の適切な表現（配列形式でのTodo一覧）
   - エラー時の明確な応答（ステータスコードとエラーメッセージ）

3. **バリデーションの考慮**
   - パラメータの形式確認（型チェック、値の範囲確認）
   - 不正な値の検出（無効な完了状態値など）
   - エラーメッセージの提供（ユーザーフレンドリーな説明）

## 12.2 検索エンドポイントのテスト

検索機能の実装を始める前に、まずテストを書いていきます。
検索機能では、複数の条件での検索や、エラーケースの処理など、様々なシナリオをテストする必要があります。
以下のコードでは、基本的な検索機能から、エラー処理まで、網羅的にテストケースを実装していきます。

```typescript
// src/controllers/todo.controller.test.ts
describe('TodoController', () => {
    let app: Express;
    let controller: TodoController;
    let service: TodoService;

    beforeEach(async () => {
        // Expressアプリケーションの設定
        app = express();
        app.use(express.json());
        
        // 依存関係の設定
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

        // 一つのTodoを完了状態に更新
        await request(app)
            .patch(`/todos/${todo3.body.id}`)
            .send({ completed: true });
    });
```

```typescript
// src/controllers/todo.controller.test.ts
    describe('POST /todos', () => {
    // 既存のテスト...

    // 検索機能のテストケース
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
**コードのポイント解説**
1. **テスト環境のセットアップ**
  - `beforeEach` で各テストケース実行前に環境を初期化
  - Expressアプリケーションの設定
  - 依存関係（repository, service）の注入

2. **ルーティングの設定**
  - GET: 検索エンドポイント（バリデーション付き）
  - POST: 作成エンドポイント（テストデータ用）
  - PATCH: 更新エンドポイント（完了状態の更新用）

3. **テストデータの準備**
  - 複数のTodoを作成（検索結果の多様性確保）
  - 異なる状態のデータ（完了/未完了）
  - 検索キーワードに対応するタイトル設定

《テスト実行結果》
```bash
 FAIL  src/controllers/todo.controller.test.ts
 ● Test suite failed to run
    src/controllers/todo.controller.test.ts:23:38 - error TS2339: Property 'getTodos' does not exist on type 'TodoController'.
```

## 12.3 クエリパラメータのバリデーション

APIの安全性と信頼性を確保するために、クエリパラメータのバリデーションは非常に重要です。
express-validatorを使用して、このような観点でバリデーションを実装していきます。

- パラメータの型の確認（文字列、真偽値など）
- 不正な値の検出と適切なエラーメッセージの提供
- 入力値の正規化（トリミングなど）

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

**コードのポイント解説**
1. **完了状態の検証**
   - optional(): パラメータの任意設定
  - isBoolean(): 真偽値としての妥当性確認
  - toBoolean(): 文字列から真偽値への自動変換

2. **検索キーワードの検証**
   - isString(): 文字列としての妥当性確認
   - trim(): 前後の空白除去による正規化


## 12.4 検索エンドポイントの実装

バリデーションの準備ができたところで、実際の検索機能を実装していきます。
この実装では、以下の点に特にポイントです。

- TypeScriptの型安全性を活用した実装
- エラーハンドリングの適切な実装
- クエリパラメータの適切な解析と変換

また、テストデータ作成に必要な更新機能も併せて実装します。

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

    // 完了状態更新用のメソッド
    async updateTodo(req: Request, res: Response): Promise<void> {
        try {
            const todo = await this.service.updateTodo(
                req.params.id,
                req.body
            );
            res.json(todo);
        } catch (error) {
            if (error instanceof Error) {
                res.status(400).json({ errors: [error.message] });
            } else {
                res.status(500).json({ errors: ['Internal server error'] });
            }
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

《テスト実行結果》
```bash
PASS  src/controllers/todo.controller.test.ts
  TodoController
    GET /todos
      ✓ returns all todos when no query parameters (42ms)
      ✓ filters todos by search term (38ms)
      ✓ filters todos by completion status (35ms)
      ✓ handles invalid completed parameter (12ms)
```

**コードのポイント解説**
1. **入力値の検証と変換**
   ```typescript
   const { search, completed } = req.query;
   if (completed !== undefined) {
       searchParams.completed = String(completed).toLowerCase() === 'true';
   }
   ```
   - クエリパラメータの型安全な取得
   - 文字列から真偽値への適切な変換
     - `completed` は string 以外も入ってくる可能性があるため

2. **エラーハンドリング**
   ```typescript
   if (!errors.isEmpty()) {
       res.status(400).json({ 
           errors: errors.array().map(err => err.msg)
       });
       return;
   }
   ```
   - バリデーションエラーの適切な処理
     - 400 Bad Request
   - ユーザーフレンドリーなエラーメッセージ
     - 500 Internal Server Error

3. **更新機能の実装**
   ```typescript
   async updateTodo(req: Request, res: Response): Promise<void> {
   ```
   - テストデータ準備のための必須機能
   - エラーハンドリングの実装


## 12.5 RESTful APIのベストプラクティス

検索APIを実装する際はRESTfulなAPI設計の原則に従うことで、直感的で使いやすいインターフェースを提供できます。
今回の実装では、以下のベストプラクティスを意識しています。

1. **URLの設計**
   - GETメソッドの使用
     - 検索・取得系の操作はGETメソッドで統一
     - キャッシュの活用が可能
   - クエリパラメータによる条件指定
     - `?search=keyword&completed=true` の形式で条件を指定
     - 複数の検索条件を組み合わせ可能
   - 意図の明確な命名
     - `/todos`: リソースを明確に表現
     - `search`, `completed`: パラメータ名で機能を明示

2. **レスポンス形式**
   - 一貫性のあるJSON形式
     - すべてのレスポンスで統一されたJSON構造を使用
     - フロントエンド側での処理が容易
   - 配列形式での結果返却
     - 検索結果は常に配列形式で返却
     - 結果が0件の場合も空配列を返却
   - エラー情報の統一的な提供
     - エラーメッセージは `errors` 配列で返却
     - 人間が理解しやすいエラーメッセージを提供

3. **HTTPステータスコード**
   - 200: 検索成功
     - 検索結果が0件の場合も200を返却
   - 400: パラメータ不正
     - バリデーションエラーの場合
     - 不正なパラメータ値の場合
   - 500: サーバーエラー
     - 予期せぬエラーが発生した場合

## 12.6 サービスレイヤーとの連携

レイヤードアーキテクチャの利点を最大限に活かすため、コントローラーとサービスの間で明確な責務の分離を行っています。
この分離によりコードの保守性が向上し、将来的な機能追加や変更が容易になります。

1. **コントローラーの責務**
   - HTTPパラメータの解析
     - クエリパラメータの取得と型変換
     - リクエストボディのパース
   - 基本的なバリデーション
     - パラメータの存在チェック
     - 形式チェック
   - レスポンスの形成
     - ステータスコードの設定
     - JSONレスポンスの構築
     - エラーメッセージのフォーマット

2. **サービスへの委譲**
   - 実際の検索処理
     - データベースへのクエリ構築
     - 検索条件の適用
   - ビジネスロジックの適用
     - 検索結果のフィルタリング
     - アクセス権限のチェック
   - データの加工
     - 必要なフィールドの選択
     - レスポンス用のデータ整形


ここまで分離,分離と意識してきましたが、ここまで行うことで以下メリットを実現できているわけです。
- テストの容易性：各レイヤーを独立してテスト可能
- コードの再利用：ビジネスロジックを他のエンドポイントでも利用可能
- 保守性の向上：各レイヤーの変更が他のレイヤーに影響を与えにくい

---
この章では、検索機能のエンドポイントを実装し、URLパラメータを使用した柔軟な検索機能を実現しました。  
次章では、更新エンドポイントの実装に進みます。
