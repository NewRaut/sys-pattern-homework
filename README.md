### Домашнее задание к занятию `«Резервное копирование баз данных»` - <Брындин Олег>


### Задание 1. Резервное копирование
Кейс
Финансовая компания решила увеличить надёжность работы баз данных и их резервного копирования.

Необходимо описать, какие варианты резервного копирования подходят в случаях:

1.1. Необходимо восстанавливать данные в полном объёме за предыдущий день.
1.2. Необходимо восстанавливать данные за час до предполагаемой поломки.
1.3.* Возможен ли кейс, когда при поломке базы происходило моментальное переключение на работающую или починенную базу данных.

Приведите ответ в свободной форме

`ОТВЕТ`
1.1. Восстановление данных в полном объёме за предыдущий день
Подходящий вариант: Полные (физические) резервные копии + бинарные логи (WAL/redo logs)

Стратегия:
- Ежедневное создание полной физической резервной копии (например, в 2:00 ночи)
- Непрерывное архивирование бинарных/транзакционных логов (WAL для PostgreSQL, binlog для MySQL)
- Хранение всех логов за прошедшие сутки

Процесс восстановления:
- Восстановить полную копию за вчерашний день
- Применить все WAL/binlog файлы до конца предыдущего дня
- База восстановлена до состояния на конец предыдущего дня

1.2. Восстановление данных за час до поломки
Подходящий вариант: Полные копии + бинарные логи + точка восстановления (PITR)

Стратегия:
- Ежедневные полные резервные копии
- Непрерывное архивирование WAL/binlog
- Возможность указывать точное время восстановления (Point-in-Time Recovery)

Процесс восстановления:
- Восстановить последнюю полную резервную копию
- Применять WAL/binlog файлы до момента "за час до поломки"
- Остановить восстановление на заданном временном моменте

1.3.* Моментальное переключение при поломке

Варианты реализации:

Репликация Master-Slave с автоматическим failover:
- Master работает в активном режиме
- Slave в режиме горячего резерва с постоянной синхронной/асинхронной репликацией
- Мониторинг HA (Pacemaker, Patroni, Orchestrator)
- При падении Master автоматическое переключение на Slave

Кластерные решения:
- PostgreSQL: Patroni + etcd, PostgreSQL с синхронной репликацией
- MySQL: MySQL Group Replication, InnoDB Cluster
- Galera Cluster для MariaDB/Percona

Распределённые СУБД:
- CockroachDB, YugabyteDB — автоматическое распределение и восстановление

Требования:
- Синхронная репликация для нулевой потери данных
- Мониторинг состояния узлов
- Механизм кворума для выбора нового лидера
- Автоматическое перераспределение клиентов


### Задание 2. PostgreSQL
2.1. С помощью официальной документации приведите пример команды резервирования данных и восстановления БД (pgdump/pgrestore).
2.1.* Возможно ли автоматизировать этот процесс? Если да, то как?

Приведите ответ в свободной форме.

`ОТВЕТ`

2.1. Пример команд резервирования и восстановления

Создание резервной копии с помощью pg_dump:
# Резервное копирование всей базы данных в custom-формате (с возможностью восстановления отдельных объектов)
pg_dump -U postgres -h localhost -Fc -f /backup/db_backup.dump mydatabase
# Резервное копирование в plain SQL формат
pg_dump -U postgres -h localhost -f /backup/db_backup.sql mydatabase
# Резервное копирование сжатой копии
pg_dump -U postgres -h localhost -Fc -Z 9 -f /backup/db_backup.dump.gz mydatabase
# Резервное копирование только схемы
pg_dump -U postgres -h localhost -s -f /backup/schema_only.sql mydatabase
# Резервное копирование только данных
pg_dump -U postgres -h localhost -a -f /backup/data_only.sql mydatabase


Восстановление с помощью pg_restore:
# Восстановление из custom-формата в новую базу данных
pg_restore -U postgres -h localhost -d newdatabase -C /backup/db_backup.dump
# Восстановление только структуры таблиц
pg_restore -U postgres -h localhost -d mydatabase -s /backup/db_backup.dump
# Восстановление только данных
pg_restore -U postgres -h localhost -d mydatabase -a /backup/db_backup.dump
# Восстановление с параллельным выполнением (ускорение)
pg_restore -U postgres -h localhost -d mydatabase -j 4 /backup/db_backup.dump


### Задание 3. MySQL
3.1. С помощью официальной документации приведите пример команды инкрементного резервного копирования базы данных MySQL.
3.1.* В каких случаях использование реплики будет давать преимущество по сравнению с обычным резервным копированием?

Приведите ответ в свободной форме.


`ОТВЕТ`

3.1. Пример инкрементного резервного копирования

Способ 1: Использование бинарных логов (binlog) для инкрементного бэкапа
# 1. Делаем полный бэкап
mysqldump -u root -p --all-databases --single-transaction --flush-logs \
  --master-data=2 > /backup/full_backup_$(date +%Y%m%d).sql
# После этого MySQL начнёт новый бинарный лог
# 2. Периодически сохраняем бинарные логи для инкрементного бэкапа
# Ручное переключение логов и сохранение
mysql -u root -p -e "FLUSH BINARY LOGS;"
cp /var/lib/mysql/mysql-bin.* /backup/binlogs/
# Или использование mysqlbinlog
mysqlbinlog --read-from-remote-server --host=localhost --user=root \
  --raw --stop-never mysql-bin.000012 > /backup/incremental_backup.binlog


Способ 2: Использование XtraBackup (Percona) для инкрементных бэкапов
# 1. Полный бэкап
xtrabackup --backup --user=root --password=password \
  --target-dir=/backup/full/
# 2. Первый инкрементный бэкап (относительно полного)
xtrabackup --backup --user=root --password=password \
  --target-dir=/backup/inc1/ \
  --incremental-basedir=/backup/full/
# 3. Второй инкрементный бэкап (относительно предыдущего инкрементного)
xtrabackup --backup --user=root --password=password \
  --target-dir=/backup/inc2/ \
  --incremental-basedir=/backup/inc1/
# 4. Применение инкрементных бэкапов для восстановления
xtrabackup --prepare --apply-log-only --target-dir=/backup/full/
xtrabackup --prepare --apply-log-only --target-dir=/backup/full/ \
  --incremental-dir=/backup/inc1/
xtrabackup --prepare --apply-log-only --target-dir=/backup/full/ \
  --incremental-dir=/backup/inc2/
xtrabackup --prepare --target-dir=/backup/full/


Способ 3: MySQL Enterprise Backup (коммерческая версия)
# Инкрементный бэкап
mysqlbackup --user=root --password=password \
  --incremental --incremental-base=history:last_full_backup \
  --backup-dir=/backup/incremental_backup backup