
# Лабораторная работа №7

## Шаблонизация в PHP: нативные шаблоны и Twig

---

## 1. Цель работы

Освоить принципы шаблонизации в PHP — как с использованием нативных PHP-шаблонов, так и с применением готового шаблонизатора Twig. Улучшить структуру проекта из лабораторной №6, разделив логику обработки данных и представление.

---

## 2. Структура проекта

```
lab7/
├── composer.json               — зависимости (Twig)
├── composer.lock
├── vendor/                     — пакеты Composer (Twig и др.)
│
├── src/
│   ├── Recipe.php              — модель данных (без изменений)
│   ├── RecipeValidator.php     — валидация (без изменений)
│   ├── RecipeStorage.php       — хранилище JSON (без изменений)
│   └── TwigFactory.php         — фабрика: создаёт и настраивает Twig
│
├── templates/                  — нативные PHP-шаблоны
│   ├── layout.php              — общий макет (header + footer)
│   ├── list.php                — таблица рецептов
│   └── form.php                — форма добавления рецепта
│
├── templates/twig/             — Twig-шаблоны
│   ├── layout.html.twig        — базовый макет (extends/block)
│   ├── list.html.twig          — таблица рецептов
│   └── form.html.twig          — форма добавления рецепта
│
├── index.php                   — список рецептов (роутер → шаблон)
├── create.php                  — форма добавления (роутер → шаблон)
├── save.php                    — обработка POST (без шаблона, редирект)
└── data.json                   — хранилище данных
```

---

## 3. Шаг 1 — Нативные PHP-шаблоны

### 3.1 Принцип разделения логики и представления

Каждый PHP-файл-обработчик (`index.php`, `create.php`) теперь только **подготавливает переменные** и затем **подключает шаблон**. Сами шаблоны в `templates/` не содержат никакой бизнес-логики.

**До (лаб. №6) — логика и вёрстка смешаны:**
```php
// index.php
$recipes = $storage->getAll();
usort($recipes, ...);
echo "<table>";
foreach ($recipes as $r) {
    echo "<tr><td>" . htmlspecialchars($r->title) . "</td></tr>";
}
echo "</table>";
```

**После (лаб. №7) — разделены:**
```php
// index.php — только логика
$recipes = $storage->getAll();
$recipes = $storage->sort($recipes, $sort, $dir);
require __DIR__ . '/templates/layout.php'; // шаблон делает вёрстку
```

### 3.2 Общий макет `templates/layout.php`

Макет — это «обёртка» для всех страниц. Он выводит `<html>`, `<head>`, шапку и подвал, а в середину вставляет содержимое конкретной страницы через `$content`:

```php
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title><?= htmlspecialchars($title ?? 'Рецепты') ?></title>
    <link rel="stylesheet" href="/style.css">
</head>
<body>
    <header>
        <nav>
            <a href="/index.php">Все рецепты</a>
            <a href="/create.php">Добавить рецепт</a>
        </nav>
    </header>
    <main>
        <?php require __DIR__ . '/' . $content; ?>
    </main>
    <footer>
        <p>Книга рецептов &copy; 2026</p>
    </footer>
</body>
</html>
```

Использование в `index.php`:
```php
$title   = 'Список рецептов';
$content = 'list.php';
require __DIR__ . '/templates/layout.php';
```

### 3.3 Шаблон списка `templates/list.php`

Шаблон только выводит данные из переменной `$recipes`. Никакой логики получения данных — только HTML:

```php
<h1>Все рецепты</h1>

<?php if (empty($recipes)): ?>
    <p class="empty">Рецептов пока нет. <a href="/create.php">Добавить первый</a></p>
<?php else: ?>
<table class="recipes-table">
    <thead>
        <tr>
            <th><a href="?sort=title&dir=<?= $nextDir ?>">Название</a></th>
            <th><a href="?sort=author&dir=<?= $nextDir ?>">Автор</a></th>
            <th><a href="?sort=category&dir=<?= $nextDir ?>">Категория</a></th>
            <th><a href="?sort=prep_time&dir=<?= $nextDir ?>">Время (мин)</a></th>
            <th><a href="?sort=difficulty&dir=<?= $nextDir ?>">Сложность</a></th>
            <th><a href="?sort=created_at&dir=<?= $nextDir ?>">Дата</a></th>
        </tr>
    </thead>
    <tbody>
        <?php foreach ($recipes as $r): ?>
        <tr>
            <td><?= htmlspecialchars($r->title) ?></td>
            <td><?= htmlspecialchars($r->author) ?></td>
            <td><?= htmlspecialchars($r->category) ?></td>
            <td><?= $r->prep_time ?></td>
            <td><?= htmlspecialchars($r->difficulty) ?></td>
            <td><?= htmlspecialchars($r->created_at) ?></td>
        </tr>
        <?php endforeach; ?>
    </tbody>
</table>
<?php endif; ?>
```

### 3.4 Шаблон формы `templates/form.php`

Форма использует переменные `$errors` и `$old`, которые подготовил обработчик:

