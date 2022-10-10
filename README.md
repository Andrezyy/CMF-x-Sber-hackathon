# CMF x Sber market hackathon

### Постановка задачи
Сбер маркет развивается и становится узнаваемым, растет ежедневное количество заказов. В связи с этим появляется потребность в прогнозировании спроса и планировании рабочих смен, чтобы лучше распоряжаться человечским ресурсом.

### Этапы задачи
1. Спрогнозировать количество заказов, по часам, на 7 дней вперед
2. Определить оптимальное количество курьеров на каждый час, чтобы вероятность опозданий была меньше 0.05
3. В соответсвии с выбраным количеством курьеров, составить оптимальное расписание из 4 - 8 часовых смен

### Исходные данные
1. orders.csv - данные по кол-ву заказов

    ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/orders.png)

2. patners_delays .csv - данные по кол-ву курьеров, с данной вероятность опоздать

    ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/patners_delays.png)



### Решение
### Работа с данным
1. Каждая зона доставки работает свое количество часов, в разное время начинает и заканчивает работать. В таблице orders есть пропуски, часы в которые не было заказов отсутсвуют. Восполним их в соотвествии с временем работы каждой зоны доставки.  Часы, в которые зоны доставок не работают, заполняться не будут, это не нарушит внутрисуточную переодичность. Также добавим некоторые признаки связанные со временем.

    ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/orders_modified.png)

2. Создадим таблицу work_hours в которой будет показаны время работы каждой зоны доставки, это информация пригодится на для заполнения пропусков и на 3ьем этапе.

    ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/work_hours.png)

3. Добавим к таблице patners_delay информацию о заказах, из дозаполенной orders и несколько признаков связанных со времнем. Информация о заказах поможет на 2ом этапе, чтобы на их основе создать новые признаки. 

    ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/partners_delays_modified.png)

4. Создадим таблицу daily_orders, в которой количество заказов будет не по часам а по дням. Она пригодится для 1ого этапа.

    ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/daily_orders.png)
    
5. Создадим тестовый набор данных, который будем заполнять 

    ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/test_data.png)

### I Этап
**Кратко:**
- Для каждой зоны доставки обучим Prophet.
- Будем учить прогнозировать общее количество заказов за день, а затем распределим их внутри дня по долям.
- Используем только 9 последних недель для обучения, так как иначе Prophet плохо справлятся.
- Зоны 2, 181, 182, 190, 195, 200, 367, 453 перестали функционировать, их предсказываем нулями.

    ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/dead_zone_181.png)

**Последовательность действий:**
1. Для зоны доставки №k берем данные
2. Преобразуем их box-cox
3. Обучаем Prophet
4. Делаем прогноз
5. Обратный box-cox
6. Распределяем кол-во заказов внутри дня

**Подсчет долей**
1. Подсчет долей от общего кол-ва заказов за день, распределение для каждой зоны доставки, каждого дня недели, по часам. Результатом будет словарь в виде словаря с ключами: (территория, день недели)
2. Внутри будет словарь вида "час работы : доля". Результат для конкретного ключа: { ... ,14: 0.08, 15: 0.09, ...}

**Качество Propheta на данной задаче**

Для проверки качества модели была проведена кроссвалидация, на 5 фолдов для каждой зоны доставки. Схема кроссвалидации по фолдам следующая:

<ol>
  <li>на 34-42 неделе обучаем, на 43 тестируем</li>
  <li>на 35-43 неделе обучаем, на 44 тестируем</li>
  ...
  <li value="5">на 34-42 неделе обучаем, на 43 тестируем</li>
</ol>

На каждой считали mean mape на трейне и тесте и потом посчитали mean mape среди mean mape по зонам доставки, они получились следующими:
1. Средняя из средних train mape = 0.2053
2. Средняя из средних test mape = 0.2202

**Несколько графиков**

   ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/prophets.png)

**Результат I Этапа**

   ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/first_part.png)
   
### II Этап

Кратко:
- Создадим признак `ord_per_partner`, отношение кол-ва заказов на кол-во курьеров
- Решим задачу бинарной классификации, для этого создадим бинарный признак `high_delay_rate`, равный 1 если `delay_rate >= 0.05` и равный 0 иначе
- Избавимся от категориальных признаков (`delivery_area_id`, `month`, `hour`, `day_of_week`), воспользовавшись WoE бинингом признаков, это приводит их в формат более удобный для подачи в логистическую регрессию
- Обучаем логистическую регрессию (в ходе тестов выбор пал на threshold = 0.29)
- После для каждой строчки из тестового набора данных находим минимальное кол-во курьеров, чтобы наша логистическая регрессия выдалава 0, то бишь `delay_rate<0.05`

С помощью WoE можно посчитать Information value, предсказательную силу признаков

   ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/IV.png)
   
Интересное набллюдение, что `delivery_area_id` обладает сильной предсказательной силой, значит в некоторых интервалах номеров зон доставок попадает большее (или меньшее) число случаев с `delay_rate >= 0.05`

**Результат II Этапа**

   ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/second_part.png)
   
### III Этап

Составим задачу линейного программирования и решим ее симплекс методом

**Результат III Этапа**

   ![minipic](https://github.com/Andrezyy/CMF-x-Sber-hackathon/blob/main/Images/third_part.png)
