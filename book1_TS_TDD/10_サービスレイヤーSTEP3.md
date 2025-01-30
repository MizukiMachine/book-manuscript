# 第10章 TodoServiceの検索機能実装

## 10.1 検索機能の設計思想

サービスレイヤーでの検索機能は、単純なデータ取得以上の役割を担います。  
以下のような要件を満たす必要があります：

1. **検索条件の整理**
   - 複数の検索条件の組み合わせ
   - 検索パラメータの正規化
   - クエリの最適化

2. **検索結果の加工**
   - 必要なデータの抽出
   - データの整形
   - セキュリティ上の考慮

3. **ユーザビリティの向上**
   - 大文字小文字を区別しない検索
   - あいまい検索への対応
   - 使いやすい検索インターフェース

## 10.2 検索機能のテスト実装

```typescript
// src/services/todo.service.test.ts
describe('TodoService', () => {
    let service: TodoService;
    let repository: TodoRepository;

    beforeEach(async () => {
        repository = new TodoRepository();
        service = new TodoService(repository);

        // テストデータを準備
        await service.createTodo({ title: 'Shopping', description: 'Buy groceries' });
        await service.createTodo({ title: 'Coding', description: 'Implement search' });
        await service.createTodo({ title: 'Exercise', description: 'Go to gym' });
        const completedTodo = await service.createTodo({ 
            title: 'Reading', 
            description: 'Read book' 
        });
        await service.updateTodo(completedTodo.id, { completed: true });
    });

    describe('findTodos', () => {
        it('finds todos by title search', async () => {
            const todos = await service.findTodos({ title: 'ing' });
            
            expect(todos).toHaveLength(3);
            expect(todos.map(t => t.title)).toEqual(
                expect.arrayContaining(['Shopping', 'Coding', 'Reading'])
            );
        });

        it('finds todos by completion status', async () => {
            const completed = await service.findTodos({ completed: true });
            expect(completed).toHaveLength(1);
            expect(completed[0].title).toBe('Reading');

            const incomplete = await service.findTodos({ completed: false });
            expect(incomplete).toHaveLength(3);
        });

        it('combines search criteria', async () => {
            const todos = await service.findTodos({
                title: 'ing',
                completed: false
            });

            expect(todos).toHaveLength(2);
            expect(todos.map(t => t.title)).toEqual(
                expect.arrayContaining(['Shopping', 'Coding'])
            );
        });

        it('returns empty array when no matches found', async () => {
            const todos = await service.findTodos({ 
                title: 'nonexistent' 
            });
            expect(todos).toHaveLength(0);
        });

        it('performs case-insensitive search', async () => {
            const upperCase = await service.findTodos({ title: 'SHOPPING' });
            const lowerCase = await service.findTodos({ title: 'shopping' });
            
            expect(upperCase).toHaveLength(1);
            expect(lowerCase).toHaveLength(1);
            expect(upperCase[0]).toEqual(lowerCase[0]);
        });
    });
});
```

**検索機能テストのポイント：**

1. **テストデータの準備**
   - 複数のテストケースに対応するデータ
   - 完了状態の異なるデータ
   - 検索条件に合致するデータ

2. **検索シナリオの網羅**
   - タイトルでの部分一致検索
   - 完了状態による検索
   - 複数条件の組み合わせ

3. **エッジケースの確認**
   - データが見つからない場合
   - 大文字小文字の区別なし
   - 空の結果セット

## 10.3 検索パラメータの型定義

検索用のインターフェースを定義します。

```typescript
// src/types.ts に追加
export interface TodoSearchParams {
    title?: string;      // タイトルでの部分一致検索
    description?: string;// 説明文での部分一致検索
    completed?: boolean; // 完了状態での検索
}
```

**型定義のポイント：**
- オプショナルなパラメータ
- 型安全な検索条件
- 拡張性を考慮した設計

## 10.4 検索機能の実装

```typescript
// src/services/todo.service.ts
export class TodoService {
    constructor(private repository: TodoRepository) {}

    async findTodos(params: TodoSearchParams): Promise<Todo[]> {
        // まず全てのTodoを取得
        const allTodos = await this.repository.findAll();

        // 検索条件に基づいてフィルタリング
        return allTodos.filter(todo => {
            // タイトルで検索
            if (params.title && !todo.title.toLowerCase()
                .includes(params.title.toLowerCase())) {
                return false;
            }

            // 説明文で検索
            if (params.description && !todo.description?.toLowerCase()
                .includes(params.description.toLowerCase())) {
                return false;
            }

            // 完了状態で検索
            if (params.completed !== undefined && 
                todo.completed !== params.completed) {
                return false;
            }

            return true;
        });
    }
}
```

**実装のポイント：**

1. **大文字小文字を区別しない検索**
   ```typescript
   todo.title.toLowerCase().includes(params.title.toLowerCase())
   ```
   - 検索文字列の正規化
   - ユーザーフレンドリーな検索
   - 柔軟な検索マッチング

2. **オプショナルチェーンの活用**
   ```typescript
   todo.description?.toLowerCase()
   ```
   - 説明文が未設定の場合の安全な処理
   - TypeScriptの型安全性の活用
   - エラー防止の実装

3. **複数条件の組み合わせ**
   ```typescript
   if (params.completed !== undefined && 
       todo.completed !== params.completed) {
       return false;
   }
   ```
   - 条件の論理的な組み合わせ(`AND`条件)
   - `undefined`の適切な処理
   - 柔軟な検索条件の実現

## 10.5 リポジトリレイヤーとの違いの確認

### 1. 検索条件の処理
- リポジトリ：基本的なデータアクセス
- サービス：より高度な検索ロジック
  - 大文字小文字の無視
  - 部分一致検索
  - 複数条件の組み合わせ

### 2. 検索結果の加工
- リポジトリ：生データの取得
- サービス：検索用のデータ加工
  - 条件に応じたフィルタリング
  - データの整形
  - 必要な情報の抽出

### 3. ユーザビリティの考慮
- リポジトリ：データの永続化に注力
- サービス：使いやすさの向上
  - より柔軟な検索条件
  - 直感的な検索結果
  - エラーハンドリング

テスト実行結果：

```bash
PASS  src/services/todo.service.test.ts
  TodoService
    findTodos
      ✓ finds todos by title search (2ms)
      ✓ finds todos by completion status (1ms)
      ✓ combines search criteria (1ms)
      ✓ returns empty array when no matches found (1ms)
      ✓ performs case-insensitive search (1ms)
```

この章では、TodoServiceの検索機能を実装し、ユーザーフレンドリーな検索機能を実現しました。  
次章では、コントローラーレイヤーの実装に進みます。
