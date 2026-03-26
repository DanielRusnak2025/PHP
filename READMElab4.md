# Лабораторная работа №4 — Разбор кода index.php

---

## 1. Строгая типизация

```php
<?php
declare(strict_types=1);
```

Открывающий тег PHP. `declare(strict_types=1)` включает **строгую типизацию** — PHP будет строго проверять типы аргументов функций. Например, если функция ожидает `int`, а передать строку `"3"` — будет ошибка, а не молчаливое преобразование.

---

## 2. Массив транзакций `$transactions`

```php
$transactions = [
    [
        "id"          => 1,
        "date"        => "2019-01-01",
        "amount"      => 100.00,
        "description" => "Payment for groceries",
        "merchant"    => "SuperMart",
    ],
    ...
];
```

Это **многомерный ассоциативный массив**. Каждый элемент — это тоже массив (одна транзакция) с ключами:

| Ключ | Тип | Описание |
|------|-----|----------|
| `id` | int | Уникальный номер транзакции |
| `date` | string | Дата в формате YYYY-MM-DD |
| `amount` | float | Сумма платежа |
| `description` | string | Описание платежа |
| `merchant` | string | Название организации |

Квадратные скобки `[]` — современный синтаксис создания массива (аналог старого `array()`). `=>` — оператор присваивания ключа значению в ассоциативном массиве.

---

## 3. Функции

### 3.1 `calculateTotalAmount`

```php
function calculateTotalAmount(array $transactions): float
{
    $total = 0.0;
    foreach ($transactions as $t) {
        $total += $t['amount'];
    }
    return $total;
}
```

**Что делает:** вычисляет общую сумму всех транзакций.

**Разбор строк:**
- `array $transactions` — параметр: принимает массив транзакций
- `: float` — тип возвращаемого значения (число с плавающей точкой)
- `$total = 0.0` — начальное значение суммы
- `foreach ($transactions as $t)` — перебираем каждую транзакцию, называя её `$t`
- `$total += $t['amount']` — прибавляем сумму транзакции к итогу (`+=` это сокращение от `$total = $total + $t['amount']`)
- `return $total` — возвращаем итоговую сумму

---

### 3.2 `findTransactionByDescription`

```php
function findTransactionByDescription(string $descriptionPart): ?array
{
    global $transactions;

    foreach ($transactions as $t) {
        if (stripos($t['description'], $descriptionPart) !== false) {
            return $t;
        }
    }
    return null;
}
```

**Что делает:** ищет транзакцию по части описания, без учёта регистра.

**Разбор строк:**
- `string $descriptionPart` — часть строки для поиска, например `"groceries"`
- `?array` — функция вернёт либо массив (транзакцию), либо `null` (знак `?` означает "может быть null")
- `global $transactions` — берём глобальный массив внутрь функции (без этого функция его не видит)
- `stripos($t['description'], $descriptionPart)` — ищет подстроку в строке без учёта регистра (`str` = строка, `i` = case-insensitive, `pos` = позиция). Возвращает позицию или `false`
- `!== false` — если позиция найдена (не `false`), значит подстрока есть
- `return $t` — сразу возвращаем найденную транзакцию
- `return null` — если ничего не нашли

---

### 3.3 `findTransactionById` (через foreach)

