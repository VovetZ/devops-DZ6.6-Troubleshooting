# devops-DZ6.6-Troubleshooting
# Домашнее задание к занятию "6.6. Troubleshooting"

## Задача 1

Перед выполнением задания ознакомьтесь с документацией по [администрированию MongoDB](https://docs.mongodb.com/manual/administration/).

Пользователь (разработчик) написал в канал поддержки, что у него уже 3 минуты происходит CRUD операция в MongoDB и её 
нужно прервать. 

Вы как инженер поддержки решили произвести данную операцию:
- напишите список операций, которые вы будете производить для остановки запроса пользователя
- предложите вариант решения проблемы с долгими (зависающими) запросами в MongoDB

## Решение

1) Найти `opid` операций, которые выполняются дольше 3 минут (180 секунд)

```js  
db.currentOp(
   {
     "active" : true,
     "secs_running" : { "$gt" : 180 },
   }
)
```
Завершить операцию по найденным `opid`

```js
db.killOp(<opid>)
```

2) Решение "в лоб" - установить максимально возможное время выполнения запросов CRUD через `$maxTimeMS`. Но по хорошему надо настроить профилирование и проанализировать долгие запросы. Далее оптимизировать запросы, создать необходимые индексы

## Задача 2

Перед выполнением задания познакомьтесь с документацией по [Redis latency troobleshooting](https://redis.io/topics/latency).

Вы запустили инстанс Redis для использования совместно с сервисом, который использует механизм TTL. 
Причем отношение количества записанных key-value значений к количеству истёкших значений есть величина постоянная и
увеличивается пропорционально количеству реплик сервиса. 

При масштабировании сервиса до N реплик вы увидели, что:
- сначала рост отношения записанных значений к истекшим
- Redis блокирует операции записи

Как вы думаете, в чем может быть проблема?
 
## Решение

Предполагаю, проблема в методе удаления ключей с истекшим сроком действия. 
Если `Redis` блокирует операции записи, значит в базе появилось большое количество ключей, которые истекают в одно время (их количество больше 25% от текущей совокупности ключей с истекшим сроком действия). 
В `Redis` есть два способа очистить "просроченные" записи: "ленивый" и "активный". В случае с "ленивым" - "просроченные" записи запрашиваются командой, "активный" способ - каждые 100 мс проводит записи в состояние устаревшие, затем данные записи исключаются. ACTIVE_EXPIRE_CYCLE_LOOKUPS_PER_LOOP по умолчанию имеет значение 20, таким образом за раз можно пометить и очистить около 200 устаревших записей. Процесс проверки зациклится и мы ощутим задержки, а потом и вовсе не будут приниматься данные на запись, если появятся истекшие записи более 25% по отношению к общему количеству. 

## Задача 3

Вы подняли базу данных MySQL для использования в гис-системе. При росте количества записей, в таблицах базы,
пользователи начали жаловаться на ошибки вида:
```python
InterfaceError: (InterfaceError) 2013: Lost connection to MySQL server during query u'SELECT..... '
```

Как вы думаете, почему это начало происходить и как локализовать проблему?

Какие пути решения данной проблемы вы можете предложить?

## Решение

Вероятные причины:
1) Сетевые проблемы. Необходимо проверить подключение. Также необходимо выполнить команду `SHOW GLOBAL STATUS LIKE 'Aborted_connects'`. Если его значение увеличивается (оно увеличивается при каждом прерывании соединения со стороны сервера), то необходимо увеличить параметр `connect_timeout` 
2) Большой запрос, который не успевает отработать за `net_read_timeout` (30 секунд по умолчанию). Увеличить данный параметр
3) Если в ошибке есть `ER_NET_PACKET_TOO_LARGE`, то вероятно проблема в превышении размера сообщения. Необходимо увеличить параметр `max_allowed_packet`

## Задача 4

Вы решили перевести гис-систему из задачи 3 на PostgreSQL, так как прочитали в документации, что эта СУБД работает с 
большим объемом данных лучше, чем MySQL.

После запуска пользователи начали жаловаться, что СУБД время от времени становится недоступной. В dmesg вы видите, что:

`postmaster invoked oom-killer`

Как вы думаете, что происходит?

Как бы вы решили данную проблему?

## Решение

Полагаю, что на сервере `PostgreSQL` утилизировал всю доступную оперативную память, поэтому `oom-killer` (процесс, который завершает приложение при нехватки памяти) завершил его процесс. Это необходимо, чтобы не обрушилась вся система.

Какие есть пути решения:

2) Ограничить доступную для `PostgreSQL` память
3) Не запускать на одном сервере с `PostgreSQL` другие приложения, которые могут утилизировать память
4) Увеличить ресурсы сервера (есть вероятность, что текущих мощностей уже не хватает)
1) Отключить `oom-killer`. Это небезопасно для сервера, поэтому не рекомендуется
