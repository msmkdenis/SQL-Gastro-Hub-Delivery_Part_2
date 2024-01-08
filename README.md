### Cервис доставки еды Gastro Hub Delivery - Part 2

В файле ниже — пользовательские скрипты, которые выполняются на базе данных. 
Выполните их на своём компьютере. 
Проверьте, что в вашей СУБД включён модуль pg_stat_statements — это обязательное условие.

Результат решения - скрипт sql, выполнение которого должно происходить без ошибок.

Полное решение доступно в файле [sprint_4_2.sql](https://github.com/msmkdenis/SQL-Gastro-Hub-Delivery_Part_2/blob/main/sprint_4_2.sql).
Ниже приведено решение задач по оптимизации запросов

Определим id базы данных для выборки медленных запросов
```sql
select oid, datname from pg_database;
```

Определим 5 самых медленных запросов
```sql
select
    query,
    ROUND(mean_exec_time::numeric,2),
    ROUND(total_exec_time::numeric,2),
    ROUND(min_exec_time::numeric,2),
    ROUND(max_exec_time::numeric,2),
    calls,
    rows
from pg_stat_statements
-- Подставьте своё значение dbid.
where dbid = 59856 order by mean_exec_time desc
limit 5;
```

5 самых медленных запросов и их модификация:


### #1 Из 5 самых медленных запросов

определяет количество неоплаченных заказов
```sql
explain analyse verbose
SELECT count(*)
FROM order_statuses os
         JOIN orders o ON o.order_id = os.order_id
WHERE (SELECT count(*)
       FROM order_statuses os1
       WHERE os1.order_id = o.order_id AND os1.status_id = 2) = 0
  AND o.city_id = 1;
```

Проблема - медленный nested loop, неэффективный запрос, недостаток индексов

Добавим индексы
```sql
create index if not exists orders_statuses_status_id_idx on order_statuses (status_id);
create index if not exists  orders_city_id_idx on orders (city_id);
```

Переработаем запрос, избавимся от nested loop.
```sql
explain analyse verbose
with not_paid as (
    select
        order_id
    from order_statuses
    where order_statuses.order_id not in (
        select
            order_id
        from order_statuses
        where status_id = 2))
select
    count(*)
from not_paid np
    left join orders o on o.order_id = np.order_id
where o.city_id = 1;
```
- Время выполнения уменьшилось ~ в 2_100 раз.


### #2 Из 5 самых медленных запросов

ищет логи за текущий день
```sql
explain analyse verbose
select *
from user_logs
where datetime = current_date;
```

Проблема - отсутствие индексов в партициях: последовательное сканирование каждой партиции
```sql
create index if not exists  user_logs_y2021q2_datetime_idx on user_logs_y2021q2 (datetime);
create index if not exists  user_logs_y2021q3_datetime_idx on user_logs_y2021q3 (datetime);
create index if not exists  user_logs_y2021q4_datetime_idx on user_logs_y2021q4 (datetime);

explain analyse verbose
select *
from user_logs
where datetime = current_date;
```
- Время выполнения уменьшилось ~ в 4_700 раз (благодаря индексам быстро получен ответ - что логи не найдены).

- К партициям применяется сканирование по индексу


### #3 Из 5 самых медленных запросов

ищет действия и время действия определенного посетителя
```sql
explain analyse verbose
select event, datetime
from user_logs
where visitor_uuid = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'
order by 2;
```

Проблема - отсутствие индексов в партициях: последовательное сканирование каждой партиции по visitor_uuid

```sql
create index if not exists  user_logs_visitor_uuid_idx on user_logs ((visitor_uuid::text));
create index if not exists  user_logs_y2021q2_visitor_uuid_idx on user_logs_y2021q2 ((visitor_uuid::text));
create index if not exists  user_logs_y2021q3_visitor_uuid_idx on user_logs_y2021q3 ((visitor_uuid::text));
create index if not exists  user_logs_y2021q4_visitor_uuid_idx on user_logs_y2021q4 ((visitor_uuid::text));

explain analyse verbose
select event, datetime
from user_logs
where visitor_uuid = 'fd3a4daa-494a-423a-a9d1-03fd4a7f94e0'
order by 2;
```

- Время выполнения уменьшилось ~ в 2_100 раз.

- К партициям применяется Bitmap Index Scan


### #4 Из 5 самых медленных запросов

выводит данные о конкретном заказе: id, дату, стоимость и текущий статус
```sql
explain analyse verbose
select o.order_id, o.order_dt, o.final_cost, s.status_name
from order_statuses os
         join orders o on o.order_id = os.order_id
         join statuses s on s.status_id = os.status_id
where o.user_id = 'c2885b45-dddd-4df3-b9b3-2cc012df727c'::uuid
  and os.status_dt in (
    select max(status_dt)
    from order_statuses
    where order_id = o.order_id
);
```

Nested loop в запросе.

Отсутствует индекс в order_statuses
```sql
create index if not exists  order_statuses_order_id_status_id_idx on order_statuses (order_id, status_id);

explain analyse verbose
with orders_by_user as(
    select
        o.order_id,
        o.order_dt,
        os.status_dt,
        o.final_cost,
        os.status_id,
        rank() over (partition by o.order_id order by os.status_dt desc)
    from orders o
             left join order_statuses os on o.order_id = os.order_id
    where user_id = 'c2885b45-dddd-4df3-b9b3-2cc012df727c'::uuid
    order by order_id, os.status_dt)
select
    obu.order_id,
    obu.order_dt,
    obu.final_cost,
    s.status_name
from orders_by_user obu
         left join statuses s on obu.status_id = s.status_id
where obu.rank = 1;
```

- Время выполнения уменьшилось ~ в 800 раз. (Упростили запрос, применяется индексное сканирование)


### #5 Из 5 самых медленных запросов

вычисляет количество заказов позиций, продажи которых выше среднего
```sql
explain analyse verbose
select
    d.name,
    SUM(count) as orders_quantity
from order_items oi
join dishes d on d.object_id = oi.item
where oi.item in (
    select item
    from (select item, SUM(count) AS total_sales
          from order_items oi
          group by 1) dishes_sales
    where dishes_sales.total_sales > (
        select SUM(t.total_sales)/ COUNT(*)
        from (select item, SUM(count) as total_sales
              from order_items oi
              group by
                  1) t)
)
group by 1
order by orders_quantity desc;
```

Необходимо устранить повторения, воспользуемся cte

Добавим индекс

```sql
create index if not exists  order_items_item_idx on order_items (item);
create index if not exists  dishes_object_id_idx on dishes (object_id);

explain analyse verbose
with statistics as (
    select
        item,
        SUM(count) AS total_sales
    from order_items oi
    group by 1
)
select
    d.name,
    SUM(count) as orders_quantity
from order_items oi
join dishes d on d.object_id = oi.item
where oi.item in (
    select item
    from statistics
    where statistics.total_sales > (
        select SUM(total_sales)/ COUNT(*)
        from statistics)
)
group by 1
order by orders_quantity desc;
```
- Время выполнения уменьшилось ~ в 1.5 раза.
