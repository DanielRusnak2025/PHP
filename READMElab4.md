# Лабораторная работа: Массивы и функции в PHP

---

## Задание 1. Работа с массивами (банковские транзакции)

### 1.1. Подготовка среды

В начале файла `index.php` обязательно включаем строгую типизацию — это помогает избежать скрытых ошибок с типами данных:

```php
<?php

declare(strict_types=1);
```

---

### 1.2. Создание массива транзакций

Каждая транзакция — это вложенный ассоциативный массив с пятью полями:

```php
$transactions = [
    [
        "id"          => 1,
        "date"        => "2019-01-01",
        "amount"      => 100.00,
        "description" => "Payment for groceries",
        "merchant"    => "SuperMart",
    ],
    [
        "id"          => 2,
        "date"        => "2020-02-15",
        "amount"      => 75.50,
        "description" => "Dinner with friends",
        "merchant"    => "Local Restaurant",
    ],
    // ... ещё транзакции
];
```

---

### 1.3. Вывод таблицы через `foreach`

Перебираем массив и выводим каждую строку таблицы. Добавлен дополнительный столбец — количество дней с момента транзакции:

```php
<table border="1">
    <thead>
        <tr>
            <th>ID</th>
            <th>Дата</th>
            <th>Сумма ($)</th>
            <th>Описание</th>
            <th>Организация</th>
            <th>Дней назад</th>
        </tr>
    </thead>
    <tbody>
        <?php foreach ($transactions as $t): ?>
        <tr>
            <td><?= htmlspecialchars((string)$t['id']) ?></td>
            <td><?= htmlspecialchars($t['date']) ?></td>
            <td><?= number_format($t['amount'], 2) ?></td>
            <td><?= htmlspecialchars($t['description']) ?></td>
            <td><?= htmlspecialchars($t['merchant']) ?></td>
            <td><?= daysSinceTransaction($t['date']) ?></td>
        </tr>
        <?php endforeach; ?>
    </tbody>
</table>
```

> `htmlspecialchars()` используется для безопасного вывода — защищает от XSS-атак.

---

### 1.4. Реализация функций

#### `calculateTotalAmount` — сумма всех транзакций

```php
/**
 * Вычисляет общую сумму всех транзакций.
 *
 * @param array $transactions Массив транзакций
 * @return float Общая сумма
 */
function calculateTotalAmount(array $transactions): float
{
    $total = 0.0;
    foreach ($transactions as $t) {
        $total += $t['amount'];
    }
    return $total;
}
```

Вызов и вывод в конце таблицы:

```php
<tr>
    <td colspan="2">Итого:</td>
    <td><?= number_format(calculateTotalAmount($transactions), 2) ?></td>
    <td colspan="3"></td>
</tr>
```

---

#### `findTransactionByDescription` — поиск по описанию

```php
/**
 * Ищет транзакцию по части описания (нечувствительно к регистру).
 *
 * @param string $descriptionPart Часть описания для поиска
 * @return array|null Найденная транзакция или null
 */
function findTransactionByDescription(string $descriptionPart): ?array
{
    global $transactions;

    foreach ($transactions as $t) {
        // stripos — поиск без учёта регистра
        if (stripos($t['description'], $descriptionPart) !== false) {
            return $t;
        }
    }
    return null;
}
```

---

#### `findTransactionById` — поиск по ID через `foreach`

```php
/**
 * Ищет транзакцию по ID с помощью цикла foreach.
 *
 * @param int $id Идентификатор транзакции
 * @return array|null Найденная транзакция или null
 */
function findTransactionById(int $id): ?array
{
    global $transactions;

    foreach ($transactions as $t) {
        if ($t['id'] === $id) {
            return $t;
        }
    }
    return null;
}
```

---

#### `findTransactionByIdFilter` — поиск по ID через `array_filter`

```php
/**
 * Ищет транзакцию по ID с помощью array_filter.
 *
 * @param int $id Идентификатор транзакции
 * @return array|null Найденная транзакция или null
 */
function findTransactionByIdFilter(int $id): ?array
{
    global $transactions;

    // array_filter оставляет только те элементы, для которых колбэк вернул true
    $result = array_filter($transactions, function ($t) use ($id) {
        return $t['id'] === $id;
    });

    // array_values сбрасывает ключи, чтобы обратиться по индексу [0]
    $result = array_values($result);
    return $result[0] ?? null;
}
```

> `array_filter` — это более «функциональный» подход, результат тот же, но код короче и выразительнее.

---

#### `daysSinceTransaction` — дней с момента транзакции

