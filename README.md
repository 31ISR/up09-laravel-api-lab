# REST API на Laravel — система поддержки (тикеты)

## Шаг 1. Создание проекта

```bash
composer create-project laravel/laravel tickets-api
cd tickets-api
```

Откройте `.env`. Замените блок настроек URL:

```env
APP_URL=ссылка на 8000 порт
```

Инициализируйте и проверьте запуск:

```bash
composer run setup
composer run dev
```

Должны увидеть `Server running on [http://127.0.0.1:8000]`. Остановите сервер (`Ctrl+C`).

---

## Шаг 2. Установка Sanctum

```bash
php artisan install:api
```

Команда устанавливает `laravel/sanctum`, создаёт миграцию таблицы токенов и файл `routes/api.php`.

Откройте `app/Models/User.php`. Добавьте импорт и трейт:

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, Notifiable;

    // остальной код не меняйте
}
```

---

## Шаг 3. Как работают связи между таблицами

В реляционной базе данных таблицы связываются через **внешний ключ** — столбец, который хранит `id` записи из другой таблицы.

```
users                       tickets
+----+----------+           +----+---------+------------------+
| id | name     |           | id | user_id | title            |
+----+----------+           +----+---------+------------------+
|  1 | Иван     |  <------  |  1 |    1    | Не работает вход |
|  2 | Мария    |           |  2 |    1    | Ошибка 500       |
+----+----------+           |  3 |    2    | Вопрос по API    |
                            +----+---------+------------------+
```

Тикеты 1 и 2 принадлежат Ивану, тикет 3 — Марии. Это определяется значением `user_id`.

В миграции внешний ключ объявляется так:

```php
$table->foreignId('user_id')->constrained()->onDelete('cascade');
```

- `foreignId('user_id')` — столбец типа `BIGINT UNSIGNED`
- `constrained()` — база данных не позволит вставить тикет с `user_id`, которого нет в таблице `users`
- `onDelete('cascade')` — при удалении пользователя все его тикеты удаляются автоматически

Для `onDelete` есть другие варианты: `restrict` запрещает удалить пользователя, пока у него есть тикеты; `set null` записывает `NULL` в `user_id` при удалении пользователя (тогда столбец должен быть `nullable()`).

### Eloquent Relations

Внешний ключ в базе — просто число. Eloquent превращает его в объект через методы связи в модели.

Если в таблице `tickets` есть столбец `user_id` — у модели `Ticket` пишем `belongsTo`, потому что внешний ключ лежит здесь. У модели `User` пишем `hasMany`, потому что в таблице `users` никакого `ticket_id` нет:

```php
// Ticket — внешний ключ user_id лежит здесь → belongsTo
public function user(): BelongsTo
{
    return $this->belongsTo(User::class);
}

// User — ключа в этой таблице нет, но пользователь владеет многими тикетами → hasMany
public function tickets(): HasMany
{
    return $this->hasMany(Ticket::class);
}
```

После объявления связей в коде можно писать:

```php
$ticket->user      // объект User: SELECT * FROM users WHERE id = ?
$user->tickets     // коллекция Ticket: SELECT * FROM tickets WHERE user_id = ?
```

Правило: **`belongsTo` — в той модели, в таблице которой физически лежит внешний ключ.**

### Цепочка в этом проекте

```
User ──hasMany──► Ticket ──hasMany──► Comment
                  Comment ──belongsTo──► Ticket
                  Comment ──belongsTo──► User
```

У `Comment` два `belongsTo` — потому что в таблице `comments` два внешних ключа: `ticket_id` и `user_id`.

---

## Шаг 4. Генерация файлов

```bash
php artisan make:model Ticket -mc
php artisan make:model Comment -mc
```

`-m` создаёт миграцию, `-c` — контроллер.

---

## Шаг 5. Миграции

### `database/migrations/..._create_tickets_table.php`

> Это файл миграции, не модель. Он находится в `database/migrations/`.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('tickets', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->string('title');
            $table->text('body');
            $table->enum('status', ['open', 'in_progress', 'closed'])->default('open');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('tickets');
    }
};
```

`enum` ограничивает допустимые значения на уровне базы данных — вставить значение вне списка не получится даже в обход PHP-валидации.

### `database/migrations/..._create_comments_table.php`

