今回は、ビジネスロジックを担当するサービスレイヤーをTDDで実装していきます。

## 1. TodoServiceの基本実装

まず、TodoServiceのテストを作成します

```typescript
// src/services/todo.service.test.ts
import { TodoService } from './todo.service';
import { TodoRepository } from '../repositories/todo.repository';
import { CreateTodoDTO } from '../types';

describe('TodoService', () => {
    let service: TodoService;
    let repository: TodoRepository;

    beforeEach(() => {
        repository = new TodoRepository();
        service = new TodoService(repository);
    });

    describe('createTodo', () => {
        test('creates a todo with valid input', async () => {
            const dto: CreateTodoDTO = {
                title: 'Test Todo',
                description: 'Test Description'
            };

            const todo = await service.createTodo(dto);
            
            expect(todo.title).toBe(dto.title);
            expect(todo.description).toBe(dto.description);
            expect(todo.completed).toBe(false);
        });
    });
});
```

テスト結果
```
FAIL  src/services/todo.service.test.ts
  ● Test suite failed to run
    Cannot find module './todo.service' from 'src/services/todo.service.test.ts'
```

TodoServiceの基本実装を作成します

```typescript
// src/services/todo.service.ts
import { TodoRepository } from '../repositories/todo.repository';
import { CreateTodoDTO, UpdateTodoDTO, Todo } from '../types';

export class TodoService {
    constructor(private repository: TodoRepository) {}

    async createTodo(dto: CreateTodoDTO): Promise<Todo> {
        return this.repository.create(dto);
    }
}
```

テスト結果
```
PASS  src/services/todo.service.test.ts
  TodoService
    createTodo
      ✓ creates a todo with valid input (3ms)
```

## 2. バリデーションとビジネスルールの追加

まず、必要なパッケージをインストールします

```bash
npm install sanitize-html
npm install --save-dev @types/sanitize-html
```

サービスレイヤーでより厳密なバリデーションとビジネスルールを実装します。
テストを追加します

```typescript
// src/services/todo.service.test.ts
describe('createTodo', () => {
    // 既存のテスト...

    test('throws error for title exceeding maximum length', async () => {
        const dto: CreateTodoDTO = {
            title: 'a'.repeat(101),  // 101文字
            description: 'Test Description'
        };

        await expect(service.createTodo(dto))
            .rejects
            .toThrow('Title cannot exceed 100 characters');
    });

    test('throws error for description exceeding maximum length', async () => {
        const dto: CreateTodoDTO = {
            title: 'Test Todo',
            description: 'a'.repeat(501)  // 501文字
        };

        await expect(service.createTodo(dto))
            .rejects
            .toThrow('Description cannot exceed 500 characters');
    });

    test('sanitizes malicious content in title and description', async () => {
        const dto: CreateTodoDTO = {
            title: '<script>alert("xss")</script>Test Todo<p onclick="alert()">',
            description: '<b onclick="alert()">Description</b><img src="x" onerror="alert()">'
        };

        const todo = await service.createTodo(dto);
        
        expect(todo.title).toBe('Test Todo');
        expect(todo.description).toBe('Description');
    });

    test('trims whitespace from title and description', async () => {
        const dto: CreateTodoDTO = {
            title: '  Test Todo  ',
            description: '  Description  '
        };

        const todo = await service.createTodo(dto);
        
        expect(todo.title).toBe('Test Todo');
        expect(todo.description).toBe('Description');
    });
});
```

実装を更新します

```typescript
// src/services/todo.service.ts
import { TodoRepository } from '../repositories/todo.repository';
import { CreateTodoDTO, Todo } from '../types';
import sanitizeHtml from 'sanitize-html';

export class TodoService {
    private readonly MAX_TITLE_LENGTH = 100;
    private readonly MAX_DESCRIPTION_LENGTH = 500;

    constructor(private repository: TodoRepository) {}

    async createTodo(dto: CreateTodoDTO): Promise<Todo> {
        // 入力値の検証とサニタイズ
        const sanitizedTitle = this.sanitizeText(dto.title);
        const sanitizedDescription = dto.description 
            ? this.sanitizeText(dto.description)
            : undefined;

        // タイトルの長さチェック
        if (sanitizedTitle.length > this.MAX_TITLE_LENGTH) {
            throw new Error('Title cannot exceed 100 characters');
        }

        // 説明文の長さチェック
        if (sanitizedDescription && 
            sanitizedDescription.length > this.MAX_DESCRIPTION_LENGTH) {
            throw new Error('Description cannot exceed 500 characters');
        }

        return this.repository.create({
            title: sanitizedTitle,
            description: sanitizedDescription
        });
    }

    private sanitizeText(text: string): string {
        return sanitizeHtml(text, {
            allowedTags: [],      // HTMLタグを全て除去
            allowedAttributes: {}, // 属性を全て除去
        }).trim();
    }
}
```

実装のポイント
- `sanitize-html`パッケージを使用して安全なサニタイズを実現
- 全てのHTMLタグと属性を除去
- テキストのトリミングも同時に実施
- 未定義の説明文は`undefined`のまま維持

テスト結果
```
PASS  src/services/todo.service.test.ts
  TodoService
    createTodo
      ✓ creates a todo with valid input (3ms)
      ✓ throws error for title exceeding maximum length (1ms)
      ✓ throws error for description exceeding maximum length (1ms)
      ✓ sanitizes malicious content in title and description (2ms)
      ✓ trims whitespace from title and description (1ms)
```
