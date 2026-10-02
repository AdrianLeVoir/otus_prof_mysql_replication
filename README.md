#Домашнее задание: MySQL репликация

**Цель:** настроить репликацию MySQL с фильтром по таблицам на GTID.

**Реализация:** два ВМ на Proxmox из cloud-образа Ubuntu 24.04 (cloud-init для первичной настройки пользователя/сети, дальше — обычный netplan/sshd). На мастере развёрнута база `bet` из дампа, настроена GTID-репликация на реплику с фильтром таблиц.

## Стенд

| Роль | ВМ | IP | ОС | MySQL |
|------|-----|-----|-----|-------|
| Мастер (`mysql-master`) | 560 | 192.168.1.190 | Ubuntu 24.04 | mysql-server 8.0 |
| Реплика (`mysql-replica`) | 561 | 192.168.1.181 | Ubuntu 24.04 | mysql-server 8.0 |

## Файлы

- `mysql_repl-master.conf` — конфиг мастера (`server-id=1`, `log-bin`, `gtid-mode=ON`, `enforce-gtid-consistency`, `log-replica-updates`, `binlog_format=row`);
- `mysql_repl-replica.cnf` — конфиг реплики (`server-id=2`, `read-only=ON`, `relay-log`, фильтры `replicate-wild-do-table` на 5 таблиц);
- `bet-224190-d906e5.dmp` — дамп базы `bet` (развёрнут на мастере);
- `logs/` — проверочные логи (утилита `script`).

## Реплицируемые таблицы

`bookmaker`, `competition`, `market`, `odds`, `outcome` — фильтр на реплике через `replicate-wild-do-table = bet.<table>`.

## Результаты проверки

- **Мастер**: `SHOW TABLES FROM bet` → 7 таблиц; `@@gtid_mode = ON`, `binlog_format = ROW`;
- **Реплика**: `SHOW TABLES FROM bet` → ровно 5 таблиц (`bookmaker`, `competition`, `market`, `odds`, `outcome`); `read_only = ON`;
- `SHOW REPLICA STATUS`: `Replica_IO_Running: Yes`, `Replica_SQL_Running: Yes`, `Seconds_Behind_Source: 0`;
- **Пример изменения строки**: на мастере `INSERT ... ('OddsPortal')` → `UPDATE ... 'OddsPortal Live'` → на реплике строка появилась (`7 | OddsPortal Live`); затем `UPDATE ... 'OddsPortal Exchange'` и `INSERT ('Marathonbet')` — обе операции отражены на реплике;
- фильтр проверен: таблицы дампа вне списка (`events_on_demand`, `v_same_event`) на реплике отсутствуют.

## Логи (script)

- `logs/mysql_master_show_tables.log` — SHOW TABLES на мастере, параметры репликации, первый INSERT/UPDATE строки;
- `logs/mysql_master_row_change.log` — UPDATE/INSERT строки `OddsPortal Exchange` / `Marathonbet` на мастере;
- `logs/mysql_replica_state.log` — SHOW TABLES на реплике (5 таблиц), параметры (`server_id=2, gtid_mode=ON, read_only=1`), статус репликации (`Replica_IO_Running: Yes`, `Replica_SQL_Running: Yes`, `Auto_Position: 1`) и таблица bookmaker с приехавшими строками.

Реплика `CHANGE REPLICATION SOURCE TO SOURCE_HOST='192.168.1.190', SOURCE_USER='repl', SOURCE_AUTO_POSITION=1, GET_SOURCE_PUBLIC_KEY=1;`

## Грабли

- `bind-address` в Ubuntu по умолчанию `127.0.0.1` — реплика не могла подключиться, пока не выставлен `0.0.0.0`;
- база `bet` на реплике должна существовать до старта реплики: транзакции дампа в GTID-потоке идут как `USE bet; CREATE TABLE...` и падают с `Unknown database 'bet'` — после `CREATE DATABASE` репликация поднялась;
- при включённом `replicate-wild-do-table` события по остальным таблицам пропускаются (`Empty GTID`), ошибки не блокируют поток.
