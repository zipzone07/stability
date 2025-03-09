### **Продвинутый SQL**
Проект состоит из двух частей:
- В первой части решим несколько задач.
- Вторая часть проекта — аналитическая.

Будем изучать базу данных [StackOverflow](https://www.stackoverflow.com/) — сервиса вопросов и ответов о программировании. В базе хранятся данные о постах за 2008 год.
#### **Часть 1**
##### **Задача 1** 
Найдем количество вопросов, которые набрали больше 300 очков или как минимум 100 раз были добавлены в «Закладки».
```sql
SELECT COUNT(*)
FROM stackoverflow.posts
WHERE post_type_id = 1
AND (score > 300 OR favorites_count >= 100)
```

|count|
|---|
|1355|
##### **Задача 2** 
Сколько в среднем в день задавали вопросов с 1 по 18 ноября 2008 включительно? Результат округим до целого числа.
```sql
SELECT
	ROUND(AVG(daily_question_count)) AS average_questions_per_day
FROM (
	SELECT
		EXTRACT(day FROM creation_date) AS day,
		COUNT(id) AS daily_question_count
	FROM stackoverflow.posts
	WHERE creation_date BETWEEN '2008-11-01' AND '2008-11-19'
	AND post_type_id = 1 -- Предполагаем, что 1 соответствует вопросам
	GROUP BY day
) AS daily_counts;
```

|average_questions_per_day|
|---|
|383|
##### **Задача 3** 
Сколько пользователей получили значки сразу в день регистрации? Выведем количество уникальных пользователей.
```sql
SELECT 
    COUNT(DISTINCT u.id) AS unique_users
FROM 
    stackoverflow.users u
JOIN 
    stackoverflow.badges b ON u.id = b.user_id
WHERE 
    DATE(u.creation_date) = DATE(b.creation_date);
```

|unique_users|
|---|
|7047|
##### **Задача 4** 
Сколько уникальных постов пользователя с именем Joel Coehoorn получили хотя бы один голос?
```sql
SELECT COUNT(DISTINCT p.id) AS unique_post_count
FROM stackoverflow.users u
JOIN stackoverflow.posts p ON u.id = p.user_id
JOIN stackoverflow.votes v ON p.id = v.post_id
WHERE u.display_name = 'Joel Coehoorn';
```

|unique_post_count|
|---|
|12|
##### **Задача 5** 
Выгрузим все поля таблицы `vote_types`. Добавим к таблице поле `rank`, в которое войдут номера записей в обратном порядке. Таблица должна быть отсортирована по полю `id`. Чтобы пронумеровать записи в обратном порядке, используем оконную функцию.
```sql
SELECT 
    vt.*, 
    ROW_NUMBER() OVER (ORDER BY vt.id DESC) AS rank
FROM 
    stackoverflow.vote_types vt
ORDER BY 
    vt.id;
```

| id  | name                 | rank |
| --- | -------------------- | ---- |
| 1   | AcceptedByOriginator | 15   |
| 2   | UpMod                | 14   |
| 3   | DownMod              | 13   |
| 4   | Offensive            | 12   |
##### **Задача 6** 
Отберем 10 пользователей, которые поставили больше всего голосов типа `Close`. Отобразим таблицу из двух полей: идентификатором пользователя и количеством голосов. Отсортируем данные сначала по убыванию количества голосов, потом по убыванию значения идентификатора пользователя. Таблицы `vote_types` и `users` не связаны напрямую, поэтому понадобится присоединить несколько таблиц.
```sql
SELECT 
    v.user_id,
    COUNT(v.id) AS vote_count
FROM 
    stackoverflow.votes v
JOIN 
    stackoverflow.vote_types vt ON v.vote_type_id = vt.id
WHERE 
    vt.name = 'Close'
GROUP BY 
    v.user_id
ORDER BY 
    vote_count DESC, 
    v.user_id DESC
LIMIT 10;
```

|user_id|vote_count|
|---|---|
|20646|36|
|14728|36|
|27163|29|
|41158|24|
|24820|23|
|9345|23|
|3241|23|
|44330|20|
|38426|19|
|19074|19|
##### **Задача 7** 
Отберем 10 пользователей по количеству значков, полученных в период с 15 ноября по 15 декабря 2008 года включительно. Отобразим несколько полей:
- идентификатор пользователя;
- число значков;
- место в рейтинге — чем больше значков, тем выше рейтинг.

Пользователям, которые набрали одинаковое количество значков, присвойтеим одно и то же место в рейтинге. Отсортируем записи по количеству значков по убыванию, а затем по возрастанию значения идентификатора пользователя.

```sql
WITH UserBadges AS (
    SELECT 
        user_id,
        COUNT(*) AS badge_count
    FROM 
        stackoverflow.badges
    WHERE 
        creation_date BETWEEN '2008-11-15' AND '2008-12-16'
    GROUP BY 
        user_id
),
RankedUsers AS (
    SELECT 
        user_id,
        badge_count,
        DENSE_RANK() OVER (ORDER BY badge_count DESC) AS rank -- Убираем user_id из сортировки
    FROM 
        UserBadges
)
SELECT 
    u.id AS user_id,
    ru.badge_count,
    ru.rank
FROM 
    RankedUsers ru
JOIN 
    stackoverflow.users u ON ru.user_id = u.id
ORDER BY 
    ru.rank ASC  -- Сортировка по рангу
LIMIT 10;
```

|user_id|badge_count|rank|
|---|---|---|
|22656|149|1|
|34509|45|2|
|1288|40|3|
|5190|31|4|
|13913|30|5|
|893|28|6|
|10661|28|6|
|33213|25|7|
|12950|23|8|
|25222|20|9|
##### **Задача 8** 
Сколько в среднем очков получает пост каждого пользователя? Сформируем таблицу из следующих полей:
- заголовок поста;
- идентификатор пользователя;
- число очков поста;
- среднее число очков пользователя за пост, округлённое до целого числа.

Не учитем посты без заголовка, а также те, что набрали ноль очков. Используем оконную функцию и укажем поле, по которому сформировать окна.
```sql
SELECT
	title,
	user_id,
	score,
	ROUND(AVG(score) OVER (PARTITION BY user_id)) AS avg_user_score
FROM stackoverflow.posts
WHERE
	title IS NOT NULL
	AND score != 0
```

| title                                   | user_id | score | avg_user_score |
| --------------------------------------- | ------- | ----- | -------------- |
| Diagnosing Deadlocks in SQL Server 2005 | 1       | 82    | 573            |
| How do I calculate someone's age in C#? | 1       | 1743  | 573            |
| Why doesn't IE7 copy                    | 1       | 37    | 573            |
| Calculate relative time in C#           | 1       | 1348  | 573            |
##### **Задача 9** 
Отобразим заголовки постов, которые были написаны пользователями, получившими более 1000 значков. Посты без заголовков не должны попасть в список. Сформируем список пользователей, которые заработали больше 1000 значков. С помощью этого списка можно отфильтровать записи в основном запросе.
```sql
SELECT 
    p.title
FROM 
    stackoverflow.posts p
WHERE 
    p.user_id IN (
        SELECT 
            user_id
        FROM 
            stackoverflow.badges
        GROUP BY 
            user_id
        HAVING 
            COUNT(*) > 1000
    )
    AND p.title IS NOT NULL               -- Исключаем посты без заголовка
    AND p.title != '';                    -- Исключаем посты с пустыми заголовками
```

| title                                                       |
| ----------------------------------------------------------- |
| What's the strangest corner case you've seen in C# or .NET? |
| What's the hardest or most misunderstood aspect of LINQ?    |
| What are the correct version numbers for C#?                |
| Project management to go with GitHub                        |
##### **Задача 10** 
Напишем запрос, который выгрузит данные о пользователях из Канады (англ. Canada). Разделим пользователей на три группы в зависимости от количества просмотров их профилей:
- пользователям с числом просмотров больше либо равным 350 присвойте группу `1`;
- пользователям с числом просмотров меньше 350, но больше либо равно 100 — группу `2`;
- пользователям с числом просмотров меньше 100 — группу `3`.

Отобразим в итоговой таблице идентификатор пользователя, количество просмотров профиля и группу. Пользователи с количеством просмотров меньше либо равным нулю не должны войти в итоговую таблицу. Чтобы создать категории пользователей, используем оператор `CASE`. Отфильтруем данные по стране, пользуясь оператором `LIKE`.
```sql
SELECT 
    id AS user_id,
    views,
    CASE 
        WHEN views >= 350 THEN 1
        WHEN views >= 100 THEN 2
        WHEN views < 100 AND views > 0 THEN 3
    END AS group_id
FROM 
    stackoverflow.users
WHERE 
    TRIM(location) LIKE '%Canada%' 
    AND views > 0;
```
##### **Задача 11** 
Дополним предыдущий запрос. Отобразим лидеров каждой группы — пользователей, которые набрали максимальное число просмотров в своей группе. Выведем поля с идентификатором пользователя, группой и количеством просмотров. Отсортируем таблицу по убыванию просмотров, а затем по возрастанию значения идентификатора. Добавме предыдущий запрос в подзапрос. Посчитаем максимальное количество просмотров по категориям. В список должны попасть пользователи, у которых число просмотров равно максимальному значению.
```sql
SELECT 
    u.user_id,
    u.group_id,
    u.views
FROM (
    SELECT 
        id AS user_id,
        views,
        CASE 
            WHEN views >= 350 THEN 1
            WHEN views >= 100 THEN 2
            WHEN views < 100 AND views > 0 THEN 3
        END AS group_id
    FROM 
        stackoverflow.users
    WHERE 
        TRIM(location) LIKE '%Canada%' 
        AND views > 0
) AS u
JOIN (
    SELECT 
        group_id,
        MAX(views) AS max_views
    FROM (
        SELECT 
            CASE 
                WHEN views >= 350 THEN 1
                WHEN views >= 100 THEN 2
                WHEN views < 100 AND views > 0 THEN 3
            END AS group_id,
            views
        FROM 
            stackoverflow.users
        WHERE 
            TRIM(location) LIKE '%Canada%' 
            AND views > 0
    ) AS inner_query
    GROUP BY 
        group_id
) AS m ON u.views = m.max_views AND u.group_id = m.group_id
ORDER BY 
    u.views DESC, 
    u.user_id ASC;
```

|user_id|group_id|views|
|---|---|---|
|3153|1|21991|
|46981|2|349|
|3444|3|99|
|22273|3|99|
|190298|3|99|
##### **Задача 12** 
Посчитаем ежедневный прирост новых пользователей в ноябре 2008 года. Сформируем таблицу с полями:
- номер дня;
- число пользователей, зарегистрированных в этот день;
- сумму пользователей с накоплением.

Для подсчёта суммы с накоплением понадобится оконная функция.
```sql
SELECT
	EXTRACT (day FROM creation_date) AS day,
	COUNT(*) AS new_users,
	SUM(COUNT(*)) OVER (ORDER BY EXTRACT (day FROM creation_date)) AS cumulative_users
FROM stackoverflow.users
WHERE creation_date BETWEEN '2008-11-01' AND '2008-12-01'
GROUP BY day
ORDER BY day;
```

|day|new_users|cumulative_users|
|---|---|---|
|1|34|34|
|2|48|82|
|3|75|157|
|4|192|349|
##### **Задача 13** 
Для каждого пользователя, который написал хотя бы один пост, найдем интервал между регистрацией и временем создания первого поста. Отобразим:
- идентификатор пользователя;
- разницу во времени между регистрацией и первым постом.

Для каждого пользователя найдем время создания первого поста с помощью оконной функции ранжирования. Если от этого времени отнять дату регистрации пользователя, получится нужный интервал.
```sql
SELECT 
    u.id AS user_id,
    MIN(p.creation_date) - u.creation_date AS interval
FROM 
    stackoverflow.users u
JOIN 
    stackoverflow.posts p ON u.id = p.user_id
GROUP BY 
    u.id, u.creation_date
HAVING 
    MIN(p.creation_date) IS NOT NULL;
```

| user_id | interval          |
| ------- | ----------------- |
| 24174   | 6:38:11           |
| 282     | 29 days, 2:45:21  |
| 3980    | 24 days, 10:55:03 |
| 25511   | 0:28:23           |
| 7691    | 5:07:46           |
#### **Часть 2**
##### **Задача 1** 
Выведем общую сумму просмотров у постов, опубликованных в каждый месяц 2008 года. Если данных за какой-либо месяц в базе нет, такой месяц можно пропустить. Результат отсортируем по убыванию общего количества просмотров. Используем функцию для усечения даты, а затем сгруппируем и отсортируем данные.
```sql
SELECT
    DATE_TRUNC('month', creation_date)::date AS month,
    SUM(views_count) AS total_views
FROM
    stackoverflow.posts
WHERE
    EXTRACT(YEAR FROM creation_date) = 2008
GROUP BY
    month
ORDER BY
    total_views DESC;
```

|month|total_views|
|---|---|
|2008-09-01|452928568|
|2008-10-01|365400138|
##### **Задача 2**
Выведем имена самых активных пользователей, которые в первый месяц после регистрации (включая день регистрации) дали больше 100 ответов. Вопросы, которые задавали пользователи, не учитываем. Для каждого имени пользователя выведем количество уникальных значений `user_id`. Отсортируем результат по полю с именами в лексикографическом порядке. Нужно присоединить несколько таблиц. Чтобы добавить промежуток времени к дате, используем ключевое слово `INTERVAL`, например, так: `<дата> + INTERVAL '1 year 2 months 3 days'`.
```sql
WITH users_with_posts_count AS (
    SELECT
        u.display_name,
        u.id AS user_id
    FROM
        stackoverflow.posts AS p
    JOIN
        stackoverflow.users AS u ON u.id = p.user_id
    JOIN
        stackoverflow.post_types AS pt ON pt.id = p.post_type_id
    WHERE
        DATE_TRUNC('day', p.creation_date) >= DATE_TRUNC('day', u.creation_date)
        AND DATE_TRUNC('day', p.creation_date) <= DATE_TRUNC('day', u.creation_date) + INTERVAL '1 month'
        AND pt.type = 'Answer'
),
counted_users AS (
    SELECT
        display_name,
        COUNT(DISTINCT user_id) AS unique_user_id_count,
        COUNT(*) 
    FROM
        users_with_posts_count
    GROUP BY
        display_name
    HAVING
        COUNT(*) > 100
)
SELECT
    display_name,
    unique_user_id_count
FROM
    counted_users
ORDER BY
    display_name;
```

|display_name|unique_user_id_count|
|---|---|
|1800 INFORMATION|1|
|Adam Bellaire|1|
|Adam Davis|1|
|Adam Liss|1|
##### **Задача 3** 
Выведем количество постов за 2008 год по месяцам. Отберем посты от пользователей, которые зарегистрировались в сентябре 2008 года и сделали хотя бы один пост в декабре того же года. Отсортируем таблицу по значению месяца по убыванию. Сначала найдем идентификаторы пользователей, которые зарегистрировались в сентябре 2008 года и оставили хотя бы один пост в декабре. Затем используем результат для среза и посчитаем посты по месяцам.
```sql
-- Шаг 1: Получаем идентификаторы пользователей, зарегистрировавшихся в сентябре 2008
WITH users_in_september AS (
    SELECT id
    FROM stackoverflow.users
    WHERE creation_date >= '2008-09-01' AND creation_date < '2008-10-01'
)

-- Шаг 2: Фильтруем пользователей, которые сделали хотя бы один пост в декабре 2008
, active_users AS (
    SELECT DISTINCT u.id
    FROM users_in_september u
    JOIN stackoverflow.posts p ON u.id = p.user_id
    WHERE p.creation_date >= '2008-12-01' AND p.creation_date < '2009-01-01'
)

-- Шаг 3: Считаем количество постов за 2008 год по месяцам от найденных пользователей
SELECT
    DATE_TRUNC('month', p.creation_date)::date AS month,
    COUNT(*) AS post_count
FROM
    stackoverflow.posts p
JOIN
    active_users u ON p.user_id = u.id
WHERE
    p.creation_date >= '2008-01-01' AND p.creation_date < '2009-01-01'
GROUP BY
    month
ORDER BY
    month DESC;  -- Сортировка по месяцу по убыванию
```

| month      | post_count |
| ---------- | ---------- |
| 2008-12-01 | 17641      |
| 2008-11-01 | 18294      |
| 2008-10-01 | 27171      |
| 2008-09-01 | 24870      |
| 2008-08-01 | 32         |
##### **Задача 4** 
Используя данные о постах, выведем несколько полей:
- идентификатор пользователя, который написал пост;
- дата создания поста;
- количество просмотров у текущего поста;
- сумма просмотров постов автора с накоплением.

Данные в таблице должны быть отсортированы по возрастанию идентификаторов пользователей, а данные об одном и том же пользователе — по возрастанию даты создания поста. Для подсчёта суммы с накоплением используем оконную функцию.
```sql
SELECT
    p.user_id,
    p.creation_date,
    p.views_count,
    SUM(p.views_count) OVER (PARTITION BY p.user_id ORDER BY p.creation_date) AS cumulative_views
FROM
    stackoverflow.posts p
ORDER BY
    p.user_id ASC,
    p.creation_date ASC;
```

|user_id|creation_date|views_count|cumulative_views|
|---|---|---|---|
|1|2008-07-31 23:41:00|480476|480476|
|1|2008-07-31 23:55:38|136033|616509|
|1|2008-07-31 23:56:41|0|616509|
##### **Задача 5** 
Сколько в среднем дней в период с 1 по 7 декабря 2008 года включительно пользователи взаимодействовали с платформой? Для каждого пользователя отберем дни, в которые он или она опубликовали хотя бы один пост. Нужно получить одно целое число и округлить результат. Посчитаем, сколько активных дней было у каждого пользователя. Добавим данные в общее табличное выражение и используем в основном запросе.
```sql
WITH active_days AS (
    SELECT 
        p.user_id,
        COUNT(DISTINCT DATE(p.creation_date)) AS active_day_count
    FROM 
        stackoverflow.posts p
    WHERE 
        p.creation_date >= '2008-12-01' 
        AND p.creation_date <= '2008-12-07'
    GROUP BY 
        p.user_id
)

SELECT 
    ROUND(AVG(active_day_count)) AS average_active_days
FROM 
    active_days;
```

|average_active_days|
|---|
|2|
##### **Задача 6** 
На сколько процентов менялось количество постов ежемесячно с 1 сентября по 31 декабря 2008 года? Отобразим таблицу со следующими полями:
- Номер месяца.
- Количество постов за месяц.
- Процент, который показывает, насколько изменилось количество постов в текущем месяце по сравнению с предыдущим.

Если постов стало меньше, значение процента должно быть отрицательным, если больше — положительным. Округлим значение процента до двух знаков после запятой.

При делении одного целого числа на другое в PostgreSQL в результате получится целое число, округлённое до ближайшего целого вниз. Чтобы этого избежать, переведем делимое в тип `numeric`. 

Эту задачу стоит декомпозировать. Сформирем запрос, который отобразит номер месяца и количество постов. Затем можно использовать оконную функцию, которая вернёт значение за предыдущий месяц, и посчитать процент.
```sql
WITH monthly_posts AS (
    SELECT 
        EXTRACT(MONTH FROM p.creation_date) AS month_number,
        COUNT(*) AS post_count
    FROM 
        stackoverflow.posts p
    WHERE 
        p.creation_date >= '2008-09-01' AND p.creation_date < '2009-01-01'
    GROUP BY 
        month_number
    ORDER BY 
        month_number
)

SELECT 
    month_number,
    post_count,
    ROUND(
        (CAST(post_count AS numeric) - LAG(post_count) OVER (ORDER BY month_number)) 
        / NULLIF(LAG(post_count) OVER (ORDER BY month_number), 0) * 100, 2) AS percentage_change
FROM 
    monthly_posts;
```

| month_number | post_count | percentage_change |
| ------------ | ---------- | ----------------- |
| 9            | 70371      |                   |
| 10           | 63102      | -10.33            |
| 11           | 46975      | -25.56            |
| 12           | 44592      | -5.07             |
##### **Задача 7**
Найдем пользователя, который опубликовал больше всего постов за всё время с момента регистрации. Выведем данные его активности за октябрь 2008 года в таком виде:
- номер недели;
- дата и время последнего поста, опубликованного на этой неделе.

Декомпозируем задачу:
- Найдем пользователя, который опубликовал больше всего постов.
- Найдем дату и время создания каждого поста этого пользователя и номер недели.
- Отобразим данные только о последних постах пользователя. Для этого можно использовать оконную функцию.
```sql
WITH user_post_counts AS (
    SELECT 
        p.user_id,
        COUNT(*) AS post_count
    FROM 
        stackoverflow.posts p
    GROUP BY 
        p.user_id
),
top_user AS (
    SELECT 
        user_id
    FROM 
        user_post_counts
    ORDER BY 
        post_count DESC
    LIMIT 1
),
october_posts AS (
    SELECT 
        p.creation_date,
        EXTRACT(WEEK FROM p.creation_date) AS week_number
    FROM 
        stackoverflow.posts p
    JOIN 
        top_user tu ON p.user_id = tu.user_id
    WHERE 
        p.creation_date >= '2008-10-01' AND p.creation_date < '2008-11-01'
),
last_posts AS (
    SELECT 
        week_number,
        MAX(creation_date) AS last_post_date
    FROM 
        october_posts
    GROUP BY 
        week_number
)
SELECT 
    week_number,
    last_post_date
FROM 
    last_posts
ORDER BY 
    week_number;
```

| week_number | last_post_date      |
| ----------- | ------------------- |
| 40          | 2008-10-05 09:00:58 |
| 41          | 2008-10-12 21:22:23 |
| 42          | 2008-10-19 06:49:30 |
| 43          | 2008-10-26 21:44:36 |
| 44          | 2008-10-31 22:16:01 |
