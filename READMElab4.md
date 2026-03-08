<?php

declare(strict_types=1);

/**
 * Система управления банковскими транзакциями
 * Лабораторная работа — Массивы и функции в PHP
 */

// ==========================================
// Массив транзакций
// ==========================================

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
    [
        "id"          => 3,
        "date"        => "2021-06-10",
        "amount"      => 250.00,
        "description" => "Online course subscription",
        "merchant"    => "Udemy",
    ],
    [
        "id"          => 4,
        "date"        => "2022-11-23",
        "amount"      => 35.99,
        "description" => "Movie tickets",
        "merchant"    => "CinemaCity",
    ],
    [
        "id"          => 5,
        "date"        => "2023-03-07",
        "amount"      => 520.00,
        "description" => "Flight ticket",
        "merchant"    => "AirBaltic",
    ],
];

// ==========================================
// Функции
// ==========================================

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
        if (stripos($t['description'], $descriptionPart) !== false) {
            return $t;
        }
    }
    return null;
}

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

/**
 * Ищет транзакцию по ID с помощью array_filter.
 *
 * @param int $id Идентификатор транзакции
 * @return array|null Найденная транзакция или null
 */
function findTransactionByIdFilter(int $id): ?array
{
    global $transactions;

    // array_filter возвращает массив подходящих элементов
    $result = array_filter($transactions, function ($t) use ($id) {
        return $t['id'] === $id;
    });

    // array_values сбрасывает ключи, берём первый элемент
    $result = array_values($result);
    return $result[0] ?? null;
}

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

// ==========================================
// Добавляем новую транзакцию
// ==========================================

addTransaction(6, "2024-07-19", 89.90, "Gym membership", "FitLife");

// ==========================================
// Сортировка по дате (usort + DateTime)
// ==========================================

usort($transactions, function (array $a, array $b): int {
    return strcmp($a['date'], $b['date']);
});

// ==========================================
// Сортировка по сумме (по убыванию)
// ==========================================

usort($transactions, function (array $a, array $b): int {
    // Сравниваем суммы в обратном порядке
    if ($a['amount'] === $b['amount']) return 0;
    return $a['amount'] < $b['amount'] ? 1 : -1;
});

// ==========================================
// Пример поиска
// ==========================================

$found = findTransactionByDescription("groceries");
$foundById = findTransactionByIdFilter(3);

?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Банковские транзакции</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; background: #f4f4f4; }
        h1 { color: #333; }
        table { border-collapse: collapse; width: 100%; background: #fff; }
        th { background-color: #4a90d9; color: white; padding: 10px; }
        td { padding: 8px 12px; border: 1px solid #ccc; }
        tr:nth-child(even) { background-color: #f0f0f0; }
        .total-row { font-weight: bold; background-color: #d4edda; }
        .info-block { margin-top: 30px; background: #fff; padding: 15px; border-left: 4px solid #4a90d9; }
    </style>
</head>
<body>

<h1>Список банковских транзакций</h1>

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
        <tr class="total-row">
            <td colspan="2">Итого:</td>
            <td><?= number_format(calculateTotalAmount($transactions), 2) ?></td>
            <td colspan="3"></td>
        </tr>
    </tbody>
</table>

<!-- Результаты поиска -->
<div class="info-block">
    <h3>Поиск по описанию: "groceries"</h3>
    <?php if ($found): ?>
        <p>Найдено: <strong><?= htmlspecialchars($found['description']) ?></strong>
           — <?= htmlspecialchars($found['merchant']) ?>,
           <?= number_format($found['amount'], 2) ?> $</p>
    <?php else: ?>
        <p>Не найдено.</p>
    <?php endif; ?>

    <h3>Поиск по ID (array_filter): ID = 3</h3>
    <?php if ($foundById): ?>
        <p>Найдено: <strong><?= htmlspecialchars($foundById['description']) ?></strong>
           — <?= htmlspecialchars($foundById['merchant']) ?>,
           <?= number_format($foundById['amount'], 2) ?> $</p>
    <?php else: ?>
        <p>Не найдено.</p>
    <?php endif; ?>
</div>

</body>
</html>
