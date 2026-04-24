[README.md](https://github.com/user-attachments/files/27042849/README.md)
# Лабораторная работа №7 — Шаблонизация

## Цель

Разделить логику и представление в PHP-приложении. Реализовать два подхода к шаблонизации:
- **Нативный PHP** — ручной движок через `ob_start()` / `ob_get_clean()` + `extract()`
- **Twig** — промышленный шаблонизатор с наследованием шаблонов и кастомными фильтрами

---

## Структура проекта

```
lab7/
├── composer.json            # Зависимости (twig/twig ^3.0)
├── data.json                # Хранилище рецептов (JSON)
│
├── index.php                # Точка входа — нативные PHP-шаблоны
├── index_twig.php           # Точка входа — Twig-шаблоны
│
├── src/
│   ├── Recipe.php           # Модель рецепта (10 полей)
│   ├── RecipeStorage.php    # CRUD поверх JSON-файла
│   ├── RecipeValidator.php  # Валидация полей (whitelist + типы)
│   └── functions.php        # render(), h(), old(), formatDuration() и др.
│
├── templates/               # Нативные PHP-шаблоны (Шаг 1)
│   ├── layout.php           # Базовый макет (HTML, CSS, flash)
│   ├── list.php             # Список рецептов с сортировкой
│   ├── form.php             # Форма создания и редактирования
│   └── detail.php           # Детальный просмотр рецепта
│
└── twig_templates/          # Twig-шаблоны (Шаг 2–3)
    ├── layout.html.twig     # Базовый макет ({% block content %})
    ├── list.html.twig       # Список с макросами sort_url/sort_arrow
    ├── form.html.twig       # Форма ({% for %}, {% if %}, наследование)
    └── detail.html.twig     # Детальный просмотр + кастомный фильтр
```

---

## Запуск

### 1. Установить зависимости (один раз)

```bash
cd lab7
composer install
```

Команда скачает Twig в `vendor/` и создаст `vendor/autoload.php`.

### 2. Запустить встроенный сервер PHP

**Нативные шаблоны:**
```bash
php -S localhost:8080 index.php
```
Открыть: http://localhost:8080

**Twig-шаблоны:**
```bash
php -S localhost:8081 index_twig.php
```
Открыть: http://localhost:8081

> Оба варианта используют один `data.json` — рецепты общие.

---

## Реализованные шаги

### Шаг 1 — Нативный PHP-движок шаблонов (10 баллов)

Файл `src/functions.php` реализует функцию `render()`:

```php
function render(string $template, array $data = []): void {
    $data = array_merge($GLOBALS['_renderGlobals'], $data);
    extract($data, EXTR_SKIP);          // массив → локальные переменные
    ob_start();                          // начать буферизацию вывода
    require __DIR__ . '/../templates/' . $template . '.php';
    $content = ob_get_clean();           // захватить буфер
    require __DIR__ . '/../templates/layout.php';  // вставить в макет
}
```

- Логика (маршрутизация, валидация, хранилище) — в `index.php`
- Представление — в `templates/*.php`
- `flash`-сообщения передаются через `setRenderGlobals()` во все шаблоны

### Шаг 2 — Twig: наследование шаблонов (10 баллов)

`twig_templates/layout.html.twig` объявляет блок `content`.
Каждый дочерний шаблон:

```twig
{% extends 'layout.html.twig' %}

{% block content %}
  {# содержимое страницы #}
{% endblock %}
```

Используемые конструкции Twig:
- `{% extends %}` — наследование макета
- `{% block %}` — переопределяемые секции
- `{% for item in items %}` — цикл
- `{% if condition %}` — условие
- `{{ variable }}` — вывод с авто-экранированием
- `{% macro %}` + `{% import _self as self %}` — повторно используемые фрагменты (sort_url, sort_arrow в `list.html.twig`)

### Шаг 3 — Кастомный фильтр Twig `format_duration` (10 баллов)

Регистрация в `index_twig.php`:

```php
$twig->addFilter(new \Twig\TwigFilter(
    'format_duration',
    fn(int $minutes): string => formatDuration($minutes)
));
```

Использование в шаблоне:

```twig
{{ recipe.prep_time | format_duration }}
```

Вывод: `45 мин` / `1 ч` / `1 ч 30 мин`

Функция `formatDuration()` определена один раз в `src/functions.php` и используется и в нативных шаблонах, и через обёртку-фильтр в Twig.

---

## Сравнение подходов

| Критерий | Нативный PHP | Twig |
|---|---|---|
| Синтаксис | `<?= h($var) ?>` | `{{ var }}` |
| Экранирование | Вручную через `h()` | Авто (`auto_escape`) |
| Наследование макетов | Через `ob_start()` + `$content` | `{% extends %}` / `{% block %}` |
| Кастомные хелперы | Функции PHP | Фильтры и функции Twig |
| Зависимости | Не нужны | `composer require twig/twig` |

---

## Что не реализовано

- **Кэш Twig** — в `index_twig.php` установлено `'cache' => false`. В production нужно указать папку для кэша (например, `__DIR__ . '/cache'`).
- **Отдельный CSS-файл** — стили встроены в `layout.php` и `layout.html.twig` inline для упрощения запуска без веб-сервера.
- **Аутентификация** — вне рамок задания.
