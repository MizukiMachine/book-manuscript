# 第3章 リポジトリレイヤーの基礎実装

## 3.1 リポジトリレイヤーの役割と設計思想

「レイヤー」とは、アプリケーションを機能や役割ごとに層として分割したものを指します。  
リポジトリレイヤー（Repository Layer）は、データの永続化と取得を担当するレイヤーです。  
このレイヤーの主な責務は、

1. データの保存と取得
2. データの整合性の確保
3. データアクセスの抽象化

アプリケーション内の他のレイヤーは、データがどのように保存されているかを知る必要がありません。  
例えば、現時点ではメモリ内にデータを保持し、後々データベースに変更するといった実装の変更が可能です。

## 3.2 基本的な型定義

まず、アプリケーション全体で使用する基本的な型を定義します。

```typescript
// src/types.ts
export interface Todo {
    id: string;          // Todoの一意識別子
    title: string;       // Todoのタイトル
    description?: string;// 詳細説明（オプショナル）
    completed: boolean;  // 完了状態
    createdAt: Date;    // 作成日時
    updatedAt: Date;    // 更新日時
}

// 新規Todo作成時に使用するDTO
export interface CreateTodoDTO {
    title: string;       // 必須項目
    description?: string;// オプショナル
}

// Todo更新時に使用するDTO
export interface UpdateTodoDTO {
    title?: string;      // 全てオプショナル
    description?: string;// 更新したいフィールドのみ
    completed?: boolean; // 指定可能
}
```

**型定義のポイント：**

1. **DTOパターンの採用**  
   DTOは「Data Transfer Object」の略で、レイヤー間でデータを受け渡すための専用のオブジェクト形式です。  
   これにより
   - 入力データと永続化データを明確に分離
   - 必要な項目のみを含む最小限の形式を定義
   - 型の安全性を確保

2. **オプショナルプロパティの使用**  
   TypeScriptでは`?`をプロパティ名の後につけることで、そのプロパティが省略可能であることを示します。
   - `description?: string` は、説明文が省略可能
   - UpdateDTOでは全フィールドをオプショナルに設定し、部分的な更新を可能に

## 3.3 リポジトリの初期実装

### 1. テストコードの作成

TDDの「Red」フェーズとして、まずテストを書きます。Jestというテストフレームワークを使用して、以下のようにテストを記述します。

```typescript
// src/repositories/todo.repository.test.ts
import { TodoRepository } from './todo.repository';
import { CreateTodoDTO, Todo } from '../types';

describe('TodoRepository', () => {
    let repository: TodoRepository;

    beforeEach(() => {
        repository = new TodoRepository();
    });

    describe('create', () => {
        it('creates a new todo with required fields', async () => {
            const dto: CreateTodoDTO = {
                title: 'Test Todo'
            };

            const todo = await repository.create(dto);

            expect(todo.id).toBeDefined();
            expect(todo.title).toBe(dto.title);
            expect(todo.completed).toBe(false);
            expect(todo.createdAt).toBeInstanceOf(Date);
            expect(todo.updatedAt).toBeInstanceOf(Date);
        });
    });
});
```

**テストコードの基本要素の解説：**

1. **テストの構造化**
   - `describe`: テストのグループ化を行います。ここでは`TodoRepository`という大きなグループと、その中の`create`メソッドのグループを作成しています。
   - `it`: 個別のテストケースを記述します。テストの内容を英文で説明する形式で記述します。

2. **beforeEach**
   ```typescript
   beforeEach(() => {
       repository = new TodoRepository();
   });
   ```
   各テストケースの実行前に毎回実行される処理です。ここでは、各テストで新しいリポジトリインスタンスを使用することで、テスト間の独立性を確保しています。

3. **非同期テスト**
   ```typescript
   async () => {
       const todo = await repository.create(dto);
   }
   ```
   - `async/await`を使用することで、非同期処理（データベースアクセスなど）を含むコードをテストできます。
   - テスト関数を`async`とし、非同期処理を`await`で待機します。

4. **アサーション（検証）**
   ```typescript
   expect(todo.id).toBeDefined();
   expect(todo.title).toBe(dto.title);
   ```
   `expect`関数を使って、テストの期待する結果を検証します：
   - `toBeDefined()`: 値が定義されていることを確認
   - `toBe()`: 厳密な等価性を検証
   - `toBeInstanceOf()`: オブジェクトの型を検証

これらのテストコードの基本要素は、以降の章でも継続して使用していきます。

### 2. リポジトリの実装
テストを実行すると、以下のエラーが発生します。

```bash
FAIL  src/repositories/todo.repository.test.ts
  ● Test suite failed to run
    Cannot find module './todo.repository' from 'src/repositories/todo.repository.test.ts'
```
これは当然の結果です。
まだ`todo.repository.ts`ファイルを作成していないためエラーになります。  
では、このエラーを解消するため、TodoRepositoryの実装を行います。

テストが失敗することを確認した後、「Green」フェーズとして実装を行います。

```typescript
// src/repositories/todo.repository.ts
import { Todo, CreateTodoDTO } from '../types';
import { v4 as uuidv4 } from 'uuid';

export class TodoRepository {
    private todos: Map<string, Todo> = new Map();

    async create(dto: CreateTodoDTO): Promise<Todo> {
        const todo: Todo = {
            id: uuidv4(),          // ユニークなIDを生成
            title: dto.title,      // DTOからタイトルを取得
            description: dto.description,
            completed: false,      // 新規作成時は未完了
            createdAt: new Date(), // 現在の日時
            updatedAt: new Date()  // 現在の日時
        };

        this.todos.set(todo.id, todo);
        return todo;
    }
}
```

**実装のポイント：**

1. **データ構造の選択**
   `Map`を使用する理由：
   - キーと値の関係を明確に管理可能
   - IDによる高速なデータアクセス
   - 削除や更新が容易

2. **非同期処理の採用**  
   将来的なデータベース統合を見据えて、最初から非同期処理として実装：
   - `async/await`を使用
   - `Promise`を返却

## 3.4 パッケージのインストール

UUIDの生成に必要なパッケージをインストールします。

```bash
npm install uuid
npm install --save-dev @types/uuid
```

`uuid`パッケージは、一意の識別子を生成するためのライブラリです。  
アプリケーション内で重複のないIDを確実に生成できます。

## 3.5 実装のポイント解説

### 1. インメモリデータストア
現段階では、データベースの複雑さを避けるため、メモリ内でデータを管理します。
- 開発初期の迅速な実装が可能
- テストが容易
- 後でデータベースに置き換え可能

### 2. DTOパターンの活用
入力データと永続化データを分離することで
- データの受け渡しを型安全に
- 必要なデータのみを扱う
- バリデーションやデータ変換の責務を明確化

この章では、TDDの基本的なサイクルに従いながら、TodoRepositoryの基本機能を実装しました。  
次章では、より高度な機能とバリデーションの実装に進みます。
