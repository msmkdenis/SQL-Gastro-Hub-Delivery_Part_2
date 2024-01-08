### Cервис доставки еды Gastro Hub Delivery - Part 2

В файле ниже — пользовательские скрипты, которые выполняются на базе данных. 
Выполните их на своём компьютере. 
Проверьте, что в вашей СУБД включён модуль pg_stat_statements — это обязательное условие.

Результат решения - скрипт sql, выполнение которого должно происходить без ошибок.

Полное решение доступно в файле [sprint_4_1.sql](https://github.com/msmkdenis/SQL-Gastro-Hub-Delivery_Part_1/blob/main/sprint_4_1.sql).
Ниже приведено решение задач по оптимизации запросов

#### Задание 1
Клиенты сервиса начали замечать, что после нажатия на кнопку `Оформить заказ` система на какое-то время подвисает.
Вот команда для вставки данных в таблицу orders, которая хранит общую информацию о заказах:
```sql
INSERT INTO orders
    (order_id, order_dt, user_id, device_type, city_id, total_cost, discount, 
    final_cost)
SELECT MAX(order_id) + 1, current_timestamp, 
    '329551a1-215d-43e6-baee-322f2467272d', 
    'Mobile', 1, 1000.00, null, 1000.00
FROM orders;
```
Проблема:
- Запрос на вставку вызывает сканирование всех строк таблицы orders
- Избыточные индексы на таблицу orders

Решение
1) Необходимо оптимизировать индексы для таблицы orders:
    - Вставка новых значений требует регулярной перестройки индексов.
    - Некоторые индексы явно не релевантны (null значения, повторяющиеся значения)
    - Некоторые индексы явно избыточны (индекс на timestamp создания заказа)
    - Оставим только востребованные индексы: наиболее частый вариант поиска по таблице
      представляется по полям order_id, user_id.
```sql
drop index if exists
    orders_city_id_idx,
    orders_device_type_city_id_idx,
    orders_device_type_idx,
    orders_discount_idx,
    orders_final_cost_idx,
    orders_final_cost_idx,
    orders_order_dt_idx,
    orders_total_cost_idx,
    orders_total_final_cost_discount_idx;
```
2) Необходимо создать sequence на поле order_id чтобы избавиться от ручного расчета
   который вызывает сканирование всей таблицы для поиска max значения
```sql
create sequence if not exists orders_id_seq
    increment by 1
    owned by public.orders.order_id;

select setval('orders_id_seq', (select max(order_id) from orders));

alter table public.orders
alter column order_id set default nextval('orders_id_seq');
```
3) Также добавим default значения для поля order_dt
```sql
alter table public.orders
alter column order_dt set default current_timestamp;
```
4) Скорректируем запрос: удалим select и значения, который будут вставляться автоматически (order_id, order_dt)
   В результате запрос на вставку не будет как ранее последовательно сканировать все строки таблицы orders
   Проверка
```sql
begin;
explain analyse verbose
INSERT INTO orders
    (user_id, device_type, city_id, total_cost, discount, final_cost)
values
    ('329551a1-215d-43e6-baee-322f2467272d', 'Mobile', 1, 1000.00, null, 1000.00);
rollback;
```

#### Задание 2
Клиенты сервиса в свой день рождения получают скидку. Расчёт скидки и отправка клиентам промокодов происходит на стороне сервера приложения.
Каждый раз список именинников формируется и возвращается недостаточно быстро. Оптимизируйте этот процесс.
Список клиентов возвращается из БД в приложение таким запросом:
```sql
SELECT user_id::text::uuid, first_name::text, last_name::text, 
    city_id::bigint, gender::text
FROM users
WHERE city_id::integer = 4
    AND date_part('day', to_date(birth_date::text, 'yyyy-mm-dd')) 
        = date_part('day', to_date('31-12-2023', 'dd-mm-yyyy'))
    AND date_part('month', to_date(birth_date::text, 'yyyy-mm-dd')) 
        = date_part('month', to_date('31-12-2023', 'dd-mm-yyyy'))
```
Проблема:
- Неэффективные типы данных в таблице users
- Последовательное сканирование таблицы users

Решение
1) Оптимизируем типы данных в таблице users
```sql
ALTER TABLE users ALTER COLUMN user_id type uuid using user_id::text::uuid;
alter table users alter column  first_name type text;
alter table users alter column  last_name type text;
alter table users alter column  city_id type int;
alter table users alter column  gender type text;
alter table users alter column birth_date type timestamp using to_timestamp(birth_date, 'YYYY-MM-DD HH24:MI:SS');
alter table users alter column registration_date type timestamp without time zone using to_timestamp(registration_date, 'YYYY-MM-DD HH24:MI:SS');
```
2) Создадим индекс для более эффективного поиска по городам
```sql
create index users_city_id_idx on users (city_id);
```
3) Скорректируем запрос с учетом выполненных улучшений, избавившись от лишнего приведения типов
   Согласно плану запросов общая эффективность запроса увеличилась более чем в 4 раза
