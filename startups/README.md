# Описание проекта
## Задачи
1. Предсказать, какие стартапы закроются, а какие нет. 

## Данные
В представленной таблице есть колонки, которые имеют следующее значение:
- name - Название стартапа 
- category_list - Список категорий, к которым относится стартап  
- funding_total_usd - Общая сумма финансирования в USD num
- status - Статус стартапа (закрыт или действующий)  
- country_code - Код страны  ohe
- state_code - Код штата  ohe
- region - Регион  ohe
- city - Город  ohe
- funding_rounds - Количество раундов финансирования num
- founded_at - Дата основания  
- first_funding_at - Дата первого раунда финансирования  
- last_funding_at - Дата последнего раунда финансирования  
- closed_at - Дата закрытия стартапа (если применимо)  
- lifetime - Время существования стартапа в днях num
- category - синтетический ohe
- lifetime_year - синтетический num
- funding_year - синтетический num

## Используемые библиотеки
*pandas, catboost, sklearn, phik*