```php
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

**Что делает:** ищет транзакцию по точному ID с помощью цикла.

**Разбор строк:**
- `int $id` — принимает целое число (идентификатор)
- `$t['id'] === $id` — строгое сравнение `===` проверяет и **значение**, и **тип** одновременно. `==` сравнивало бы только значение, `===` надёжнее
- Как только нашёл совпадение — сразу `return $t`, не продолжая цикл

---

### 3.4 `findTransactionByIdFilter` (через array_filter)

```php
function findTransactionByIdFilter(int $id): ?array
{
    global $transactions;

    $result = array_filter($transactions, function ($t) use ($id) {
        return $t['id'] === $id;
    });

    $result = array_values($result);
    return $result[0] ?? null;
}
```

**Что делает:** то же самое, но через встроенную функцию `array_filter`.

**Разбор строк:**
- `array_filter($transactions, ...)` — фильтрует массив, оставляя только элементы, для которых колбэк вернул `true`
- `function ($t) use ($id)` — **анонимная функция** (колбэк). `use ($id)` захватывает переменную `$id` из внешнего контекста внутрь анонимной функции — без `use` она была бы недоступна
- `array_values($result)` — после `array_filter` ключи массива могут быть непоследовательными (например, 0, 2, 4). `array_values` сбрасывает их обратно в 0, 1, 2...
- `$result[0] ?? null` — берём первый элемент. `??` — оператор **null-coalescing**: если `$result[0]` не существует, вернёт `null`

---

### 3.5 `daysSinceTransaction`

```php
function daysSinceTransaction(string $date): int
{
    $transactionDate = new DateTime($date);
    $today = new DateTime('today');
    $diff = $today->diff($transactionDate);
    return (int) $diff->days;
}
```

**Что делает:** считает, сколько дней прошло с даты транзакции до сегодня.

**Разбор строк:**
- `new DateTime($date)` — создаёт объект класса `DateTime` из строки вида `"2019-01-01"`
- `new DateTime('today')` — объект с сегодняшней датой
- `->diff($transactionDate)` — метод класса `DateTime`, вычисляет разницу между двумя датами. Возвращает объект `DateInterval`
- `$diff->days` — свойство объекта `DateInterval`, содержит полное количество дней разницы
- `(int)` — явное приведение типа к целому числу

---

### 3.6 `addTransaction`

```php
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

**Что делает:** добавляет новую транзакцию в глобальный массив.

**Разбор строк:**
- Все параметры строго типизированы: `int`, `string`, `float`
- `: void` — функция ничего не возвращает
- `global $transactions` — получаем доступ к глобальному массиву
- `$transactions[]` — запись с пустыми скобками добавляет новый элемент **в конец** массива автоматически
- Создаём новый ассоциативный массив из переданных параметров

**Вызов функции:**
```php
addTransaction(6, "2024-07-19", 89.90, "Gym membership", "FitLife");
```

---

## 4. Сортировки

### 4.1 По дате (по возрастанию)

```php
usort($transactions, function (array $a, array $b): int {
    return strcmp($a['date'], $b['date']);
});
```

- `usort` — сортирует массив с помощью пользовательской функции сравнения (u = user-defined)
- Функция сравнения получает два элемента `$a` и `$b`
- `strcmp($a['date'], $b['date'])` — сравнивает строки лексикографически. Даты в формате `YYYY-MM-DD` корректно сортируются как строки
- `strcmp` возвращает: отрицательное число (a < b), 0 (равны), положительное (a > b)

### 4.2 По сумме (по убыванию)

```php
usort($transactions, function (array $a, array $b): int {
    if ($a['amount'] === $b['amount']) return 0;
    return $a['amount'] < $b['amount'] ? 1 : -1;
});
```

- Сравниваем числовые поля вручную
- Если суммы равны — возвращаем `0`
- Если `$a['amount']` **меньше** — возвращаем `1` (то есть `$a` ставится **после** `$b`), это даёт **убывающий** порядок
- `? 1 : -1` — тернарный оператор: если условие истинно — `1`, иначе `-1`

---

## 5. Галерея изображений

```php
$dir = 'image/';
$images = [];

$files = scandir($dir);
```

- `$dir = 'image/'` — путь к папке с изображениями
- `$images = []` — пустой массив, куда будем собирать пути к картинкам
- `scandir($dir)` — возвращает массив всех файлов и папок в директории, или `false` при ошибке

```php
if ($files !== false) {
    for ($i = 0; $i < count($files); $i++) {
        if ($files[$i] != "." && $files[$i] != "..") {
```

- `!== false` — проверяем, что `scandir` не вернул ошибку
- `"."` и `".."` — специальные записи файловой системы (текущая и родительская папка), их нужно пропустить

