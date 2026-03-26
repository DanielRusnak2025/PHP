# Лабораторная работа №5 — ООП в PHP

---

## declare(strict_types=1)

Включает строгую типизацию. Если функция ожидает `int`, а передать строку — будет ошибка. Без этого PHP молча преобразует типы сам, что может привести к багам.

---

## Интерфейс TransactionStorageInterface

```php
interface TransactionStorageInterface
{
    public function addTransaction(Transaction $transaction): void;
    public function removeTransactionById(int $id): void;
    public function getAllTransactions(): array;
    public function findById(int $id): ?Transaction;
}
```

Интерфейс — это список методов, которые **обязан** реализовать любой класс, использующий этот интерфейс. Сам интерфейс не содержит никакой логики — только сигнатуры методов.

Зачем нужен: если завтра понадобится хранить транзакции не в массиве, а в базе данных — создаём новый класс, реализуем тот же интерфейс, и `TransactionManager` ничего не знает об этой замене и продолжает работать.

---

## Класс Transaction

```php
class Transaction
{
    public function __construct(
        private int    $id,
        private string $date,
        private float  $amount,
        private string $description,
        private string $merchant,
    ) {}
}
```

Описывает одну транзакцию. Все свойства **приватные** — снаружи их прочитать напрямую нельзя, только через геттеры.

`private` в конструкторе — это сокращённый синтаксис PHP 8: свойство сразу объявляется и заполняется через конструктор, без отдельного `$this->id = $id`.

### Геттеры

```php
public function getId(): int      { return $this->id; }
public function getDate(): string { return $this->date; }
// и т.д.
```

Просто возвращают приватное свойство. Нужны потому что прямой доступ `$transaction->id` запрещён из-за `private`.

### Метод getDaysSinceTransaction

```php
public function getDaysSinceTransaction(): int
{
    $transactionDate = new DateTime($this->date);
    $today = new DateTime('today');
    return (int) $today->diff($transactionDate)->days;
}
```

Создаём два объекта DateTime — дату транзакции и сегодня. `->diff()` считает разницу между ними и возвращает объект `DateInterval`, у которого есть свойство `->days` с количеством дней.

---

## Класс TransactionRepository

```php
class TransactionRepository implements TransactionStorageInterface
{
    private array $transactions = [];
    ...
}
```

`implements TransactionStorageInterface` — класс обязуется реализовать все методы из интерфейса. Если хоть один метод не написан — PHP выдаст ошибку.

Хранит массив транзакций. Он приватный — снаружи к нему не добраться, только через методы.

### addTransaction

```php
public function addTransaction(Transaction $transaction): void
{
    $this->transactions[] = $transaction;
}
```

Добавляет объект `Transaction` в конец приватного массива.

### removeTransactionById

```php
public function removeTransactionById(int $id): void
{
    foreach ($this->transactions as $key => $t) {
        if ($t->getId() === $id) {
            unset($this->transactions[$key]);
            $this->transactions = array_values($this->transactions);
            return;
        }
    }
}
```

Ищем нужную транзакцию по ключу массива. `unset` удаляет элемент, но оставляет дыры в индексах (0, 2, 3...). `array_values` пересчитывает индексы обратно (0, 1, 2...).

### findById

```php
public function findById(int $id): ?Transaction
{
    foreach ($this->transactions as $t) {
        if ($t->getId() === $id) {
            return $t;
        }
    }
    return null;
}
```

Перебираем все транзакции, сравниваем ID. Нашли — возвращаем. Не нашли — возвращаем `null`. `?Transaction` означает "либо объект Transaction, либо null".

---

## Класс TransactionManager

```php
class TransactionManager
{
    public function __construct(
        private TransactionStorageInterface $repository
    ) {}
}
```

Получает хранилище через конструктор — это называется **внедрение зависимостей**. `TransactionManager` не создаёт транзакции сам и не хранит их — он только выполняет операции над данными из репозитория.

Принимает именно интерфейс `TransactionStorageInterface`, а не конкретный класс — поэтому не важно, какое именно хранилище передать.

### calculateTotalAmount

```php
public function calculateTotalAmount(): float
{
    $total = 0.0;
    foreach ($this->repository->getAllTransactions() as $t) {
        $total += $t->getAmount();
    }
    return $total;
}
```

Берём все транзакции из репозитория и суммируем их поле `amount`.

### calculateTotalAmountByDateRange

```php
public function calculateTotalAmountByDateRange(string $startDate, string $endDate): float
{
    $total = 0.0;
    $start = new DateTime($startDate);
    $end   = new DateTime($endDate);

    foreach ($this->repository->getAllTransactions() as $t) {
        $date = new DateTime($t->getDate());
        if ($date >= $start && $date <= $end) {
            $total += $t->getAmount();
        }
    }
    return $total;
}
```