```sql
SELECT
    user_id, first_name, last_name, city_id, gender
FROM users
WHERE city_id::integer = 4
  AND date_part('day', birth_date::date)
    = date_part('day', to_date('31-12-2023', 'dd-mm-yyyy'))
  AND date_part('month', birth_date::date)
    = date_part('month', to_date('31-12-2023', 'dd-mm-yyyy'));
```

#### Задание 3
Также пользователи жалуются, что оплата при оформлении заказа проходит долго.
Разработчик сервера приложения Матвей проанализировал ситуацию и заключил, что оплата «висит» из-за того, что выполнение процедуры `add_payment` требует довольно много времени по меркам БД.
Найдите в базе данных эту процедуру и подумайте, как можно ускорить её работу.

Проблема:
- В составе процедуры add_payment неэффективно выполняется вставка в таблицу sales.
- Время планирования запроса больше выполнения
- Таблица Payments фактически не выполняет никаких полезных функций

Решение
1) Создадим покрывающий индекс
```sql
create index orders_order_id_user_id ON orders(order_id) include (order_id);
```
Согласно обновленному плану запроса мы добились index only scan по таблице orders,
стоимость запрос сократилась ~ в 2 раза

2) Удалим вставку в таблицу payments из процедуры
    - В таблице payments хранятся данные о номере заказа и сумме платежа.
    - Эти же данные хранятся в таблице orders в колонка order_id и final_cost
    - Таблица payments фактически полностью дублирует таблицу orders (являясь её частью)
    - Технически ее можно удалить проверив не нарушаются ли где-то связи.
```sql
create or replace procedure add_payment(IN p_order_id bigint, IN p_sum_payment numeric)
    language plpgsql
as
$$BEGIN
    INSERT INTO order_statuses (order_id, status_id, status_dt)
    VALUES (p_order_id, 2, statement_timestamp());

    INSERT INTO sales(sale_id, sale_dt, user_id, sale_sum)
    SELECT NEXTVAL('sales_sale_id_sq'), statement_timestamp(), user_id, p_sum_payment
    FROM orders WHERE order_id = p_order_id;
END;$$;

drop table if exists payments;
```

#### Задание 4
Все действия пользователей в системе логируются и записываются в таблицу `user_logs`. Потом эти данные используются для анализа — как правило, анализируются данные за текущий квартал.
Время записи данных в эту таблицу сильно увеличилось, а это тормозит практически все действия пользователя. Подумайте, как можно ускорить запись. Вы можете сдать решение этой задачи без скрипта или — попробовать написать скрипт. Дерзайте!

Проблема:
- Большая таблица логов, обращение с таблицей занимает большое кол-во времени

Предполагаемое решение:
- Осуществим партицирование по датам логирования через наследование имеющейся таблицы.
- Создадим функцию, автоматически создающую квартальные партиции

