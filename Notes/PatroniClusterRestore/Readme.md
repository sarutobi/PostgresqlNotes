# Демонстрация восстановления кластера под управлением Patroni из бэкапа, созданного в другом кластере Patroni

Описанная в демонстрации ситуация может возникнуть при необходимости перенести кластер баз данных, находщийся под управлением Patroni, на другой кластер под управлением Patroni, например при актуализации стенда полигона данными с основного сервера БД.

В качестве демонстрации запускаются два кластера Patroni в докерах

## Сторонние ресурсы

Образ докера и файл docker-compose адаптированы из репозитория https://github.com/zalando/patroni
В качестве набора данных использован минимальный набор данных по тайским перевозкам https://github.com/aeuge/postgres16book/tree/main/database

## Начальные условия

Имеются два кластера БД под управлением Patroni
1. Кластер основной БД Production
2. Кластер полигонной БД Polygon
В связи с проведением тестирования новой версии приложения, использующего вашу БД, нужно актуализировать имеющийся полигонный кластер. Разворачивать новый кластер для тестирования не получится из за большого объема БД. Использовавть единственный сервер БД для тестирования не получится, так как в процессе тестирования проверяется отработка смены лидера внутри кластера БД.

## Запуск Production среды
Переходим в директорию Production и выполняем команду

```bash
docker-compose up -d
```

Проверяем что у нас запущен кластер из трех серверов, для этого выполняем последовательно команды
```bash
docker-compose exec patroni1 patronictl list
docker-compose exec patroni2 patronictl list
docker-compose exec patroni3 patronictl list
```

Результат для всех трех серверов должен быть одинаковым
```
+ Cluster: production (7346975539175538712) --+----+-----------+
| Member   | Host       | Role    | State     | TL | Lag in MB |
+----------+------------+---------+-----------+----+-----------+
| patroni1 | 172.19.0.3 | Leader  | running   |  1 |           |
| patroni2 | 172.19.0.2 | Replica | streaming |  1 |         0 |
| patroni3 | 172.19.0.6 | Replica | streaming |  1 |         0 |
+----------+------------+---------+-----------+----+-----------+
```

Создаем базу данных thai
```bash
psql -h localhost -p 5000 -U postgres -c "create database thai;"
```

Скачиваем тестовый набор данных, распаковываем его в директорию tmp и записываем в базу
```bash
cat ./tmp/thai.sql | psql -h localhost -p 5000 -U postgres -d thai
```

## Запуск полигонной среды

Переходим в директорию Polygon и выполняем все действия, аналогичные запуску Production, с учетом того, что общий порт для доступа к БД изменен с 5000 на 5500. Дополнительно создаем базы данных с именем thai_polygon

## Создание бэкапа 
Выполняем команду
```bash
cd ~/ && pg_basebackup -h localhost -p 5000 -U replicator -D prod -Ft -Xs -z -P
```
В результате выполнения команды в домашнем каталоге пользователя будет создана папка prod, содержащая бэкап Production базы данных 
Можно выполнить эту команду на любом из докеров patroni в Production среде, и скопировать полученный результат.

## Восстановление из бэкапа в полигонной среде
Переходим в папку Polygon и проверяем, какой из серверов полигона является лидером
```bash
docker-compose exec patroni1 patronictl list
```
Останавливаем сервисы patroni на репликах и удаляем содержимое каталогов $PGDATA
```bash
docker-compose stop patroni1
docker-compose stop patroni2
rm -r data1/*
rm -r data2/*
```
Останавливаем сервис patroni на лидере, очищаем каталог $PGDATA и восстанавливаем из бэкапа Production на лидере полигона.
```bash
docker-compose stop patroni3
rm -r data3/*
tar xzf prod/base.tar.gz -C data3
tar xzf prod/pg_wal.tar.gs  -C data3/pg_wal
```

Запускаем patroni на лидере
```bash
docker-compose up -d patroni3
```
И ..... ошибка:

```CRITICAL: system ID mismatch, node patroni3 belongs to a different cluster: 7347229796124925975 != 7346975539175538712```

Внутри postgres есть идентификатор кластера, и он не совпадает с тем. который прописан в etcd
Проверяем

```bash
docker-compose exec -it etcd1 "ETCDCTL_API=3 etcdctl get /service/production/initialize"

/service/production/initialize
7347229796124925975
```
Заменяем на нужный ключ
```bash
docker-compose exec -it etcd1 "ETCDCTL_API=3 etcdctl put /service/production/initialize 7346975539175538712"
```
Заново запускаем patroni на лидере
И... снова проблема

```bash
docker-compose exec patroni1 patronictl list
```
```
+ Cluster: production (7346975539175538712) +----+-----------+
| Member   | Host       | Role    | State   | TL | Lag in MB |
+----------+------------+---------+---------+----+-----------+
| patroni3 | 172.30.0.3 | Replica | running |    |   unknown |
+----------+------------+---------+---------+----+-----------+
```
Смотрим логи postgres и видим там:

``` FATAL:  password authentication failed for user "postgres"```

Изменяем в конфигурации patroni (patroni.env) пароль пользователя postgres на пароль с production, перезапускаем patroni и
```
+ Cluster: production (7346975539175538712) ----+-----------+
| Member   | Host       | Role   | State   | TL | Lag in MB |
+----------+------------+--------+---------+----+-----------+
| patroni3 | 172.30.0.3 | Leader | running |  2 |           |
+----------+------------+--------+---------+----+-----------+
```
Осталось изменить пароль пользователя postgres на полигонный внутри лидера

```bash
psql -h localhost -p 5500 -U postgres -c "alter user postgres with password 'polypostgres';"
```

вернуть пароль в конфигурации patroni, перезапустить patroni и реинициализировать реплики

```bash
docker-compose up -d patroni1
docker-compose up -d patroni2
docker-compose exec patroni3 patronictl reinit production patroni1
docker-compose exec patroni3 patronictl reinit production patroni2
docker-compose exec patroni3 patronictl list
```
```
+ Cluster: production (7346975539175538712) --+----+-----------+
| Member   | Host       | Role    | State     | TL | Lag in MB |
+----------+------------+---------+-----------+----+-----------+
| patroni1 | 172.30.0.8 | Replica | streaming |  3 |         0 |
| patroni2 | 172.30.0.6 | Replica | streaming |  3 |         0 |
| patroni3 | 172.30.0.3 | Leader  | running   |  3 |           |
+----------+------------+---------+-----------+----+-----------+
```

