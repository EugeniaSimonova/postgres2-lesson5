# PostgreSQL и VKcloud 

## Создание инстанса баз данных

Заходим в **VK Cloud Аккаунт -> Ключевые пары SSH** и создаем новый ключ либо добавляем существующий. Я использовала существующий.


Открываем **VK Cloud  -> Cloud Databases -> Инстансы баз данных -> Создание новой базы данных**. Выбираем **PostgreSQL версия PostgreSQL 15** и конфигурацию **Master-Replica**.

На следующем шаге выбираем требуемые параметры, в том числе и внешний IP, а также указываем ssh ключ.

Открываем **Параметры подключения из приложений -> Shell** копируем строку подключения, подставляем свои значения для базы данных и пользователя и проверяем, сможем ли мы подключиться:

````
psql -h 185.86.147.216 -p 5432 -d otus-db1 -U user1 -W

````

На вкладке инстанса БД **Расширения** добавим *timescaledb*.

Создадим таблицу через psql:

````
otus-db1=> CREATE EXTENSION timescaledb;

CREATE EXTENSION
otus-db1=> CREATE TABLE base_table(
otus-db1(>     id uuid,
otus-db1(>     time_column timestamptz not null default clock_timestamp(),
otus-db1(>     data_column1 int default random()*1E5,
otus-db1(>     data_column2 int default random()*1E5
otus-db1(> );
CREATE TABLE
otus-db1=> 
otus-db1=> SELECT create_hypertable('base_table', by_range('time_column'));
 create_hypertable 
-------------------
 (1,t)
(1 row)

otus-db1=> INSERT INTO base_table(id, time_column) SELECT gen_random_uuid(), generate_series(
otus-db1(> now() - INTERVAL '6 months',
otus-db1(> now(),
otus-db1(> INTERVAL '1 minute'
otus-db1(> );
INSERT 0 260641
otus-db1=> 
````

Часть конфигурационнных параметров из postgresql.conf можно поменять на вкладке инстанса **Параметры баз данных** при необходимости.

## Работа с REST API

Пробуем использовать REST API:


Переходим на вкладку проекта **Доступ по API** и скачиваем файл *<имя проекта>-openrc.sh*, в моём случае `mcs2397908970-openrc.sh`.  Устанавливаем  OpenStack CLI  по инструкции из статьи [OpenStack CLI](https://cloud.vk.com/docs/ru/tools-for-using-services/cli/openstack-cli).

Проверяем, что команда `openstack project list` возвращает наш проект:

````
+----------------------------------+---------------+
| ID                               | Name          |
+----------------------------------+---------------+
| 08a0d603b7e440c28f2b916a95d2d5ab | mcs2397908970 |
+----------------------------------+---------------+

````   
Получаем токен, для использования REST API:

````
$ openstack token issue -c id -f value
gAAAAABnzay_91wS281avwUscT_f_dHIKVnVhgQe1XFFg1qlUUmgsHSsk_tQ-UdrFTwHxXihey3E5QPHid8Cif003lyDtpV2qR5WXcmHAV6zpDz7nyYz5R89c0qh3klDMcE0N7--HGOtxszMCjvkwWH2n2Pltn5mUyb76AVZN6jWyCwy74OXkI0

````

Обновляем версию PostgreSQL, подставляя токен, ID проекта и ID инстанса в команду:

````
curl --location --request PATCH 'https://infra.mail.ru:8779/v1.0/08a0d603b7e440c28f2b916a95d2d5ab/instances/a6c9857b-3792-46fb-9687-33be6a9b0c46' \
--header 'X-Auth-Token:gAAAAABnzay_91wS281avwUscT_f_dHIKVnVhgQe1XFFg1qlUUmgsHSsk_tQ-UdrFTwHxXihey3E5QPHid8Cif003lyDtpV2qR5WXcmHAV6zpDz7nyYz5R89c0qh3klDMcE0N7--HGOtxszMCjvkwWH2n2Pltn5mUyb76AVZN6jWyCwy74OXkI0' \
--header 'Content-Type: application/json' \
-d '{
  "instance":{
      "datastore_version": "16"
  }
}'

````

Убеждаемся, что результат достигнут через WEB UI. Версия инстанса должна поменяться на 16.

На вкладке инстанса **Расширения** добавляем второе расширение `postgres_exporter`.

Получаем метрики:

````
curl -s http://185.86.147.216:9187/metrics
````
Всё работает как ожидалось.


