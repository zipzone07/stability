# Описание проекта
## Задачи
1. Подтвердить несколько гипотез с помощью анализа даных.

## Данные
В представленной таблице есть колонки, которые имеют следующее значение:
- children — количество детей в семье,
- days_employed — общий трудовой стаж в днях,
- dob_years — возраст клиента в годах,
- education — уровень образования клиента,
- education_id — идентификатор уровня образования,
- family_status — семейное положение,
- family_status_id — идентификатор семейного положения,
- gender — пол клиента,
- income_type — тип занятости,
- debt — имел ли задолженность по возврату кредитов,
- total_income — ежемесячный доход,
- purpose — цель получения кредита.

## Используемые библиотеки
*pandas*

## Итоги
Определили заемщиков с худшим результатом возврата кредита:
- в категории с количеством детей - клиенты с четырьмя детьми,
- в категории семейное положение - клиенты с статусом *не женат / не замужем*. 
- в категории уровнень дохода - у клиентов с месячным доходом до 30 000 рублей. 
- в категории цель кредита - у клиентов *операции с автомобилем* и *получение образования*.