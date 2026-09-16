# Кейс: отчёт по продажам с иерархией итогов (ROLLUP / CUBE)

## Схема

```sql
CREATE TABLE sales (
    sale_id     integer PRIMARY KEY,
    sale_date   date,
    region      varchar,
    category    varchar,
    product     varchar,
    amount      numeric
);
```

## Тестовые данные

```sql
INSERT INTO sales VALUES
(1, '2026-01-15', 'North', 'Electronics', 'Phone',  1200),
(2, '2026-01-20', 'North', 'Electronics', 'Laptop', 2500),
(3, '2026-02-05', 'North', 'Food',        'Coffee',  300),
(4, '2026-01-10', 'South', 'Electronics', 'Phone',  1000),
(5, '2026-02-12', 'South', 'Food',        'Coffee',  450),
(6, '2026-02-18', 'South', 'Food',        'Tea',     200);
```

## Задания

**Задание 1.** Написать запрос, который одним проходом выдаёт: сумму продаж по каждой паре (region, category), подытог по region, и общий итог — без UNION ALL.

**Задание 2.** Добавить флаговые колонки, которые явно помечают уровень строки: `is_region_total` и `is_grand_total` (boolean), используя `GROUPING()`.

**Задание 3.** Отсортировать так, чтобы в каждом регионе сначала шли детальные строки, потом подытог, а общий итог был в самом конце.

**Задание 4.** Модифицировать запрос так, чтобы получить ещё и разрез по category отдельно (без привязки к region) — тем самым перейти от иерархии к полной кросс-табуляции. Что меняется в конструкции GROUP BY и почему вырастет число строк?

**Задание 5 (на понимание, устно).** Почему `WHERE amount > 500` и `HAVING SUM(amount) > 500` в этом запросе дадут разный результат для строк-подытогов? В какой момент выполнения запроса WHERE успевает отработать относительно ROLLUP?

---

## Решение

### Задания 1–3

```sql
SELECT
    region,
    category,
    SUM(amount) AS total,
    GROUPING(region) = 1 AS is_grand_total,
    GROUPING(category) = 1 AND GROUPING(region) = 0 AS is_region_total
FROM sales
GROUP BY ROLLUP(region, category)
ORDER BY
    GROUPING(region),
    region,
    GROUPING(category),
    category;
```

### Задание 4

```sql
GROUP BY CUBE(region, category)
```

Число групп растёт: ROLLUP даёт N+1 уровней (линейный рост по иерархии), CUBE даёт 2^k комбинаций при k колонках — появляются строки вида `(region=NULL, category='Electronics')`, которых ROLLUP никогда не даёт, потому что он не "срезает" среднюю колонку, оставляя крайние.

### Задание 5

`WHERE` отрабатывает до агрегации — фильтрует сырые строки таблицы, поэтому строка с `amount=300` (Coffee, North) вообще не попадёт в агрегацию, и подытог по North это учтёт (станет меньше на 300).

`HAVING SUM(amount) > 500` отрабатывает после того, как ROLLUP уже посчитал все уровни — режет по итоговой сумме группы, не трогая исходные строки для остальных подытогов.
