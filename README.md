# Лабораторная работа по "Базам Данных" №1 и №2


## Лабораторая работа №1


### № 1
2. Найдите все рейсы прибившие в SVO в интервале с 16:00 до 19:00 2017-07-22

**Что делает:**  
Находит все рейсы, которые прибыли в аэропорт SVO 22 июля 2017 года между 16:00 и 19:00, и сортирует их по времени прибытия.

```sql
SELECT
  flights.flight_id,
  flights.flight_no,
  flights.actual_arrival,
  flights.departure_airport AS dep_code
FROM flights
WHERE flights.arrival_airport = 'SVO'
  AND flights.actual_arrival::date = DATE '2017-07-22'
  AND flights.actual_arrival::time > TIME '16:00'
  AND flights.actual_arrival::time <  TIME '19:00'
ORDER BY flights.actual_arrival;
```
**Использованные конструкции SQL:**
- `SELECT … FROM` — базовая выборка.
- `WHERE` — фильтр по аэропорту прибытия и по дате/времени.
- Приведение типов: `timestamp::date`, `timestamp::time`.
- Сравнение c: `DATE '…'`, `TIME '…'`.
- Диапазон по времени: `>` и `<`.
- `ORDER BY` — сортировка результата.

---
### № 2

3. Увеличьте стоимость билетов на рейсы, запланированные на пиковые часы (с 7:00 до 10:00 и с 17:00 до 20:00) на 1000. 

**Что делает:**  
Повышает цену на 1000 ₽ для билетов на рейсы с плановым вылетом в утренний пик 07–09 и вечерний пик 17–19 часов.


```sql
UPDATE ticket_flights
SET amount = amount + 1000
WHERE flight_id IN (
    SELECT f.flight_id
    FROM flights f
    WHERE (EXTRACT(HOUR FROM f.scheduled_departure) BETWEEN 7 AND 9)
        OR (EXTRACT(HOUR FROM f.scheduled_departure) BETWEEN 17 AND 19));

```
**Использованные конструкции SQL:**
- `UPDATE … SET …` — обновление строк.
- `WHERE … IN (SELECT …)` — фильтрация через подзапрос.
- `EXTRACT(HOUR FROM timestamp)` — извлекаем час из даты-времени.
- `BETWEEN … AND …` — диапазон по часу.

---
### № 3

4. Для каждой даты в августе найдите колличество рейсов со статусом 'Delayed'
**Что делает:**  
Считает количество рейсов со статусом `Delayed` по каждой дате в августе 2017 и выводит в порядке дат.


```sql
SELECT
  scheduled_departure::date AS flight_date,
  COUNT(*) AS delayed_count
FROM flights
WHERE scheduled_departure::date BETWEEN DATE '2017-08-01' AND DATE '2017-08-31'
  AND status = 'Delayed'
GROUP BY scheduled_departure::date
ORDER BY flight_date;
```
**Использованные конструкции SQL:**
- Приведение `timestamp::date` для получения календарной даты.
- Фильтр по диапазону дат: `BETWEEN DATE '…' AND DATE '…'`.
- Фильтр по статусу: `status = 'Delayed'`.
- Агрегация: `COUNT(*)`.
- Группировка: `GROUP BY` по дате.
- Сортировка: `ORDER BY`.


## Лабораторая работа №2


### № 1

2. Разработайте систему оптимизации распределения самолетов по маршрутам с учетом. Для каждого маршрута определите оптимальный тип самолета, сравнивая фактически используемый с рекомендуемым. Найдите рейсы, где выгодней всего было бы использовать другой самолет, но чтобы качество услуг не изменилось.

**Что делает:**  
1. Считает вместимость каждого типа самолёта (количество мест в `seats`).  
2. Считает средний спрос по каждому маршруту (из `ticket_flights` на рейс) и добавляет 20% запас.  
3. Для каждого маршрута подбирает минимальный тип борта, у которого мест ≥ требуемому.  
4. Сравнивает с фактическим бортом каждого рейса и показывает, где есть «лишние места» (потенциал поставить борт поменьше).


```sql
-- Вместимость
WITH cap AS (
  SELECT aircraft_code, COUNT(*) AS seats
  FROM seats
  GROUP BY aircraft_code),

-- Кол-во пассажиров на рейсе
pax AS (
  SELECT flight_id, COUNT(*) AS pax_on_flight
  FROM ticket_flights
  GROUP BY flight_id),

-- Средний спрос (dep->arr)
need AS (
  SELECT
    f.departure_airport AS dep,
    f.arrival_airport   AS arr,
    CEIL(AVG(COALESCE(p.pax_on_flight,0)) * 1.2) ::int AS req_seats
  FROM flights f
  LEFT JOIN pax p USING (flight_id)
  GROUP BY f.departure_airport, f.arrival_airport),

-- Рекомендация: минимальный тип с достаточной вместимостью
rec AS (
  SELECT
    n.dep, n.arr, n.req_seats,
    (SELECT c.aircraft_code FROM cap c
     WHERE c.seats >= n.req_seats
     ORDER BY c.seats ASC
     LIMIT 1) AS rec_code
  FROM need n)

-- Итог: где фактический борт крупнее рекомендованного (лишние места)
SELECT
  (SELECT airport_name->>'ru' FROM airports_data WHERE airport_code = f.departure_airport) || ' → ' ||
  (SELECT airport_name->>'ru' FROM airports_data WHERE airport_code = f.arrival_airport)                            AS "Маршрут",
  f.flight_no                                 AS "Рейс",
  f.scheduled_departure                       AS "Вылет (план)",
  f.aircraft_code                             AS "Факт. борт",
  (SELECT seats FROM cap WHERE aircraft_code = f.aircraft_code) AS "Факт. мест",
  r.rec_code                                  AS "Реком. борт",
  (SELECT seats FROM cap WHERE aircraft_code = r.rec_code)      AS "Реком. мест",
  ( (SELECT seats FROM cap WHERE aircraft_code = f.aircraft_code)
  - (SELECT seats FROM cap WHERE aircraft_code = r.rec_code) )  AS "Лишних мест"
FROM flights f
JOIN rec r ON r.dep = f.departure_airport AND r.arr = f.arrival_airport
WHERE (SELECT seats FROM cap WHERE aircraft_code = f.aircraft_code)
    > (SELECT seats FROM cap WHERE aircraft_code = r.rec_code)
ORDER BY "Лишних мест" DESC, "Вылет (план)";
```

**Использованные конструкции SQL:**
- `WITH … AS (…)` — общие табличные выраженияv для пошаговой логики.
- Агрегации: `COUNT(*)`, `AVG(...)`.
- Обработка пустых значений: `COALESCE(...)`.
- Округление вверх: `CEIL(...)`.
- Соединения: `LEFT JOIN … USING (…)`, `JOIN`.
- Коррелированные подзапросы в `SELECT` (подтянуть вместимость/название по коду).
- `ORDER BY … LIMIT` внутри подзапроса для выбора «минимально подходящего» борта.
- Вычисляемые столбцы и форматирование (маршрут `A → B`).