Создаём объекты DateTime для начала и конца периода. Для каждой транзакции создаём DateTime из её даты и проверяем, попадает ли она в диапазон. Объекты DateTime можно сравнивать операторами `>=` и `<=`.

### countTransactionsByMerchant

```php
public function countTransactionsByMerchant(string $merchant): int
{
    $count = 0;
    foreach ($this->repository->getAllTransactions() as $t) {
        if (stripos($t->getMerchant(), $merchant) !== false) {
            $count++;
        }
    }
    return $count;
}
```

`stripos` ищет строку без учёта регистра. Если находит — увеличиваем счётчик.

### sortTransactionsByDate и sortTransactionsByAmountDesc

```php
public function sortTransactionsByDate(): array
{
    $transactions = $this->repository->getAllTransactions();
    usort($transactions, function (Transaction $a, Transaction $b): int {
        return strcmp($a->getDate(), $b->getDate());
    });
    return $transactions;
}
```

Получаем копию массива из репозитория и сортируем её через `usort`. Оригинальный массив в репозитории остаётся нетронутым. `strcmp` сравнивает строки — даты в формате `YYYY-MM-DD` сортируются корректно как строки.

Для сортировки по сумме по убыванию: если `$a->getAmount()` меньше — возвращаем `1` (ставим `$a` после `$b`), что даёт обратный порядок.

---

## Класс TransactionTableRenderer

```php
final class TransactionTableRenderer
{
    public function render(array $transactions): string
    {
        $html = '<table border="1">';
        // ...
        foreach ($transactions as $t) {
            $html .= '<tr>';
            $html .= '<td>' . htmlspecialchars((string)$t->getId()) . '</td>';
            // ...
        }
        $html .= '</tbody></table>';
        return $html;
    }
}
```

`final` — класс нельзя наследовать. Используется здесь потому что рендерер не предназначен для расширения.

Метод `render` принимает массив транзакций, собирает HTML-строку через конкатенацию и возвращает её. В основном файле остаётся только `echo $renderer->render(...)`.

`htmlspecialchars` защищает от XSS — преобразует `<`, `>`, `&` в безопасные символы.

---

## Начальные данные

```php
$repository = new TransactionRepository();
$repository->addTransaction(new Transaction(1, "2019-01-01", 100.00, "Payment for groceries", "SuperMart"));
// ...
```

Создаём объект репозитория. Каждую транзакцию создаём через `new Transaction(...)` и сразу передаём в `addTransaction`.

```php
$manager  = new TransactionManager($repository);
$renderer = new TransactionTableRenderer();
```

Передаём репозиторий в менеджер через конструктор. Рендерер создаём отдельно.

---

## Ответы на контрольные вопросы

### 1. Зачем нужна строгая типизация?

`declare(strict_types=1)` заставляет PHP строго проверять типы аргументов. Без неё PHP молча приводит типы: например, строка `"5"` становится числом `5`. Это удобно, но создаёт трудноуловимые баги. Со строгой типизацией такая передача вызовет ошибку сразу — проще найти проблему.

### 2. Что такое класс и его основные компоненты?

Класс — это шаблон (blueprint) для создания объектов. Он описывает, какие данные хранит объект и что умеет делать.

Основные компоненты:
- **Свойства** — переменные внутри класса (`private int $id`)
- **Конструктор** — метод `__construct`, вызывается при создании объекта через `new`
- **Методы** — функции внутри класса (`getDate()`, `getDaysSinceTransaction()`)
- **Модификаторы доступа** — `private`, `protected`, `public` — определяют, кто может обращаться к свойству или методу

### 3. Что такое полиморфизм?

Полиморфизм — возможность работать с разными классами через одинаковый интерфейс. В PHP реализуется через интерфейсы и наследование.

Пример из лабораторной: `TransactionManager` принимает `TransactionStorageInterface`. Можно создать `DatabaseRepository` и `FileRepository`, оба реализуют интерфейс — и `TransactionManager` будет работать с любым из них без изменений.

### 4. Чем интерфейс отличается от абстрактного класса?

| | Интерфейс | Абстрактный класс |
|---|---|---|
| Содержит логику | Нет | Может содержать |
| Свойства | Нет | Может иметь |
| Наследование | Класс может реализовать несколько | Только один |
| Ключевое слово | `implements` | `extends` |

Интерфейс — чистый контракт "что должно быть". Абстрактный класс — частичная реализация "что уже есть и что надо дописать".

### 5. Какие преимущества даёт интерфейс в этой лабораторной?

В лабораторной `TransactionManager` работает с `TransactionStorageInterface`, а не с конкретным `TransactionRepository`. Это значит:

- Можно создать `DatabaseTransactionRepository`, реализующий тот же интерфейс, и передать его в `TransactionManager` без изменения кода менеджера
- Легко тестировать — можно подставить фиктивный класс-заглушку вместо реального репозитория
- Код становится независимым от конкретной реализации хранилища

---

*Лабораторная работа №5 — ООП в PHP*
