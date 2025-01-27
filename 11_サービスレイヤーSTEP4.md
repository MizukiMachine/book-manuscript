前回までではTodoServiceの実装を行いました。
今回は、HTTPリクエストを受け付けて適切なレスポンスを返すコントローラーレイヤーをTDDで実装していきます。

## 1. 準備

まず、Express用の型定義を確認し、必要なパッケージを追加でインストールします

```bash
npm install express-validator
npm install --save-dev supertest @types/supertest
```

## 2. コントローラーの基本実装

まずはTodoコントローラーのテストを作成します

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
        test('creates a new todo with valid input', async () => {
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

### テストコードの解説
こちら、重要なことが多いので丁寧に解説していきます。

1. テスト環境のセットアップ
```typescript
let app: Express;
let controller: TodoController;
let service: TodoService;

beforeEach(() => {
    app = express();
    app.use(express.json());
    // ...
});
```
- テストを実行する前に、Webアプリケーション（Express）の新しい環境を準備します
- `express.json()` は、クライアントから送られてくるJSONデータを自動的に解析してくれる機能を追加します

2. システムの構造を組み立てる
```typescript
const repository = new TodoRepository();
service = new TodoService(repository);
controller = new TodoController(service);
```
- システムを3つの層に分けてます
  - Repository: データの保存と取得を担当（冷蔵庫やパントリー）
  - Service: ビジネスルールを管理（店長の方針）
  - Controller: ユーザーからのリクエストを受け付ける（ウェイター）
- それぞれの役割を明確に分けることで、拡張や仕様変更に強くなります

3. URLとプログラムの紐付け
```typescript
app.post('/todos', controller.createTodo.bind(controller));
```
- `/todos`というURLにアクセスしたときに、どの処理を実行するかを設定します
- `bind(controller)`は、プログラムが正しく動作するために必要な設定です
  - 「このメソッドは必ずこのクラスのものとして動作してね」という"固定"や"紐付け"

4. HTTPリクエストのテスト
```typescript
const response = await request(app)
    .post('/todos')
    .send({
        title: 'Test Todo',
        description: 'Test Description'
    });
```
- `supertest`というツールを使って、実際のWebブラウザからのリクエストをシミュレーションします
- これは、お店のオープン前に、実際の接客をシミュレーションして問題がないか確認するようなものです

5. 返ってきた結果の確認
```typescript
expect(response.status).toBe(201);
expect(response.body).toMatchObject({
    title: 'Test Todo',
    description: 'Test Description',
    completed: false
});
expect(response.body.id).toBeDefined();
```
- ステータスコード201は「新しいデータが作成されました」という意味です
- 返ってきたデータが、期待通りの内容になっているか確認
- `toMatchObject`は、必要な情報が全て含まれているか確認する方法です
- IDが付与されているか確認します（商品にタグが付いているかチェックするようなもの）

**このテストのポイント**
- 実際のユーザーがアプリを使う時と同じような流れでテストします
- プログラムの入り口（HTTP通信）から出口（データ保存）まで、全体の流れをテストします
- それぞれの部品（Controller, Service, Repository）が正しく連携できているか確認します
- エラーが起きないか、正しい結果が返ってくるか、細かくチェックします

このような統合テストを行うことで、実際にアプリケーションをリリースする前に、問題がないことを確認できます。


実装を追加します。

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

- エラーハンドリングを適切に実装
- 201 Createdステータスコードを使用
- エラー時は400 Bad RequestまたはInternalサーバーエラー