```php
            $ext = strtolower(pathinfo($files[$i], PATHINFO_EXTENSION));
            if (in_array($ext, ['jpg', 'jpeg', 'png', 'gif', 'webp'])) {
                $images[] = $path;
            }
```

- `pathinfo($files[$i], PATHINFO_EXTENSION)` — извлекает расширение файла (например, `jpg`)
- `strtolower` — переводим в нижний регистр, чтобы `.JPG` и `.jpg` обрабатывались одинаково
- `in_array` — проверяет, есть ли расширение в списке допустимых форматов

---

## 6. HTML-шаблон

### Вывод таблицы

```php
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
```

- `<?= ... ?>` — короткий тег вывода, аналог `<?php echo ... ?>`
- `htmlspecialchars(...)` — защита от XSS-атак: преобразует спецсимволы `<`, `>`, `&`, `"` в безопасные HTML-сущности
- `(string)$t['id']` — приводим `int` к строке, так как `htmlspecialchars` ожидает строку
- `number_format($t['amount'], 2)` — форматирует число: второй аргумент `2` задаёт два знака после запятой

### Итоговая строка

```php
<tr class="total-row">
    <td colspan="2">Итого:</td>
    <td><?= number_format(calculateTotalAmount($transactions), 2) ?></td>
    <td colspan="3"></td>
</tr>
```

- `colspan="2"` — ячейка занимает 2 столбца
- Вызываем `calculateTotalAmount($transactions)` прямо в шаблоне и форматируем результат

### Вывод результатов поиска

```php
<?php if ($found): ?>
    Найдено: <?= htmlspecialchars($found['description']) ?> ...
<?php else: ?>
    Не найдено.
<?php endif; ?>
```

Альтернативный синтаксис условий (`if:` / `endif;`) удобен внутри HTML-шаблонов — код выглядит чище, чем с фигурными скобками.

### Вывод галереи

```php
<?php for ($i = 0; $i < count($images); $i++): ?>
    <img src="<?= htmlspecialchars($images[$i]) ?>"
         alt="<?= htmlspecialchars(basename($images[$i])) ?>">
<?php endfor; ?>
```

- `count($images)` — количество элементов массива
- `basename($images[$i])` — извлекает только имя файла из пути (например, из `image/cat.jpg` получаем `cat.jpg`)
- `alt` — атрибут изображения для доступности и SEO

---

## 7. Ответы на контрольные вопросы

### Что такое массивы в PHP?

**Массив** — это упорядоченная структура данных, хранящая одно или несколько значений в одной переменной. Каждое значение имеет свой **ключ** (индекс). PHP поддерживает три вида массивов:

- **Индексированные** — ключи числа (0, 1, 2, ...)
- **Ассоциативные** — ключи строки (`"id"`, `"name"`, ...)
- **Многомерные** — элементами являются другие массивы

### Как создать массив в PHP?

```php
// Способ 1 — современный синтаксис (PHP 5.4+)
$arr = ['apple', 'banana', 'cherry'];

// Способ 2 — через функцию array()
$arr = array('apple', 'banana', 'cherry');

// Способ 3 — ассоциативный массив
$user = ['name' => 'Alice', 'age' => 25];

// Способ 4 — поэлементное добавление
$arr = [];
$arr[] = 'apple';
$arr[] = 'banana';
```

### Для чего используется цикл foreach?

`foreach` предназначен для **перебора всех элементов массива**. Это самый удобный способ итерации в PHP, так как он автоматически проходит по всем элементам без ручного управления индексами.

```php
// Только значения
foreach ($array as $value) {
    echo $value;
}

// Ключ и значение
foreach ($array as $key => $value) {
    echo "$key: $value";
}
```

**Преимущества перед `for`:**
- Не нужно знать размер массива
- Работает с ассоциативными массивами (строковые ключи)
- Код читается проще
- Не выходит за границы массива

---

*Лабораторная работа №4 — Массивы и функции в PHP*