> Здесь два внешних ключа: комментарий привязан и к тикету, и к пользователю.

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('comments', function (Blueprint $table) {
            $table->id();
            $table->foreignId('ticket_id')->constrained()->onDelete('cascade');
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->text('body');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('comments');
    }
};
```

Запустите миграции:

```bash
php artisan migrate
```

Если нужно начать заново — `php artisan migrate:fresh` удаляет все таблицы и создаёт их заново. Все данные теряются.

---

## Шаг 6. Модели

### `app/Models/Ticket.php`

> Модель Eloquent. Находится в `app/Models/`, не в `database/migrations/`.

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class Ticket extends Model
{
    protected $fillable = ['user_id', 'title', 'body', 'status'];

    // В таблице tickets есть user_id → belongsTo
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    // В таблице comments есть ticket_id → hasMany здесь
    public function comments(): HasMany
    {
        return $this->hasMany(Comment::class);
    }
}
```

### `app/Models/Comment.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

class Comment extends Model
{
    protected $fillable = ['ticket_id', 'user_id', 'body'];

    // В таблице comments есть ticket_id → belongsTo
    public function ticket(): BelongsTo
    {
        return $this->belongsTo(Ticket::class);
    }

    // В таблице comments есть user_id → ещё один belongsTo
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

### `app/Models/User.php` — добавить два метода

Откройте существующий файл и добавьте в класс:

```php
use Illuminate\Database\Eloquent\Relations\HasMany;

public function tickets(): HasMany
{
    return $this->hasMany(Ticket::class);
}

public function comments(): HasMany
{
    return $this->hasMany(Comment::class);
}
```

---

## Шаг 7. Контроллер авторизации

```bash
php artisan make:controller AuthController
```

Откройте `app/Http/Controllers/AuthController.php`. Логика авторизации не зависит от предметной области и полностью совпадает с заданием 1:

```php
<?php

namespace App\Http\Controllers;

use App\Models\User;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;

class AuthController extends Controller
{
    public function register(Request $request): JsonResponse
    {
        $data = $request->validate([
            'name'     => 'required|string|max:255',
            'email'    => 'required|email|unique:users',
            'password' => 'required|string|min:8|confirmed',
        ]);

        $user = User::create([
            'name'     => $data['name'],
            'email'    => $data['email'],
            'password' => Hash::make($data['password']),
        ]);

        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'user'         => $user,
            'access_token' => $token,
            'token_type'   => 'Bearer',
        ], 201);
    }

    public function login(Request $request): JsonResponse
    {
        $data = $request->validate([
            'email'    => 'required|email',
            'password' => 'required|string',
        ]);

        $user = User::where('email', $data['email'])->first();

        if (! $user || ! Hash::check($data['password'], $user->password)) {
            return response()->json(['message' => 'Неверный email или пароль.'], 401);
        }

        $token = $user->createToken('auth_token')->plainTextToken;

        return response()->json([
            'user'         => $user,
            'access_token' => $token,
            'token_type'   => 'Bearer',
        ]);
    }

    public function logout(Request $request): JsonResponse
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json(['message' => 'Выход выполнен.']);
    }
}
```

---

## Шаг 8. Контроллер тикетов

Откройте `app/Http/Controllers/TicketController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Ticket;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class TicketController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        // with('comments') — жадная загрузка.
        // Без неё при обращении к $ticket->comments в цикле Eloquent
        // делал бы отдельный SQL-запрос на каждый тикет.
        // С ним — один запрос: SELECT * FROM comments WHERE ticket_id IN (1,2,3,...)
        $tickets = $request->user()->tickets()->with('comments')->latest()->get();

        return response()->json($tickets);
    }

    public function store(Request $request): JsonResponse
    {
        $data = $request->validate([
            'title' => 'required|string|max:255',
            'body'  => 'required|string',
        ]);

        $ticket = $request->user()->tickets()->create($data);

        return response()->json($ticket, 201);
    }

    public function show(Request $request, Ticket $ticket): JsonResponse
    {
        if ($ticket->user_id !== $request->user()->id) {
            return response()->json(['message' => 'Доступ запрещён.'], 403);
        }

        // load('comments.user') — точка означает вложенную загрузку:
        // загрузить comments, и для каждого из них загрузить его user
        $ticket->load('comments.user');

        return response()->json($ticket);
    }

    public function update(Request $request, Ticket $ticket): JsonResponse
    {
        if ($ticket->user_id !== $request->user()->id) {
            return response()->json(['message' => 'Доступ запрещён.'], 403);
        }

        $data = $request->validate([
            'title'  => 'sometimes|string|max:255',
            'body'   => 'sometimes|string',
            'status' => 'sometimes|in:open,in_progress,closed',
        ]);

        $ticket->update($data);

        return response()->json($ticket);
    }

    public function destroy(Request $request, Ticket $ticket): JsonResponse
    {
        if ($ticket->user_id !== $request->user()->id) {
            return response()->json(['message' => 'Доступ запрещён.'], 403);
        }

        $ticket->delete();

        return response()->json(['message' => 'Тикет удалён.']);
    }
}
```

---

## Шаг 9. Контроллер комментариев

Откройте `app/Http/Controllers/CommentController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\Comment;
use App\Models\Ticket;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class CommentController extends Controller
{
    public function store(Request $request, Ticket $ticket): JsonResponse
    {
        $data = $request->validate([
            'body' => 'required|string',
        ]);

        $comment = $ticket->comments()->create([
            'user_id' => $request->user()->id,
            'body'    => $data['body'],
        ]);

        // Подгружаем автора, чтобы вернуть его в ответе
        $comment->load('user');

        return response()->json($comment, 201);
    }

    public function destroy(Request $request, Ticket $ticket, Comment $comment): JsonResponse
    {
        if ($comment->user_id !== $request->user()->id) {
            return response()->json(['message' => 'Доступ запрещён.'], 403);
        }

        // Без этой проверки можно передать в URL comment_id из другого тикета
        // и удалить комментарий, который к этому тикету не относится
        if ($comment->ticket_id !== $ticket->id) {
            return response()->json(['message' => 'Комментарий не найден.'], 404);
        }

        $comment->delete();

        return response()->json(['message' => 'Комментарий удалён.']);
    }
}
```

---

## Шаг 10. Маршруты

Откройте `routes/api.php` и замените содержимое:

```php
<?php

use App\Http\Controllers\AuthController;
use App\Http\Controllers\CommentController;
use App\Http\Controllers\TicketController;
use Illuminate\Support\Facades\Route;

Route::post('/register', [AuthController::class, 'register']);
Route::post('/login',    [AuthController::class, 'login']);

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);

    Route::get('/tickets',             [TicketController::class, 'index']);
    Route::post('/tickets',            [TicketController::class, 'store']);
    Route::get('/tickets/{ticket}',    [TicketController::class, 'show']);
    Route::put('/tickets/{ticket}',    [TicketController::class, 'update']);
    Route::delete('/tickets/{ticket}', [TicketController::class, 'destroy']);

    Route::post('/tickets/{ticket}/comments',             [CommentController::class, 'store']);
    Route::delete('/tickets/{ticket}/comments/{comment}', [CommentController::class, 'destroy']);
});
```

`{ticket}` и `{comment}` в URL — route model binding. Laravel ищет запись в базе по `id` из URL и передаёт объект в метод контроллера. Если запись не найдена — возвращает `404` до вашего кода.

Проверьте список маршрутов:

```bash
php artisan route:list
```

---

## Шаг 11. Ответ 401 вместо редиректа

По умолчанию при запросе к защищённому маршруту без токена Laravel пытается редиректить на маршрут с именем `login`. В API нужен `401 JSON`.

Откройте `bootstrap/app.php`, найдите `withExceptions` и замените его содержимое:

```php
->withExceptions(function (Exceptions $exceptions) {
    $exceptions->shouldRenderJsonWhen(function (Request $request) {
        return $request->is('api/*');
    });
})
```

Убедитесь, что в начале файла есть:

```php
use Illuminate\Http\Request;
```

Запустите сервер:

```bash
php artisan serve
```

---

## Шаг 12. Тестирование

Для всех запросов:

```
Content-Type: application/json
Accept: application/json
```

Для защищённых маршрутов:

```
Authorization: Bearer <токен>
```

---

### 12.1. Регистрация

**Метод:** `POST` | **URL:** `http://127.0.0.1:8000/api/register`

```json
{
    "name": "Иван Петров",
    "email": "ivan@example.com",
    "password": "password123",
    "password_confirmation": "password123"
}
```

Ожидаемый ответ `201`. Скопируйте `access_token`.

```json
{
    "user": { "id": 1, "name": "Иван Петров", "email": "ivan@example.com" },
    "access_token": "1|aBcDeFgH...",
    "token_type": "Bearer"
}
```

---

### 12.2. Создание тикета

**Метод:** `POST` | **URL:** `http://127.0.0.1:8000/api/tickets`

```json
{
    "title": "Не работает авторизация",
    "body": "При входе получаю ошибку 500."
}
```

Ожидаемый ответ `201`. `status` выставился в `open` автоматически — это значение `default` из миграции.

```json
{
    "id": 1,
    "user_id": 1,
    "title": "Не работает авторизация",
    "body": "При входе получаю ошибку 500.",
    "status": "open",
    "created_at": "...",
    "updated_at": "..."
}
```

---

### 12.3. Список тикетов

**Метод:** `GET` | **URL:** `http://127.0.0.1:8000/api/tickets`

В ответе — массив тикетов, у каждого вложенный массив `comments`. Пока пустой.

---

### 12.4. Добавление комментария

**Метод:** `POST` | **URL:** `http://127.0.0.1:8000/api/tickets/1/comments`

```json
{
    "body": "Проверьте логи в storage/logs/laravel.log"
}
```

Ожидаемый ответ `201`. В ответе — вложенный объект `user`, хотя вы его не передавали. Eloquent подтянул данные по `user_id` через `$comment->load('user')`.

```json
{
    "id": 1,
    "ticket_id": 1,
    "user_id": 1,
    "body": "Проверьте логи в storage/logs/laravel.log",
    "user": {
        "id": 1,
        "name": "Иван Петров",
        "email": "ivan@example.com"
    }
}
```

---

### 12.5. Просмотр одного тикета

**Метод:** `GET` | **URL:** `http://127.0.0.1:8000/api/tickets/1`

В ответе — тикет и массив `comments`. У каждого комментария вложенный объект `user`. Это результат `$ticket->load('comments.user')`.

---

### 12.6. Обновление статуса

**Метод:** `PUT` | **URL:** `http://127.0.0.1:8000/api/tickets/1`

```json
{
    "status": "in_progress"
}
```

Попробуйте передать несуществующий статус — должен вернуться `422`.

---

### 12.7. Удаление комментария

**Метод:** `DELETE` | **URL:** `http://127.0.0.1:8000/api/tickets/1/comments/1`

---

### 12.8. Удаление тикета

**Метод:** `DELETE` | **URL:** `http://127.0.0.1:8000/api/tickets/1`

После удаления запросите `GET /api/tickets` — тикет должен исчезнуть. Комментарии к нему тоже пропадут: это `onDelete('cascade')` на `ticket_id` в таблице `comments`.

---

### 12.9. Проверка прав доступа

Зарегистрируйте второго пользователя через `POST /api/register` с другим email. Получите его токен. Выполните три запроса от его имени:

- `DELETE /api/tickets/1` — ожидается `403`
- `POST /api/tickets/1/comments` — ожидается `201` (комментировать может любой авторизованный)
- `DELETE /api/tickets/1/comments/1` (комментарий первого пользователя) — ожидается `403`

---

## Таблица эндпоинтов

| Метод  | Маршрут                                  | Авторизация |
|--------|------------------------------------------|-------------|
| POST   | /api/register                            | Нет         |
| POST   | /api/login                               | Нет         |
| POST   | /api/logout                              | Да          |
| GET    | /api/tickets                             | Да          |
| POST   | /api/tickets                             | Да          |
| GET    | /api/tickets/{ticket}                    | Да          |
| PUT    | /api/tickets/{ticket}                    | Да          |
| DELETE | /api/tickets/{ticket}                    | Да          |
| POST   | /api/tickets/{ticket}/comments           | Да          |
| DELETE | /api/tickets/{ticket}/comments/{comment} | Да          |

# Как сдавать

- Создайте форк репозитория в вашей организации с названием-этого-репозитория-вашафамилия
- Используя ветку wip сделайте задание
- Зафиксируйте изменения в вашем репозитории
- Когда документ будет готов - создайте пул реквест из ветки wip (вашей) на ветку main (тоже вашу) и укажите меня (ktkv419) как reviewer

Не мержите сами коммит, это сделаю я после проверки задания