ВНИМАНИЕ! Скипт может выполняться определенное время - на моем компьютере заняло около 3,5 минут.
```sql
alter table user_logs
drop constraint user_logs_pkey;

alter table user_logs
add constraint user_logs_pkey primary key (log_id, log_date);

create or replace function insert_user_logs()
    returns trigger as $$
declare
    current_date_part date;
    quarter_begin_month_text text;
    quarter_end_month_text text;
    partition_table_name text;
    first_day_of_quarter text;
    last_day_of_quarter text;
BEGIN
    current_date_part := cast(date_trunc('quarter', NEW.log_date) as date);
    quarter_begin_month_text := regexp_replace(current_date_part::text, '-','_','g');
    quarter_end_month_text := regexp_replace(cast(date_trunc('quarter', current_date_part + '3 month'::interval) as date)::text, '-','_','g');
    -- сформируем название новой партицированной таблицы (с учетом даты и квартала)
    partition_table_name := format('user_logs_%s_%s', quarter_begin_month_text::text, quarter_end_month_text::text);
    -- если партиции нет - создадим новую
    IF (to_regclass(partition_table_name::text) isnull) then
        first_day_of_quarter := current_date_part;
        last_day_of_quarter := current_date_part + '3 month'::interval;
        execute format(
                'create table %I ('
                    '  check (log_date >= date %L and log_date < date %L)'
                    ') inherits (user_logs);'
            , partition_table_name, first_day_of_quarter, last_day_of_quarter);
        -- добавим к партиции соответствующий первичный ключ
        execute format(
                'alter table only %1$I add constraint %1$s__pkey primary key (log_id, log_date);'
            , partition_table_name);
        -- добавим индекс для более эффективного поиска по датам
        execute format(
                'create index %1$s__log_date_idx on %1$I (log_date);'
            , partition_table_name);
    end if;
    -- выполним вставку в новую партицию или уже имеющуюся (если партиция не создавалась)
    execute format('insert into %I values ($1.*)', partition_table_name) using new;

    return null;
END;
$$
    language plpgsql;

create trigger insert_bigtable
before insert on user_logs
for each row execute function insert_user_logs();

begin;
create temp table if not exists temp_user_logs on commit drop as
select * from user_logs;
delete from only user_logs;
insert into user_logs select * from temp_user_logs;
truncate only user_logs;
commit;

-- судя по планам запросов партицирование удалось - происходит index scan по нужной партиции
explain analyse verbose
select * from user_logs where log_date = '2021-06-01';

explain analyse verbose
select * from user_logs where log_date = '2021-02-01';

-- в качестве теста вставим 4 записи - создастся 4 партиции на 2023 год поквартально
insert into user_logs
values
    ('622843e4-53c0-4b6c-b89f-d856efc9db4a', 'e695c709-c780-41cb-a114-7486c206cf44', 'order', '2023-01-01 23:45:07.000000', '2023-01-01'),
    ('622843e4-53c0-4b6c-b89f-d856efc9db4a', 'e695c709-c780-41cb-a114-7486c206cf44', 'order', '2023-04-01 23:45:07.000000', '2023-04-01'),
    ('622843e4-53c0-4b6c-b89f-d856efc9db4a', 'e695c709-c780-41cb-a114-7486c206cf44', 'order', '2023-07-01 23:45:07.000000', '2023-07-01'),
    ('622843e4-53c0-4b6c-b89f-d856efc9db4a', 'e695c709-c780-41cb-a114-7486c206cf44', 'order', '2023-10-01 23:45:07.000000', '2023-10-01');

-- поиск осуществляется по партиции
explain analyse verbose
select * from user_logs where log_date = '2023-04-01';
```

#### Задание 5
Маркетологи сервиса регулярно анализируют предпочтения различных возрастных групп.
В столбцах spicy, fish и meat отображается, какой % блюд, заказанных каждой категорией пользователей, содержал эти признаки.
В возрастных интервалах верхний предел входит в интервал, а нижний — нет.
Также по правилам построения отчётов в них не включается текущий день.
Администратор БД Серёжа заметил, что регулярные похожие запросы от разных маркетологов нагружают базу, и в результате увеличивается время работы приложения.
Подумайте с точки зрения производительности, как можно оптимально собирать и хранить данные для такого отчёта. В ответе на это задание не пишите причину — просто опишите ваш способ получения отчёта и добавьте соответствующий скрипт.

Проблема:
- Регулярное создание отчета, обращающегося к большому кол-ву статистических данных
- Решение

Т.к. в отчет не включается текущий день создадим materialized view, который можно будет обновлять утром следующего дня.
```sql
create materialized view if not exists statistics_dishes_age as
with
    statistics as (
        select
            od.order_dt::date as date,
            oi.count as dish_count,
            di.fish * oi.count as fish,
            di.spicy * oi.count as spicy,
            di.meat * oi.count as meat,
            extract(year from od.order_dt::date) - extract(year from us.birth_date::date) as raw_age,
            case
                when extract(year from od.order_dt::date) - extract(year from us.birth_date::date) > 0
                    and extract(year from od.order_dt::date) - extract(year from us.birth_date::date) <= 20 then '0 - 20'
                when extract(year from od.order_dt::date) - extract(year from us.birth_date::date) > 20
                    and extract(year from od.order_dt::date) - extract(year from us.birth_date::date) <= 30 then '20 - 30'
                when extract(year from od.order_dt::date) - extract(year from us.birth_date::date) > 30
                    and extract(year from od.order_dt::date) - extract(year from us.birth_date::date) <= 40 then '30 - 40'
                when extract(year from od.order_dt::date) - extract(year from us.birth_date::date) > 40
                    and extract(year from od.order_dt::date) - extract(year from us.birth_date::date) <= 100 then '40 - 100'
                end as age
        from
            order_items oi
                left join dishes di on oi.item = di.object_id
                left join orders od on oi.order_id = od.order_id
                left join users us on od.user_id = us.user_id)
select
    date,
    round(sum(fish)::numeric/sum(dish_count)*100) as fish,
    round(sum(spicy)::numeric/sum(dish_count)*100) as spicy,
    round(sum(meat)::numeric/sum(dish_count)*100) as meat,
    age
from statistics
group by date, age
order by date desc, age;
```
