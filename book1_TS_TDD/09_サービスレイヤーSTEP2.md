# 第9章 TodoServiceの更新機能実装

## 9.1 更新機能の設計思想

サービスレイヤーでの更新機能は、単なるデータの更新だけでなく、より高度なビジネスルールを適用します。  
主な要件は以下の通りです。

1. **ビジネスルールの適用**
   - 完了済みTodoの更新禁止
   - 文字数制限の適用
   - データの整合性確保

2. **セキュリティ対策**
   - 入力値のサニタイズ処理
   - XSS対策の実施
   - 不正な更新の防止

3. **バリデーションの統合**
   - リポジトリレイヤーのバリデーション
   - サービスレイヤー独自のルール
   - エラーメッセージの統一

## 9.2 更新機能のテスト実装

```typescript
// src/services/todo.service.test.ts
import { wait } from '../test-utils/helpers';
describe('TodoService', () => {
    let service: TodoService;
    let repository: TodoRepository;

    beforeEach(async () => {
        repository = new TodoRepository();
        service = new TodoService(repository);

        // テストデータを準備
        await service.createTodo({
            title: 'Original Title',
            description: 'Original Description'
        });
    });

    describe('updateTodo', () => {
        it('updates todo with valid input', async () => {
            // まず新しいTodoを作成
            const created = await service.createTodo({
                title: 'Original Title',
                description: 'Original Description'
            });

            await wait();

            const updateDto: UpdateTodoDTO = {
                title: 'Updated Title',
                description: 'Updated Description',
                completed: true
            };

            const updated = await service.updateTodo(created.id, updateDto);

            // 更新結果の検証
            expect(updated.title).toBe(updateDto.title);
            expect(updated.description).toBe(updateDto.description);
            expect(updated.completed).toBe(true);
            expect(updated.updatedAt.getTime()).toBeGreaterThan(created.updatedAt.getTime());
        });

        it('sanitizes and validates updated fields', async () => {
            const created = await service.createTodo({
                title: 'Original Title'
            });

            const updateDto: UpdateTodoDTO = {
                title: '<script>alert("xss")</script>Updated Title  ',
                description: '  <b>New</b> Description  '
            };

            const updated = await service.updateTodo(created.id, updateDto);

            expect(updated.title).toBe('Updated Title');
            expect(updated.description).toBe('New Description');
        });

        it('prevents updating completed todo', async () => {
            // 完了済みのTodoを作成
            const todo = await service.createTodo({ title: 'Test Todo' });
            await service.updateTodo(todo.id, { completed: true });

            // 完了済みTodoの更新を試みる
            await expect(
                service.updateTodo(todo.id, { title: 'New Title' })
            ).rejects.toThrow('Cannot update completed todo');
        });

        it('enforces maximum length constraints', async () => {
            const todo = await service.createTodo({ title: 'Test Todo' });

            await expect(
                service.updateTodo(todo.id, { 
                    title: 'a'.repeat(101) 
                })
            ).rejects.toThrow('Title cannot exceed 100 characters');

            await expect(
                service.updateTodo(todo.id, { 
                    description: 'a'.repeat(501) 
                })
            ).rejects.toThrow('Description cannot exceed 500 characters');
        });
    });
});
```

**更新機能テストのポイント：**

1. **ビジネスルールの検証**
   - 完了済みTodoの更新禁止
   - 文字数制限の確認
   - 更新日時の変更確認

2. **セキュリティ機能の確認**
   - HTMLタグの除去
   - 空白文字の正規化
   - サニタイズ処理の検証

3. **エラーケースの網羅**
   - バリデーションエラー
   - ビジネスルール違反
   - 明確なエラーメッセージ

## 9.3 更新機能の実装

```typescript
// src/services/todo.service.ts
export class TodoService {
    constructor(private repository: TodoRepository) {}

    async updateTodo(id: string, dto: UpdateTodoDTO): Promise<Todo> {
        // 既存のTodoを取得
        const existingTodo = await this.repository.findById(id);
        if (!existingTodo) {
            throw new Error('Todo not found');
        }

        // 完了済みTodoの更新を防止
        if (existingTodo.completed) {
            throw new Error('Cannot update completed todo');
        }

        // 更新用のDTOを準備
        const updateData: UpdateTodoDTO = {};

        // タイトルの更新処理
        if (dto.title !== undefined) {
            const sanitizedTitle = this.sanitizeText(dto.title);
            if (sanitizedTitle.length > 100) {
                throw new Error('Title cannot exceed 100 characters');
            }
            updateData.title = sanitizedTitle;
        }

        // 説明文の更新処理
        if (dto.description !== undefined) {
            const sanitizedDescription = this.sanitizeText(dto.description);
            if (sanitizedDescription.length > 500) {
                throw new Error('Description cannot exceed 500 characters');
            }
            updateData.description = sanitizedDescription;
        }

        // 完了状態の更新
        if (dto.completed !== undefined) {
            updateData.completed = dto.completed;
        }

        // リポジトリを使用してTodoを更新
        return this.repository.update(id, updateData);
    }

    private sanitizeText(text: string): string {
        return sanitizeHtml(text, {
            allowedTags: [],
            allowedAttributes: {},
        }).trim();
    }
}
```

**実装のポイント：**

1. **段階的なバリデーション**
   - 存在チェック
   - 完了状態の確認
   - フィールドごとの検証

2. **部分更新の実現**
   ```typescript
   if (dto.title !== undefined) {
       const sanitizedTitle = this.sanitizeText(dto.title);
       // バリデーションと更新処理
   }
   ```
   - 指定されたフィールドのみを更新
   - 各フィールドの独立した処理
   - 型安全な実装

3. **サニタイズ処理の共通化**
   ```typescript
   private sanitizeText(text: string): string {
       return sanitizeHtml(text, {
           allowedTags: [],
           allowedAttributes: {},
       }).trim();
   }
   ```
   - セキュリティ処理の一元管理
   - 処理の再利用性確保
   - 一貫した動作の保証

## 9.4 リポジトリレイヤーとの違いの確認

### 1. バリデーションの違い
- リポジトリ：基本的なデータ整合性のチェック
- サービス：より詳細なビジネスルールの適用
  - 文字数制限
  - 完了状態による制御
  - セキュリティ対策

### 2. エラー処理の違い
- リポジトリ：データ操作に関する基本的なエラー
- サービス：ビジネスルールに関連する具体的なエラー
  - 完了済みTodoの更新禁止
  - 文字数制限超過
  - 不正なデータの検出

### 3. 責務の違い
- リポジトリ：データの永続化と取得
- サービス：ビジネスロジックとバリデーション
  - セキュリティ対策
  - データの正規化
  - 業務ルールの適用

## 9.5 テスト実行結果

```bash
PASS  src/services/todo.service.test.ts
  TodoService
    updateTodo
      ✓ updates todo with valid input (3ms)
      ✓ sanitizes and validates updated fields (1ms)
      ✓ prevents updating completed todo (2ms)
      ✓ enforces maximum length constraints (2ms)
```

この章では、TodoServiceの更新機能を実装し、ビジネスルールやセキュリティ対策を組み込みました。  
次章では、検索機能の実装に進みます。
