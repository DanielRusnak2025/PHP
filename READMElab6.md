# Лабораторная работа №6
## Обработка и валидация форм
---

## 1. Цель работы

Освоить основные принципы работы с HTML-формами в PHP, включая отправку данных на сервер и их обработку, а также валидацию данных на стороне клиента и сервера.

---

## 2. Модель данных

**Тема проекта:** Управление рецептами — добавление и просмотр кулинарных рецептов.

### 2.1 Поля модели `Recipe`

| Поле | Тип PHP | Тип данных | Описание |
|---|---|---|---|
| `id` | `int` | integer | Уникальный идентификатор (Unix timestamp) |
| `title` | `string` | **string** | Название рецепта (3–255 символов) |
| `author` | `string` | **string** | Имя автора рецепта (2–100 символов) |
| `prep_time` | `int` | integer | Время приготовления в минутах (1–1440) |
| `category` | `string` | **enum** | Категория блюда из фиксированного списка |
| `difficulty` | `string` | **enum** | Уровень сложности из фиксированного списка |
| `tags` | `array` | **enum (checkbox)** | Теги рецепта — множественный выбор из списка |
| `ingredients` | `string` | **text** | Список ингредиентов (длинный текст) |
| `instructions` | `string` | **text** | Пошаговые инструкции (длинный текст) |
| `created_at` | `string` | **date** | Дата создания в формате `YYYY-MM-DD` |
| `updated_at` | `string` | datetime | Дата и время последнего обновления |

**Итого:** 11 полей (минимум 6 ✓), присутствуют все обязательные типы: string ✓, date ✓, enum ✓, text ✓, checkbox ✓.

### 2.2 Допустимые значения enum-полей

**category:** Супы, Салаты, Выпечка, Десерты, Завтраки, Основные блюда, Напитки

**difficulty:** Легко, Средне, Сложно

**tags (checkbox):** Вегетарианское, Без глютена, Острое, Быстрый рецепт, Диетическое

---

## 3. Структура файлов

```
lab6/
├── index.php           — список всех рецептов с сортировкой
├── create.php          — HTML-форма добавления рецепта
├── save.php            — обработка POST-запроса, сохранение
├── Recipe.php          — класс-модель данных
├── RecipeValidator.php — класс валидации
├── RecipeStorage.php   — класс работы с файлом
└── data.json           — хранилище данных
```

---

## 4. Шаг 2 — HTML-форма (`create.php`)

Форма содержит все поля модели, отправляет данные методом `POST` на `save.php`.

### 4.1 Типы полей в форме

| Поле | HTML-элемент | Клиентская валидация |
|---|---|---|
| `title` | `<input type="text">` | `required`, `minlength="3"`, `maxlength="255"` |
| `author` | `<input type="text">` | `required`, `minlength="2"`, `maxlength="100"` |
| `prep_time` | `<input type="number">` | `required`, `min="1"`, `max="1440"` |
| `category` | `<select>` | `required` |
| `difficulty` | `<input type="radio">` | — |
| `tags` | `<input type="checkbox" name="tags[]">` | — |
| `ingredients` | `<textarea>` | `required`, `minlength="10"` |
| `instructions` | `<textarea>` | `required`, `minlength="20"` |
| `created_at` | `<input type="date">` | `required`, `max="<сегодня>"` |

### 4.2 Сохранение данных при ошибке (flash-данные)

При ошибке валидации `save.php` сохраняет введённые значения в `$_SESSION['old']` и перенаправляет обратно на форму. Функция `old()` возвращает эти значения в `value=""` полей, чтобы пользователь не терял введённые данные.

```php
function old(string $key, string $default = ''): string {
    global $old;
    return htmlspecialchars($old[$key] ?? $default);
}
```

---

## 5. Шаг 3 — Обработка на сервере (`save.php`)

### 5.1 Получение данных из `$_POST`

```php
$data = [
    'title'        => trim($_POST['title']        ?? ''),
    'author'       => trim($_POST['author']        ?? ''),
    'prep_time'    => (int)($_POST['prep_time']    ?? 0),
    'category'     => trim($_POST['category']      ?? ''),
    'difficulty'   => trim($_POST['difficulty']    ?? ''),
    'tags'         => array_values(array_filter((array)($_POST['tags'] ?? []))),
    'ingredients'  => trim($_POST['ingredients']   ?? ''),
    'instructions' => trim($_POST['instructions']  ?? ''),
    'created_at'   => trim($_POST['created_at']    ?? ''),
];
```

### 5.2 Серверная валидация

```php
$validator = new RecipeValidator();
if (!$validator->validate($data)) {
    $_SESSION['errors'] = $validator->getErrors();
    $_SESSION['old']    = $data;
    header('Location: create.php');
    exit;
}
```

### 5.3 Сохранение и PRG-паттерн

После успешной валидации данные сохраняются в `data.json`. Применяется паттерн **Post/Redirect/Get** — после POST-запроса выполняется редирект на GET, что предотвращает повторную отправку формы при обновлении страницы.