```php
/**
 * Возвращает количество дней между датой транзакции и сегодняшним днём.
 *
 * @param string $date Дата транзакции в формате YYYY-MM-DD
 * @return int Количество прошедших дней
 */
function daysSinceTransaction(string $date): int
{
    $transactionDate = new DateTime($date);
    $today = new DateTime('today');
    $diff = $today->diff($transactionDate);
    return (int) $diff->days;
}
```

> Используется встроенный класс `DateTime` и его метод `diff()` — он возвращает объект `DateInterval` с полем `days`.

---

#### `addTransaction` — добавление транзакции

```php
/**
 * Добавляет новую транзакцию в глобальный массив $transactions.
 *
 * @param int    $id          Уникальный идентификатор
 * @param string $date        Дата в формате YYYY-MM-DD
 * @param float  $amount      Сумма транзакции
 * @param string $description Описание платежа
 * @param string $merchant    Название организации
 * @return void
 */
function addTransaction(int $id, string $date, float $amount, string $description, string $merchant): void
{
    global $transactions;

    $transactions[] = [
        "id"          => $id,
        "date"        => $date,
        "amount"      => $amount,
        "description" => $description,
        "merchant"    => $merchant,
    ];
}
```

Использование:

```php
addTransaction(6, "2024-07-19", 89.90, "Gym membership", "FitLife");
```

> `global $transactions` делает переменную доступной внутри функции. Без этого ключевого слова функция не «видит» переменные из внешней области видимости.

---

### 1.5. Сортировка транзакций

#### По дате (по возрастанию) через `usort`

```php
usort($transactions, function (array $a, array $b): int {
    return strcmp($a['date'], $b['date']);
});
```

> `strcmp` сравнивает строки лексикографически. Для формата `YYYY-MM-DD` это работает корректно как сравнение дат.

#### По сумме (по убыванию)

```php
usort($transactions, function (array $a, array $b): int {
    if ($a['amount'] === $b['amount']) return 0;
    return $a['amount'] < $b['amount'] ? 1 : -1;
});
```

> Возвращаем `1` когда `a < b` — это переворачивает порядок на убывающий.

---

## Задание 2. Работа с файловой системой (галерея изображений)

Создаём страницу, которая читает папку `image/` и выводит все `.jpg` файлы как галерею:

```php
<?php
// Путь к папке с изображениями
$dir = 'image/';

// Сканируем содержимое директории
// scandir() возвращает массив имён файлов, включая "." и ".."
$files = scandir($dir);

if ($files === false) {
    return; // Если папка не найдена — выходим
}
?>

<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Галерея</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 0; background: #1a1a1a; color: #fff; }
        header { background: #333; padding: 20px; text-align: center; font-size: 24px; }
        nav { background: #444; padding: 10px; text-align: center; }
        nav a { color: #adf; margin: 0 10px; text-decoration: none; }
        .gallery { display: flex; flex-wrap: wrap; gap: 10px; padding: 20px; justify-content: center; }
        .gallery img { width: 200px; height: 150px; object-fit: cover; border-radius: 6px; }
        footer { background: #333; text-align: center; padding: 15px; margin-top: 20px; }
    </style>
</head>
<body>

<header>Фотогалерея</header>

<nav>
    <a href="#">Главная</a>
    <a href="#">О нас</a>
    <a href="#">Контакты</a>
</nav>

<div class="gallery">
    <?php
    for ($i = 0; $i < count($files); $i++) {
        // Пропускаем "." и ".."
        if ($files[$i] === "." || $files[$i] === "..") {
            continue;
        }

        $path = $dir . $files[$i];

        // Выводим только .jpg файлы
        if (pathinfo($path, PATHINFO_EXTENSION) === 'jpg') {
            echo "<img src='" . htmlspecialchars($path) . "' alt='" . htmlspecialchars($files[$i]) . "'>";
        }
    }
    ?>
</div>

<footer>© 2024 Фотогалерея</footer>

</body>
</html>
```

> `pathinfo($path, PATHINFO_EXTENSION)` возвращает расширение файла — удобно фильтровать только `.jpg`.

---

## Выводы

В ходе работы были освоены:

- создание и обход ассоциативных массивов с помощью `foreach`;
- написание функций с типизированными параметрами и возвращаемыми значениями в соответствии с `declare(strict_types=1)`;
- поиск в массиве двумя способами: через `foreach` и через `array_filter` с анонимной функцией;
- сортировка массива объектов по разным полям с помощью `usort`;
- работа с датами через класс `DateTime` и метод `diff()`;
- сканирование директории с помощью `scandir()` и вывод файлов на страницу;
- документирование кода по стандарту PHPDoc.