```php
<h1>Добавить рецепт</h1>

<form method="POST" action="/save.php">

    <div class="field">
        <label for="title">Название рецепта</label>
        <input type="text" id="title" name="title"
               value="<?= htmlspecialchars($old['title'] ?? '') ?>"
               minlength="3" maxlength="255" required>
        <?php if (!empty($errors['title'])): ?>
            <span class="error"><?= htmlspecialchars($errors['title']) ?></span>
        <?php endif; ?>
    </div>

    <div class="field">
        <label for="prep_time">Время приготовления (мин)</label>
        <input type="number" id="prep_time" name="prep_time"
               value="<?= (int)($old['prep_time'] ?? '') ?>"
               min="1" max="1440" required>
        <?php if (!empty($errors['prep_time'])): ?>
            <span class="error"><?= htmlspecialchars($errors['prep_time']) ?></span>
        <?php endif; ?>
    </div>

    <!-- ... остальные поля ... -->

    <button type="submit">Сохранить рецепт</button>
</form>
```

---

## 4. Шаг 2 — Шаблонизатор Twig

### 4.1 Установка через Composer

```bash
composer require twig/twig
```

После этого в `composer.json` появляется зависимость:

```json
{
    "require": {
        "twig/twig": "^3.0"
    }
}
```

### 4.2 Настройка Twig — `src/TwigFactory.php`

Фабрика создаёт и настраивает экземпляр Twig, регистрирует кастомный фильтр:

```php
<?php

declare(strict_types=1);

namespace App\Services;

use Twig\Environment;
use Twig\Loader\FilesystemLoader;
use Twig\TwigFilter;

class TwigFactory
{
    public static function create(): Environment
    {
        $loader = new FilesystemLoader(__DIR__ . '/../templates/twig');

        $twig = new Environment($loader, [
            'cache'       => false,   // в продакшене: __DIR__ . '/../cache'
            'auto_reload' => true,
            'debug'       => true,
        ]);

        // Регистрируем кастомный фильтр format_duration
        $twig->addFilter(new TwigFilter('format_duration', function (int $minutes): string {
            if ($minutes < 60) {
                return $minutes . ' мин';
            }
            $hours = intdiv($minutes, 60);
            $mins  = $minutes % 60;
            return $mins > 0
                ? $hours . ' ч ' . $mins . ' мин'
                : $hours . ' ч';
        }));

        return $twig;
    }
}
```

### 4.3 Базовый макет `templates/twig/layout.html.twig`

В Twig используется **наследование шаблонов**: базовый шаблон определяет блоки (`{% block %}`), дочерние шаблоны их переопределяют:

```twig
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>{% block title %}Рецепты{% endblock %} — Книга рецептов</title>
    <link rel="stylesheet" href="/style.css">
</head>
<body>
    <header>
        <nav>
            <a href="/index.php">Все рецепты</a>
            <a href="/create.php">Добавить рецепт</a>
        </nav>
    </header>

    <main>
        {% block content %}{% endblock %}
    </main>

    <footer>
        <p>Книга рецептов &copy; {{ "now"|date("Y") }}</p>
    </footer>
</body>
</html>
```

### 4.4 Шаблон списка `templates/twig/list.html.twig`

```twig
{% extends "layout.html.twig" %}

{% block title %}Список рецептов{% endblock %}

{% block content %}
<h1>Все рецепты</h1>

{% if recipes is empty %}
    <p class="empty">Рецептов пока нет. <a href="/create.php">Добавить первый</a></p>
{% else %}
<table class="recipes-table">
    <thead>
        <tr>
            <th><a href="?sort=title&dir={{ next_dir }}">Название</a></th>
            <th><a href="?sort=author&dir={{ next_dir }}">Автор</a></th>
            <th><a href="?sort=category&dir={{ next_dir }}">Категория</a></th>
            <th><a href="?sort=prep_time&dir={{ next_dir }}">Время</a></th>
            <th><a href="?sort=difficulty&dir={{ next_dir }}">Сложность</a></th>
            <th><a href="?sort=created_at&dir={{ next_dir }}">Дата</a></th>
        </tr>
    </thead>
    <tbody>
        {% for recipe in recipes %}
        <tr>
            <td>{{ recipe.title }}</td>
            <td>{{ recipe.author }}</td>
            <td>{{ recipe.category }}</td>
            <td>{{ recipe.prep_time|format_duration }}</td>
            <td>{{ recipe.difficulty }}</td>
            <td>{{ recipe.created_at }}</td>
        </tr>
        {% endfor %}
    </tbody>
</table>
{% endif %}
{% endblock %}
```

### 4.5 Шаблон формы `templates/twig/form.html.twig`