### 5.4 Формат хранения данных (JSON)

```json
{
    "id": 1745789012,
    "title": "Борщ классический",
    "author": "Иван Петров",
    "prep_time": 120,
    "category": "Супы",
    "difficulty": "Средне",
    "tags": ["Вегетарианское"],
    "ingredients": "...",
    "instructions": "...",
    "created_at": "2026-01-05",
    "updated_at": "2026-01-05 10:00:00"
}
```

---

## 6. Шаг 4 — Вывод данных (`index.php`)

Страница читает все записи из `data.json` через `RecipeStorage::getAll()`, сортирует их и отображает в HTML-таблице с CSS-оформлением.

### 6.1 Сортировка

Сортировка реализована через GET-параметры `?sort=<поле>&dir=asc|desc`. Поддерживаемые поля:

```
title | author | category | difficulty | prep_time | created_at | updated_at
```

При повторном клике на активную колонку — направление меняется на обратное. Числовое поле `prep_time` сравнивается как число, остальные — как строки (`strcmp`).

### 6.2 Защита от XSS

Все данные выводятся через `htmlspecialchars()`:

```php
<td><?= htmlspecialchars($r->title) ?></td>
```

---

## 7. Задание 2 — ООП-реализация

### 7.1 Класс `Recipe` (модель данных)

Хранит все поля одного рецепта. Предоставляет два статических/публичных метода для преобразования данных:

| Метод | Назначение |
|---|---|
| `Recipe::fromArray(array $data)` | Создаёт объект из массива (из JSON или `$_POST`) |
| `$recipe->toArray()` | Преобразует объект в массив для сохранения в JSON |

### 7.2 Класс `RecipeValidator` (валидация)

Содержит правила проверки для каждого поля. Основные методы:

| Метод | Назначение |
|---|---|
| `validate(array $data): bool` | Запускает валидацию всех полей |
| `getErrors(): array` | Возвращает массив ошибок `поле => сообщение` |
| `getAllowedCategories(): array` | Список допустимых категорий |
| `getAllowedDifficulties(): array` | Список уровней сложности |
| `getAllowedTags(): array` | Список допустимых тегов |

Приватные методы валидации: `validateTitle`, `validateAuthor`, `validatePrepTime`, `validateCategory`, `validateDifficulty`, `validateTags`, `validateIngredients`, `validateInstructions`, `validateCreatedAt`.

### 7.3 Класс `RecipeStorage` (хранилище)

Отвечает за всю работу с файловой системой:

| Метод | Назначение |
|---|---|
| `__construct(string $file)` | Принимает путь к JSON-файлу |
| `getAll(): Recipe[]` | Читает все рецепты, возвращает массив объектов |
| `save(Recipe $recipe): bool` | Добавляет новый рецепт и перезаписывает файл |
| `sort(array $recipes, string $field, string $dir): array` | Сортировка по полю и направлению |

### 7.4 Взаимодействие классов

```
create.php  ──GET──►  RecipeValidator::getAllowed*()   ──► форма
     │
     └── POST ──► save.php
                    │
                    ├── RecipeValidator::validate()
                    ├── Recipe::fromArray()
                    └── RecipeStorage::save()
                              │
                              └── data.json

index.php ──► RecipeStorage::getAll()
          ──► RecipeStorage::sort()
          ──► HTML-таблица
```

---

## 8. Документация кода (PHPDoc)

Весь код задокументирован по стандарту PHPDoc. Каждый метод и функция содержат:
- `@param` — описание входных параметров с типами
- `@return` — описание возвращаемого значения с типом
- однострочное описание назначения

Пример:

```php
/**
 * Сортирует массив объектов Recipe по указанному полю и направлению.
 *
 * @param Recipe[] $recipes Массив объектов Recipe
 * @param string   $field   Поле для сортировки (title, prep_time и т.д.)
 * @param string   $dir     Направление: 'asc' или 'desc'
 * @return Recipe[] Отсортированный массив
 */
public function sort(array $recipes, string $field, string $dir): array
```

---

## 9. Выводы

В ходе выполнения лабораторной работы были освоены:

1. **Создание HTML-форм** с различными типами полей: `text`, `number`, `date`, `select`, `radio`, `checkbox`, `textarea`.
2. **Клиентская валидация** с помощью атрибутов `required`, `minlength`, `maxlength`, `min`, `max`.
3. **Серверная валидация** данных из `$_POST` с формированием понятных сообщений об ошибках.
4. **Паттерн PRG** (Post/Redirect/Get) и использование `$_SESSION` для передачи flash-данных между запросами.
5. **Сохранение данных в JSON** с помощью `file_put_contents` и `json_encode`.
6. **ООП-подход**: разделение ответственности между классами `Recipe`, `RecipeValidator`, `RecipeStorage`.
7. **Защита от XSS** через `htmlspecialchars()` при выводе пользовательских данных.
8. **Сортировка данных** по нескольким полям через GET-параметры.