```twig
{% extends "layout.html.twig" %}

{% block title %}Добавить рецепт{% endblock %}

{% block content %}
<h1>Добавить рецепт</h1>

<form method="POST" action="/save.php">

    <div class="field">
        <label for="title">Название рецепта</label>
        <input type="text" id="title" name="title"
               value="{{ old.title|default('') }}"
               minlength="3" maxlength="255" required>
        {% if errors.title is defined %}
            <span class="error">{{ errors.title }}</span>
        {% endif %}
    </div>

    <div class="field">
        <label for="prep_time">Время приготовления (мин)</label>
        <input type="number" id="prep_time" name="prep_time"
               value="{{ old.prep_time|default('') }}"
               min="1" max="1440" required>
        {% if errors.prep_time is defined %}
            <span class="error">{{ errors.prep_time }}</span>
        {% endif %}
    </div>

    <div class="field">
        <label for="category">Категория</label>
        <select id="category" name="category" required>
            <option value="">-- выберите --</option>
            {% for cat in categories %}
                <option value="{{ cat }}"
                    {% if old.category == cat %}selected{% endif %}>
                    {{ cat }}
                </option>
            {% endfor %}
        </select>
    </div>

    <div class="field">
        <label>Теги</label>
        {% for tag in allowed_tags %}
            <label class="checkbox-label">
                <input type="checkbox" name="tags[]" value="{{ tag }}"
                    {% if tag in (old.tags|default([])) %}checked{% endif %}>
                {{ tag }}
            </label>
        {% endfor %}
    </div>

    <button type="submit">Сохранить рецепт</button>
</form>
{% endblock %}
```

### 4.6 Рендеринг через Twig в `index.php`

```php
<?php
require __DIR__ . '/vendor/autoload.php';

use App\Services\TwigFactory;
use App\Database\RecipeStorage;

$storage = new RecipeStorage(__DIR__ . '/data.json');
$sort    = $_GET['sort'] ?? 'updated_at';
$dir     = $_GET['dir']  ?? 'desc';
$recipes = $storage->sort($storage->getAll(), $sort, $dir);
$nextDir = $dir === 'asc' ? 'desc' : 'asc';

$twig = TwigFactory::create();
echo $twig->render('list.html.twig', [
    'recipes'  => $recipes,
    'sort'     => $sort,
    'dir'      => $dir,
    'next_dir' => $nextDir,
]);
```

---

## 5. Шаг 3 — Кастомный фильтр Twig `format_duration`

### 5.1 Назначение фильтра

Фильтр `format_duration` преобразует время приготовления из **минут** в **читаемый формат**. Это практически полезно для рецептов: вместо "90" пользователь видит "1 ч 30 мин".

### 5.2 Реализация

```php
$twig->addFilter(new TwigFilter('format_duration', function (int $minutes): string {
    if ($minutes < 60) {
        return $minutes . ' мин';
    }
    $hours = intdiv($minutes, 60);
    $mins  = $minutes % 60;
    return $mins > 0
        ? $hours . ' ч ' . $mins . ' мин'
        : $hours . ' ч';
}));
```

### 5.3 Примеры работы фильтра

| Входное значение (мин) | Результат         |
|------------------------|-------------------|
| 15                     | 15 мин            |
| 60                     | 1 ч               |
| 90                     | 1 ч 30 мин        |
| 120                    | 2 ч               |
| 150                    | 2 ч 30 мин        |
| 1440                   | 24 ч              |

### 5.4 Использование в шаблоне

```twig
{# В шаблоне list.html.twig #}
<td>{{ recipe.prep_time|format_duration }}</td>

{# Вывод для prep_time = 90: "1 ч 30 мин" #}
```

---

## 6. Сравнение нативных PHP-шаблонов и Twig

| Критерий | Нативный PHP | Twig |
|---|---|---|
| Синтаксис | `<?= $var ?>`, `<?php foreach ?>` | `{{ var }}`, `{% for %}` |
| Наследование макетов | Вручную через `require` | Встроенное: `extends` + `block` |
| Защита от XSS | Вручную: `htmlspecialchars()` | Автоматически: `{{ var }}` экранирует |
| Кастомные фильтры | Обычные PHP-функции | `TwigFilter` — регистрируется в окружении |
| Производительность | Чуть быстрее (нет компиляции) | Компилирует в PHP-кеш, быстро при кешировании |
| Читаемость шаблонов | Смешан PHP-код и HTML | Чистый синтаксис, только отображение |
| Порог вхождения | Нулевой (знаешь PHP) | Нужно изучить синтаксис Twig |

---

## 7. Выводы

В ходе выполнения лабораторной работы были освоены:

1. **Разделение логики и представления** — обработчики подготавливают данные, шаблоны только выводят.
2. **Нативные PHP-шаблоны** — общий макет через `layout.php` с подстановкой контента.
3. **Установка Twig через Composer** и создание фабрики `TwigFactory` для настройки окружения.
4. **Наследование Twig-шаблонов** — базовый макет `layout.html.twig` с блоками `title` и `content`, которые дочерние шаблоны переопределяют через `{% extends %}` и `{% block %}`.
5. **Автоматическое экранирование** в Twig — защита от XSS без явного вызова `htmlspecialchars()`.
6. **Кастомный фильтр `format_duration`** — практически применимый фильтр для рецептов, преобразующий минуты в формат "X ч Y мин".
7. **Сравнение подходов** — оба варианта присутствуют в проекте, что позволяет оценить их преимущества и недостатки на практике.
