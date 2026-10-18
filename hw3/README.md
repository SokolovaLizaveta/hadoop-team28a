# Домашнее задание №3

Репозиторий содержит пошаговую инструкцию развертывания Apache Hive `4.0.0-alpha-2` на базе существующего HDFS/YARN-кластера команды `team28a`.

В рамках работы:

- развернут внешний PostgreSQL Metastore без использования embedded-подхода;
- запущены Hive Metastore и HiveServer2;
- проверено одновременное подключение двух клиентов через Beeline;
- создана партиционированная таблица Hive;
- реализована автоматизированная подготовка данных и загрузка в статические партиции;
- выполнена проверка зарегистрированных партиций и фактических данных в HDFS.

В рамках домашнего задания №3 к существующему кластеру добавляются:

- PostgreSQL на `team-28-01` — внешняя база данных Hive Metastore;
- Hive Metastore на `team-28-nn`;
- HiveServer2 на `team-28-nn`.

Итоговая архитектура после выполнения работы:

```text
team-28-en
└── edge-узел

team-28-nn
├── NameNode
├── DataNode
├── ResourceManager
├── NodeManager
├── JobHistoryServer
├── Hive Metastore
└── HiveServer2

team-28-00
├── SecondaryNameNode
├── DataNode
└── NodeManager

team-28-01
├── DataNode
├── NodeManager
└── PostgreSQL
```

HiveServer2 используется как точка подключения клиентов Hive. Hive Metastore работает как отдельный сервис и хранит метаданные в PostgreSQL, поэтому embedded Metastore не используется.

## Содержание

1. [Исходная инфраструктура](#1-исходная-инфраструктура)
2. [Подключение и проверка состояния кластера](#2-подключение-и-проверка-состояния-кластера)
3. [Настройка ресурсов YARN для Hive](#3-настройка-ресурсов-yarn-для-hive)
4. [Проверка MapReduce](#4-проверка-mapreduce)
5. [Выбор портов и каталогов Hive](#5-выбор-портов-и-каталогов-hive)
6. [Установка PostgreSQL](#6-установка-postgresql)
7. [Настройка PostgreSQL для Hive Metastore](#7-настройка-postgresql-для-hive-metastore)
8. [Установка Apache Hive](#8-установка-apache-hive)
9. [Настройка PostgreSQL JDBC Driver](#9-настройка-postgresql-jdbc-driver)
10. [Настройка hive-site.xml](#10-настройка-hive-sitexml)
11. [Подготовка HDFS warehouse](#11-подготовка-hdfs-warehouse)
12. [Инициализация схемы и запуск Hive Metastore](#12-инициализация-схемы-и-запуск-hive-metastore)
13. [Запуск HiveServer2](#13-запуск-hiveserver2)
14. [Проверка подключения через Beeline](#14-проверка-подключения-через-beeline)
15. [Проверка одновременной работы двух клиентов](#15-проверка-одновременной-работы-двух-клиентов)
16. [Создание базы и таблиц Hive](#16-создание-базы-и-таблиц-hive)
17. [Автоматизированная загрузка данных в партиции](#17-автоматизированная-загрузка-данных-в-партиции)
18. [Проверка партиций и данных](#18-проверка-партиций-и-данных)
19. [Финальная проверка сервисов](#19-финальная-проверка-сервисов)
20. [Итог](#20-итог)


## 1. Исходная инфраструктура

Домашнее задание №3 выполняется на базе HDFS- и YARN-кластера, развернутого в домашних заданиях №1 и №2.

Для команды `team28a` используются следующие виртуальные машины:

| Узел | Внутренний IP | Существующие сервисы |
|---|---|---|
| `team-28-en` | `10.28.0.10` | edge-узел |
| `team-28-nn` | `10.28.0.11` | NameNode, DataNode, ResourceManager, NodeManager, JobHistoryServer |
| `team-28-00` | `10.28.0.12` | SecondaryNameNode, DataNode, NodeManager |
| `team-28-01` | `10.28.0.13` | DataNode, NodeManager |

Edge-узел `team-28-en` используется для входа во внутреннюю сеть и управления остальными виртуальными машинами. Hadoop- и Hive-сервисы на нем не запускаются.

На трех внутренних узлах уже установлены:

```text
Java 11
Apache Hadoop 3.4.3
```

Используются пользовательские каталоги:

```text
/home/team28a/apps/java
/home/team28a/apps/hadoop
```

Обе команды `team28a` и `team28b` работают на одних физических виртуальных машинах под разными Linux-пользователями. Все каталоги, процессы и порты в данной инструкции относятся только к пользователю `team28a`.

## 2. Подключение и проверка состояния кластера

Все дальнейшие действия выполняются под пользователем `team28a`.

Подключаемся к edge-узлу:

```bash
ssh team28a@2.59.83.133
```

После подключения командная строка имеет вид:

```text
team28a@team-28-en:~$
```

Для подключения с edge-узла к внутренним виртуальным машинам используется SSH-ключ:

```text
~/.ssh/team28a_internal
```

Проверяем доступ к трем внутренним узлам:

```bash
for host in team-28-nn team-28-00 team-28-01; do
  echo "===== $host ====="
  ssh -i ~/.ssh/team28a_internal team28a@$host 'hostname'
done
```

Последовательно получены:

```text
team-28-nn
team-28-00
team-28-01
```

Перед установкой Hive проверяем, что существующие HDFS- и YARN-сервисы работают.

### Проверка Java-процессов

С edge-узла выполняем:

```bash
for host in team-28-nn team-28-00 team-28-01; do
  echo "===== $host ====="
  ssh -i ~/.ssh/team28a_internal team28a@$host     'JAVA_HOME=$HOME/apps/java $HOME/apps/java/bin/jps'
  echo
done
```

На `team-28-nn` присутствуют:

```text
NameNode
DataNode
ResourceManager
NodeManager
JobHistoryServer
```

На `team-28-00`:

```text
SecondaryNameNode
DataNode
NodeManager
```

На `team-28-01`:

```text
DataNode
NodeManager
```

### Проверка HDFS

Проверяем safe mode и состояние DataNode:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn '
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

hdfs dfsadmin -safemode get

hdfs dfsadmin -report   | grep -E "Under replicated blocks:|Blocks with corrupt replicas:|Missing blocks:|Live datanodes"
'
```

Рабочее состояние HDFS:

```text
Safe mode is OFF
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Live datanodes (3)
```

### Проверка YARN

Проверяем регистрацию NodeManager:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn '
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

yarn node -list
'
```

Полученный результат:

```text
Total Nodes:3
```

Все три NodeManager находятся в состоянии:

```text
RUNNING
```

Дополнительно убеждаемся, что перед дальнейшей настройкой нет выполняющихся YARN-приложений:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn '
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

yarn application -list -appStates RUNNING
'
```

Получаем:

```text
Total number of applications (application-types: [], states: [RUNNING] and tags: []):0
```

После этих проверок существующий HDFS/YARN-кластер готов к настройке ресурсов и последующему развертыванию Hive.

## 3. Настройка ресурсов YARN для Hive

Перед запуском Hive были ограничены ресурсы YARN. Это необходимо, поскольку пользователи `team28a` и `team28b` работают на одних физических виртуальных машинах, а их YARN-кластеры не координируют использование общей оперативной памяти.

Существующие адреса и порты YARN из домашнего задания №2 сохраняются. В рамках этого этапа изменяются только лимиты ресурсов NodeManager и калькулятор ресурсов CapacityScheduler.

Перед изменением конфигурации сохраняем резервные копии:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn '
mkdir -p ~/config-backups/yarn-before-resource-tuning
cp ~/apps/hadoop/etc/hadoop/yarn-site.xml    ~/config-backups/yarn-before-resource-tuning/yarn-site.xml
cp ~/apps/hadoop/etc/hadoop/capacity-scheduler.xml    ~/config-backups/yarn-before-resource-tuning/capacity-scheduler.xml
'
```

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-00 '
mkdir -p ~/config-backups/yarn-before-resource-tuning
cp ~/apps/hadoop/etc/hadoop/yarn-site.xml    ~/config-backups/yarn-before-resource-tuning/yarn-site.xml
cp ~/apps/hadoop/etc/hadoop/capacity-scheduler.xml    ~/config-backups/yarn-before-resource-tuning/capacity-scheduler.xml
'
```

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-01 '
mkdir -p ~/config-backups/yarn-before-resource-tuning
cp ~/apps/hadoop/etc/hadoop/yarn-site.xml    ~/config-backups/yarn-before-resource-tuning/yarn-site.xml
cp ~/apps/hadoop/etc/hadoop/capacity-scheduler.xml    ~/config-backups/yarn-before-resource-tuning/capacity-scheduler.xml
'
```

### Ресурсы NodeManager на `team-28-nn`

В файл:

```text
~/apps/hadoop/etc/hadoop/yarn-site.xml
```

добавлены параметры:

```xml
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>1024</value>
</property>

<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>1</value>
</property>
```

NodeManager на `team-28-nn` предоставляет YARN:

```text
1024 MB
1 vcore
```

### Ресурсы NodeManager на `team-28-00`

В `yarn-site.xml` добавлены:

```xml
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>3072</value>
</property>

<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>2</value>
</property>
```

NodeManager на `team-28-00` предоставляет:

```text
3072 MB
2 vcores
```

### Ресурсы NodeManager на `team-28-01`

На `team-28-01` используются те же лимиты, что и на втором рабочем узле:

```xml
<property>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>3072</value>
</property>

<property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>2</value>
</property>
```

Получаем:

```text
3072 MB
2 vcores
```

### Настройка CapacityScheduler

На всех трех внутренних узлах в файл:

```text
~/apps/hadoop/etc/hadoop/capacity-scheduler.xml
```

добавлен калькулятор ресурсов:

```xml
<property>
    <name>yarn.scheduler.capacity.resource-calculator</name>
    <value>org.apache.hadoop.yarn.util.resource.DominantResourceCalculator</value>
</property>
```

`DominantResourceCalculator` позволяет планировщику учитывать одновременно память и число vcore при распределении ресурсов.

### Проверка `mapred-site.xml`

Для выполнения MapReduce через YARN в итоговой конфигурации `mapred-site.xml` присутствуют параметры:

```xml
<property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
</property>

<property>
    <name>mapreduce.application.classpath</name>
    <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
</property>
```

Остальные адреса и порты MapReduce/JobHistoryServer остаются настроенными по домашнему заданию №2.

### Перезапуск NodeManager

После изменения resource-параметров перезапускаем NodeManager отдельно на каждом узле.

На `team-28-nn`:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn '
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

yarn --daemon stop nodemanager
yarn --daemon start nodemanager
'
```

На `team-28-00`:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-00 '
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

yarn --daemon stop nodemanager
yarn --daemon start nodemanager
'
```

На `team-28-01`:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-01 '
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

yarn --daemon stop nodemanager
yarn --daemon start nodemanager
'
```

### Проверка итоговых ресурсов

С `team-28-nn` запрашиваем подробную информацию о зарегистрированных NodeManager:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn '
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH

yarn node -list -showDetails
'
```

ResourceManager зарегистрировал три NodeManager со следующими ресурсами:

```text
team-28-nn
Configured Resources : <memory:1024, vCores:1>

team-28-00
Configured Resources : <memory:3072, vCores:2>

team-28-01
Configured Resources : <memory:3072, vCores:2>
```

Все три узла находятся в состоянии:

```text
RUNNING
```

## 4. Проверка MapReduce

После настройки ресурсов YARN выполняем тест MapReduce. Для проверки используется стандартный пример `wordcount` из поставки Hadoop.

Тест содержит только пять слов и используется для проверки полного пути:

```text
HDFS → YARN → MapReduce → HDFS
```

Все команды запускаются на `team-28-nn`.

Подключаемся к узлу:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn
```

Настраиваем переменные окружения:

```bash
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
```

### Подготовка входных данных

Создаем небольшой локальный файл:

```bash
printf "alpha beta
beta gamma
alpha
" > ~/yarn-smoke.txt
```

Получаем содержимое:

```text
alpha beta
beta gamma
alpha
```

Создаем отдельный каталог для теста в HDFS:

```bash
hdfs dfs -rm -r -f /tmp/team28a-yarn-smoke 2>/dev/null || true
hdfs dfs -mkdir -p /tmp/team28a-yarn-smoke/input
```

Загружаем входной файл:

```bash
hdfs dfs -put ~/yarn-smoke.txt   /tmp/team28a-yarn-smoke/input/input.txt
```

Проверяем его в HDFS:

```bash
hdfs dfs -cat /tmp/team28a-yarn-smoke/input/input.txt
```

Получаем:

```text
alpha beta
beta gamma
alpha
```

### Запуск WordCount

Находим JAR с примерами MapReduce:

```bash
EXAMPLES_JAR=$(ls   "$HADOOP_HOME"/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar   | head -n 1)

echo "$EXAMPLES_JAR"
```

Запускаем WordCount:

```bash
hadoop jar "$EXAMPLES_JAR"   wordcount   /tmp/team28a-yarn-smoke/input   /tmp/team28a-yarn-smoke/output
```

Задание завершается со статусом:

```text
SUCCEEDED
```

### Проверка результата

Читаем результат из HDFS:

```bash
hdfs dfs -cat   /tmp/team28a-yarn-smoke/output/part-r-00000
```

Получили:

```text
alpha   2
beta    2
gamma   1
```

Результат соответствует исходному файлу. После этого дополнительно проверяем отсутствие выполняющихся приложений:

```bash
yarn application -list -appStates RUNNING
```

Получили:

```text
Total number of applications (application-types: [], states: [RUNNING] and tags: []):0
```

### Очистка тестовых данных

После успешной проверки удаляем тестовые данные:

```bash
hdfs dfs -rm -r -f /tmp/team28a-yarn-smoke
rm -f ~/yarn-smoke.txt
```

Возвращаемся на edge-узел:

```bash
exit
```

## 5. Выбор портов и каталогов Hive

Перед установкой PostgreSQL и Hive проверяем свободные TCP-порты. Виртуальные машины используются двумя командами, поэтому для сервисов `team28a` выбираются отдельные порты, не пересекающиеся с уже работающими сервисами.

На `team-28-nn` проверяем порты Hive Metastore, HiveServer2 и Web UI HiveServer2:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn   'ss -ltn | grep -E ":(29083|29000|29002)([[:space:]]|$)" || true'
```

Строки `LISTEN` для этих портов отсутствовали, поэтому они были выбраны.

На `team-28-01` проверяем порт PostgreSQL:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-01   'ss -ltn | grep -E ":(29432)([[:space:]]|$)" || true'
```

Порт `29432` был свободен и выбран для PostgreSQL.

Итоговое распределение сервисов:

```text
team-28-nn
├── Hive Metastore :29083
├── HiveServer2    :29000
└── HS2 Web UI     :29002

team-28-01
└── PostgreSQL     :29432
```

Для установки и служебных данных используются каталоги внутри домашнего каталога пользователя `team28a`.

На `team-28-nn`:

```text
/home/team28a/apps/hive
/home/team28a/logs
/home/team28a/hw3
```

На `team-28-01`:

```text
/home/team28a/apps/postgresql
/home/team28a/downloads/postgresql16
/home/team28a/postgresql/data
/home/team28a/postgresql/run
/home/team28a/postgresql/log
```

Hive и PostgreSQL разворачиваются без `sudo` и используют отдельные пользовательские каталоги и порты команды `team28a`.

## 6. Установка PostgreSQL

Для хранения метаданных Hive используется PostgreSQL на узле `team-28-01`. Пользователь `team28a` не имеет прав `sudo`, поэтому PostgreSQL устанавливается в домашний каталог путем загрузки и распаковки Debian-пакетов.

Подключаемся к `team-28-01`:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-01
```

### Загрузка пакетов

Создаем каталоги для загрузки и установки:

```bash
mkdir -p ~/downloads/postgresql16 ~/apps/postgresql
cd ~/downloads/postgresql16
```

Загружаем PostgreSQL 16, клиент и библиотеку `libpq`:

```bash
apt download postgresql-16 postgresql-client-16 libpq5
```

В каталоге `~/downloads/postgresql16` получены три пакета:

```text
postgresql-16_16.15-0ubuntu0.24.04.1_amd64.deb
postgresql-client-16_16.15-0ubuntu0.24.04.1_amd64.deb
libpq5_16.15-0ubuntu0.24.04.1_amd64.deb
```

### Распаковка пакетов

Распаковываем все загруженные пакеты в пользовательский каталог:

```bash
for pkg in ./*.deb; do
  dpkg-deb -x "$pkg" ~/apps/postgresql
done
```

После распаковки серверные бинарные файлы PostgreSQL находятся в:

```text
/home/team28a/apps/postgresql/usr/lib/postgresql/16
```

а необходимые динамические библиотеки — в:

```text
/home/team28a/apps/postgresql/usr/lib/x86_64-linux-gnu
```

### Переменные окружения PostgreSQL

Для текущей сессии задаем:

```bash
export PG_HOME=$HOME/apps/postgresql/usr/lib/postgresql/16
export LD_LIBRARY_PATH=$HOME/apps/postgresql/usr/lib/x86_64-linux-gnu
```

Проверяем установленную версию сервера:

```bash
$PG_HOME/bin/postgres --version
```

Получили:

```text
postgres (PostgreSQL) 16.15
```

Проверяем клиент:

```bash
$PG_HOME/bin/psql --version
```

Получили:

```text
psql (PostgreSQL) 16.15
```

## 7. Настройка PostgreSQL для Hive Metastore

После установки PostgreSQL на `team-28-01` создаем отдельный экземпляр базы данных для Hive Metastore. Продолжаем работу на `team-28-01`. Переменные окружения PostgreSQL, заданные в предыдущем разделе, остаются активны в текущей сессии.

### Создание каталогов PostgreSQL

Создаем каталоги для данных, Unix-сокета и логов:

```bash
mkdir -p ~/postgresql/data          ~/postgresql/run          ~/postgresql/log
```

Используются пути:

```text
/home/team28a/postgresql/data
/home/team28a/postgresql/run
/home/team28a/postgresql/log
```

### Инициализация экземпляра PostgreSQL

Инициализируем новый кластер PostgreSQL:

```bash
$PG_HOME/bin/initdb   -D $HOME/postgresql/data   --encoding=UTF8   --locale=C.UTF-8   --auth-local=trust   --auth-host=scram-sha-256
```

Служебные файлы PostgreSQL созданы в:

```text
/home/team28a/postgresql/data
```

Для локальных подключений используется `trust`, а для сетевых подключений — `scram-sha-256`.

### Настройка `postgresql.conf`

В файл:

```text
~/postgresql/data/postgresql.conf
```

добавляем параметры:

```bash
cat >> ~/postgresql/data/postgresql.conf <<'EOF'

listen_addresses = '10.28.0.13'
port = 29432
unix_socket_directories = '/home/team28a/postgresql/run'
EOF
```

PostgreSQL принимает сетевые подключения по внутреннему адресу:

```text
10.28.0.13:29432
```

и создает Unix-сокет в каталоге:

```text
/home/team28a/postgresql/run
```

### Настройка `pg_hba.conf`

Разрешаем подключение к базе Hive Metastore с внутренних узлов кластера.

Добавляем в:

```text
~/postgresql/data/pg_hba.conf
```

следующие правила:

```bash
cat >> ~/postgresql/data/pg_hba.conf <<'EOF'

host    metastore    hive    10.28.0.10/32    scram-sha-256
host    metastore    hive    10.28.0.11/32    scram-sha-256
host    metastore    hive    10.28.0.12/32    scram-sha-256
host    metastore    hive    10.28.0.13/32    scram-sha-256
EOF
```

Пользователь `hive` получает сетевой доступ только к базе `metastore` с четырех внутренних адресов инфраструктуры `team-28`.

### Запуск PostgreSQL

Запускаем PostgreSQL:

```bash
$PG_HOME/bin/pg_ctl   -D $HOME/postgresql/data   -l $HOME/postgresql/log/postgresql.log   start
```

Проверяем порт:

```bash
ss -ltn | grep ':29432'
```

Получили:

```text
10.28.0.13:29432
```

### Создание пользователя и базы Hive Metastore

Подключаемся к PostgreSQL через локальный Unix-сокет:

```bash
$PG_HOME/bin/psql   -h $HOME/postgresql/run   -p 29432   -U team28a   -d postgres
```

Создаем отдельного пользователя Hive:

```sql
CREATE ROLE hive LOGIN PASSWORD '<HIVE_DB_PASSWORD>';
```
Репозиторий является публичным, поэтому фактический пароль пользователя PostgreSQL `hive` в инструкции не публикуется. Вместо него используется `<HIVE_DB_PASSWORD>`. При воспроизведении конфигурации его необходимо заменить на собственный пароль и использовать в дальнейшей работе.

Создаем базу `metastore` и назначаем владельцем пользователя `hive`:

```sql
CREATE DATABASE metastore OWNER hive;
```

Выходим из `psql`:

```text
\q
```

Проверяем созданные объекты:

```bash
$PG_HOME/bin/psql   -h $HOME/postgresql/run   -p 29432   -U team28a   -d postgres   -c "SELECT rolname FROM pg_roles WHERE rolname='hive';"

$PG_HOME/bin/psql   -h $HOME/postgresql/run   -p 29432   -U team28a   -d postgres   -c "SELECT datname FROM pg_database WHERE datname='metastore';"
```

Получили пользователя:

```text
hive
```

и базу:

```text
metastore
```

### Проверка сетевой доступности PostgreSQL

С `team-28-nn` проверяем соединение с портом PostgreSQL:

```bash
exit
```

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn   'timeout 3 bash -c "</dev/tcp/10.28.0.13/29432" && echo POSTGRES_REACHABLE'
```

Получили:

```text
POSTGRES_REACHABLE
```

Теперь PostgreSQL готов к использованию в качестве внешней базы данных Hive Metastore.

## 8. Установка Apache Hive

Apache Hive устанавливается на `team-28-nn`, где далее запускаются Hive Metastore и HiveServer2.

Используем рекомендованную в задании версию:

```text
Apache Hive 4.0.0-alpha-2
```

Подключаемся к `team-28-nn`:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn
```

### Подготовка каталогов

Создаем каталоги для загрузки, установки и логов:

```bash
mkdir -p ~/downloads ~/apps/hive ~/logs
cd ~/downloads
```

### Загрузка Hive

Скачиваем архив Hive `4.0.0-alpha-2`:

```bash
curl -fL --retry 3   -o apache-hive-4.0.0-alpha-2-bin.tar.gz   https://archive.apache.org/dist/hive/hive-4.0.0-alpha-2/apache-hive-4.0.0-alpha-2-bin.tar.gz
```

Проверяем архив:

```bash
gzip -t apache-hive-4.0.0-alpha-2-bin.tar.gz   && echo "HIVE_ARCHIVE_OK"
```

Получили:

```text
HIVE_ARCHIVE_OK
```

### Распаковка Hive

Распаковываем архив в пользовательский каталог:

```bash
tar -xzf apache-hive-4.0.0-alpha-2-bin.tar.gz   --strip-components=1   -C ~/apps/hive
```

Hive установлен в:

```text
/home/team28a/apps/hive
```

### Переменные окружения

Для дальнейшей настройки задаем:

```bash
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export HIVE_HOME=$HOME/apps/hive
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HIVE_HOME/bin:$PATH
```

Проверяем пути:

```bash
echo "$JAVA_HOME"
echo "$HADOOP_HOME"
echo "$HIVE_HOME"
```

Получили:

```text
/home/team28a/apps/java
/home/team28a/apps/hadoop
/home/team28a/apps/hive
```

### Проверка версии Hive

Проверяем установленную версию:

```bash
hive --version
```

Получили:

```text
Hive 4.0.0-alpha-2
```

## 9. Настройка PostgreSQL JDBC Driver

Для подключения Hive Metastore к PostgreSQL необходим JDBC-драйвер PostgreSQL. Драйвер размещается в каталоге библиотек Hive на `team-28-nn`. Продолжаем работу на `team-28-nn` с переменными окружения, заданными при установке Hive.

### Загрузка JDBC-драйвера

Переходим в каталог библиотек Hive:

```bash
cd "$HIVE_HOME/lib"
```

Загружаем PostgreSQL JDBC Driver:

```bash
curl -fL --retry 3   -o postgresql-42.7.14.jar   https://jdbc.postgresql.org/download/postgresql-42.7.14.jar
```

Проверяем наличие PostgreSQL JDBC-библиотек:

```bash
ls -lh "$HIVE_HOME"/lib/postgresql-*.jar
```

В рабочей установке присутствуют:

```text
postgresql-42.4.1.jar
postgresql-42.7.14.jar
```

### Проверка класса JDBC Driver

Проверяем наличие основного класса PostgreSQL-драйвера в загруженном JAR:

```bash
jar tf "$HIVE_HOME/lib/postgresql-42.7.14.jar"   | grep '^org/postgresql/Driver.class$'
```

Получили:

```text
org/postgresql/Driver.class
```

Для дальнейшей конфигурации Hive используется имя JDBC driver:

```text
org.postgresql.Driver
```

## 10. Настройка `hive-site.xml`

Основная конфигурация Hive задается на `team-28-nn` в файле:

```text
/home/team28a/apps/hive/conf/hive-site.xml
```

В конфигурации указываются:

- внешний PostgreSQL Metastore;
- каталог Hive warehouse в HDFS;
- адрес отдельного Hive Metastore;
- параметры HiveServer2;
- MapReduce как движок выполнения.

Создаем `hive-site.xml` (перед созданием необходимо заменить <HIVE_DB_PASSWORD> на фактический пароль пользователя hive, как упоминалось выше):

```bash
cat > "$HIVE_HOME/conf/hive-site.xml" <<'EOF'
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>

    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:postgresql://10.28.0.13:29432/metastore</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.postgresql.Driver</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
    </property>

    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value><HIVE_DB_PASSWORD></value>
    </property>

    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>

    <property>
        <name>hive.metastore.uris</name>
        <value>thrift://10.28.0.11:29083</value>
    </property>

    <property>
        <name>hive.server2.thrift.bind.host</name>
        <value>10.28.0.11</value>
    </property>

    <property>
        <name>hive.server2.thrift.port</name>
        <value>29000</value>
    </property>

    <property>
        <name>hive.server2.webui.port</name>
        <value>29002</value>
    </property>

    <property>
        <name>hive.server2.authentication</name>
        <value>NONE</value>
    </property>

    <property>
        <name>hive.server2.enable.doAs</name>
        <value>false</value>
    </property>

    <property>
        <name>hive.execution.engine</name>
        <value>mr</value>
    </property>

    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>

</configuration>
EOF
```

### Основные параметры конфигурации

Подключение к внешнему PostgreSQL выполняется по адресу:

```text
jdbc:postgresql://10.28.0.13:29432/metastore
```

Используются:

```text
Driver: org.postgresql.Driver
User:   hive
```

Hive warehouse размещается в HDFS:

```text
/user/hive/warehouse
```

Hive Metastore работает как отдельный Thrift-сервис:

```text
thrift://10.28.0.11:29083
```

HiveServer2 принимает клиентские подключения по адресу:

```text
10.28.0.11:29000
```

Web UI HiveServer2 использует порт:

```text
29002
```

### Проверка XML

Проверяем синтаксис созданного файла:

```bash
python3 - <<'PY'
import xml.etree.ElementTree as ET

ET.parse("/home/team28a/apps/hive/conf/hive-site.xml")
print("HIVE_SITE_XML_OK")
PY
```

Получили:

```text
HIVE_SITE_XML_OK
```

Embedded-подход в данной конфигурации согласно заданию не используется.

## 11. Подготовка HDFS warehouse

Перед инициализацией Hive Metastore создаем каталог warehouse в HDFS. Все команды продолжаем выполнять на `team-28-nn`.

Создаем каталог Hive warehouse:

```bash
hdfs dfs -mkdir -p /user/hive/warehouse
```

Проверяем созданный каталог:

```bash
hdfs dfs -ls -d /user/hive/warehouse
```

В HDFS создан каталог:

```text
/user/hive/warehouse
```

Именно этот путь ранее был указан в `hive-site.xml`:

```xml
<property>
    <name>hive.metastore.warehouse.dir</name>
    <value>/user/hive/warehouse</value>
</property>
```

## 12. Инициализация схемы и запуск Hive Metastore

После настройки PostgreSQL, JDBC Driver, `hive-site.xml` и HDFS warehouse инициализируем схему Hive Metastore в базе PostgreSQL `metastore`. Все команды выполняются на `team-28-nn`.

### Инициализация схемы

Запускаем `schematool` для PostgreSQL:

```bash
schematool -dbType postgres -initSchema
```

Команда создает служебные таблицы Hive Metastore в базе:

```text
metastore
```

на PostgreSQL-сервере:

```text
10.28.0.13:29432
```

Инициализация выполняется один раз для новой базы Metastore.

### Проверка схемы

После инициализации проверяем информацию о Metastore:

```bash
schematool -dbType postgres -info
```

Получили:

```text
Metastore connection URL: jdbc:postgresql://10.28.0.13:29432/metastore
Metastore Connection Driver : org.postgresql.Driver
Metastore connection User: hive
Hive distribution version: 4.0.0-alpha-2
Metastore schema version: 4.0.0-alpha-2
```

Версия установленного Hive и версия схемы Metastore совпадают:

```text
Hive distribution version: 4.0.0-alpha-2
Metastore schema version: 4.0.0-alpha-2
```


### Запуск

Создаем каталог для логов:

```bash
mkdir -p ~/logs
```

Запускаем Hive Metastore в фоновом режиме:

```bash
nohup hive --service metastore -p 29083   > ~/logs/hive-metastore-final.log 2>&1 < /dev/null &
```

Hive Metastore использует порт:

```text
29083
```

### Проверка процесса

Проверяем Java-процессы:

```bash
jps
```

В выводе присутствует отдельный процесс Hive Metastore, запущенный через `RunJar`.

Дополнительно проверяем командную строку процесса:

```bash
ps -u "$USER" -o pid,args   | grep '[H]iveMetaStore'
```

Получаем процесс Hive Metastore с параметром:

```text
-p 29083
```

### Проверка порта

Проверяем, что Metastore слушает настроенный порт:

```bash
ss -ltn | grep ':29083'
```

Получили слушающий порт:

```text
29083
```

### Проверка лога

Проверяем последние строки лога сервиса:

```bash
tail -n 30 ~/logs/hive-metastore-final.log
```

После запуска Hive Metastore продолжает работать как отдельный сетевой сервис и использует внешний PostgreSQL для хранения метаданных.

## 13. Запуск HiveServer2

После запуска Hive Metastore запускаем HiveServer2 на `team-28-nn`. HiveServer2 используется как клиентская точка доступа к Hive и позволяет одновременно подключать несколько клиентов.

Запускаем HiveServer2 в фоновом режиме:

```bash
nohup hive   --hiveconf hive.server2.enable.doAs=false   --hiveconf hive.security.authorization.enabled=false   --service hiveserver2   > ~/logs/hive-server2-final.log 2>&1 < /dev/null &
```

HiveServer2 использует:

```text
29000 — клиентский Thrift-порт
29002 — Web UI
```

### Проверка процесса

Проверяем Java-процессы:

```bash
jps
```

В выводе присутствует отдельный процесс HiveServer2, запущенный через `RunJar`.

Дополнительно проверяем командную строку процесса:

```bash
ps -u "$USER" -o pid,args   | grep '[H]iveServer2'
```

Получаем процесс HiveServer2 с параметрами:

```text
hive.server2.enable.doAs=false
hive.security.authorization.enabled=false
```

### Проверка портов

Проверяем клиентский порт и Web UI:

```bash
ss -ltn | grep -E ':(29000|29002)([[:space:]]|$)'
```

Получили два слушающих порта:

```text
29000
29002
```

### Проверка лога

Проверяем последние строки лога HiveServer2:

```bash
tail -n 30 ~/logs/hive-server2-final.log
```

После запуска HiveServer2 доступен клиентам по адресу:

```text
10.28.0.11:29000
```

а его Web UI использует порт:

```text
29002
```

Итак, клиентский сервис Hive запущен отдельно от Hive Metastore и готов к подключению через Beeline.

## 14. Проверка подключения через Beeline

Продолжаем работу на `team-28-nn`.

Подключаемся к HiveServer2:

```bash
beeline   -u 'jdbc:hive2://10.28.0.11:29000/default'   -n team28a
```

После подключения Beeline использует HiveServer2 по адресу:

```text
10.28.0.11:29000
```

Для проверки выполняем запрос, не зависящий от созданных позднее баз данных:

```sql
SELECT current_database();
```

Текущая база данных:

```text
default
```

Версия сервера при подключении:

```text
Apache Hive 4.0.0-alpha-2
```

Завершаем клиентскую сессию:

```text
!quit
```

## 15. Проверка одновременной работы двух клиентов

По условию домашнего задания Hive должен поддерживать одновременную работу более чем одного клиента. Для проверки открываем две независимые Beeline-сессии и оставляем их подключенными одновременно.

Используются два терминала, подключенных к `team-28-nn`.

### Клиент 1

В первом терминале задаем переменные окружения:

```bash
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export HIVE_HOME=$HOME/apps/hive
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HIVE_HOME/bin:$PATH
```

Запускаем Beeline:

```bash
beeline   -u 'jdbc:hive2://10.28.0.11:29000/default'   -n team28a
```

Выполняем:

```sql
SELECT 'client_1' AS client, current_database() AS current_db;
```

Получили строку:

```text
client_1    default
```

Сессию оставляем открытой.

### Клиент 2

Во втором терминале задаем те же переменные окружения:

```bash
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export HIVE_HOME=$HOME/apps/hive
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HIVE_HOME/bin:$PATH
```

Запускаем второй экземпляр Beeline:

```bash
beeline   -u 'jdbc:hive2://10.28.0.11:29000/default'   -n team28a
```

Во второй сессии выполняем:

```sql
SELECT 'client_2' AS client, current_database() AS current_db;
```

Получили:

```text
client_2    default
```

Обе Beeline-сессии были подключены к одному HiveServer2 одновременно и независимо выполнили запросы.

После проверки завершаем обе сессии командой:

```text
!quit
```

## 16. Создание базы и таблиц Hive

Создаем отдельную базу `hw3` и две таблицы:

- `events_staging` — таблица со структурой исходного CSV;
- `events_partitioned` — итоговая таблица, разбитая на партиции по `event_date`.

Все команды выполняются через Beeline на `team-28-nn`.

Подключаемся:

```bash
beeline   -u 'jdbc:hive2://10.28.0.11:29000/default'   -n team28a
```

### Создание базы

Создаем базу:

```sql
CREATE DATABASE IF NOT EXISTS hw3;
```

Проверяем список баз:

```sql
SHOW DATABASES;
```

Получили:

```text
default
hw3
```

### Создание staging-таблицы

Создаем внешнюю таблицу со структурой исходных данных:

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS hw3.events_staging (
    event_id INT,
    user_name STRING,
    amount DECIMAL(10,2),
    event_date STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION 'hdfs://team-28-nn:9000/user/hive/warehouse/hw3.db/events_staging';
```

В staging-таблице дата хранится как обычный столбец.

### Создание партиционированной таблицы

Создаем внешнюю таблицу, в которой `event_date` используется как partition key:

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS hw3.events_partitioned (
    event_id INT,
    user_name STRING,
    amount DECIMAL(10,2)
)
PARTITIONED BY (
    event_date STRING
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
LOCATION 'hdfs://team-28-nn:9000/user/hive/warehouse/hw3.db/events_partitioned';
```

В этой таблице поле `event_date` не хранится внутри строк файлов данных: значение даты определяется каталогом соответствующей партиции.

### Проверка таблиц

Проверяем список таблиц:

```sql
SHOW TABLES IN hw3;
```

Получили:

```text
events_partitioned
events_staging
```

Проверяем структуру staging-таблицы:

```sql
DESCRIBE hw3.events_staging;
```

Проверяем структуру партиционированной таблицы:

```sql
DESCRIBE hw3.events_partitioned;
```

Для `events_partitioned` Hive отдельно отображает столбец партиционирования:

```text
event_date    string
```

После создания таблиц завершаем Beeline-сессию:

```text
!quit
```

## 17. Автоматизированная загрузка данных в партиции

По условию домашнего задания необходимо реализовать автоматизированную загрузку данных в партиционированную таблицу Hive.

Для этого исходный CSV автоматически разбивается по значению `event_date`, после чего для каждой даты генерируется отдельная команда `LOAD DATA LOCAL ... PARTITION`.

Такой способ не требует тяжелого `INSERT ... SELECT` и выполняет загрузку через статические партиции.

Все команды выполняются на `team-28-nn`.

### Подготовка исходного CSV

Создаем рабочий каталог:

```bash
mkdir -p ~/hw3
```

Создаем файл:

```bash
cat > ~/hw3/events.csv <<'EOF'
1,alice,120.50,2026-10-14
2,bob,75.00,2026-10-14
3,carol,210.25,2026-10-14
4,dave,99.90,2026-10-15
5,alice,310.00,2026-10-15
6,bob,45.50,2026-10-15
7,carol,180.00,2026-10-16
8,dave,125.75,2026-10-16
9,alice,260.40,2026-10-16
EOF
```

Исходный формат строки:

```text
event_id,user_name,amount,event_date
```

В файле содержатся данные за три даты:

```text
2026-10-14
2026-10-15
2026-10-16
```

### Автоматическое разбиение по датам

Создаем каталог для файлов партиций:

```bash
SOURCE=$HOME/hw3/events.csv
PARTDIR=$HOME/hw3/partition-load

rm -rf "$PARTDIR"
mkdir -p "$PARTDIR"
```

Разбиваем CSV по четвертому полю `event_date`:

```bash
awk -F',' -v OFS=',' -v out="$PARTDIR" '
{
    file = out "/" $4 ".csv"
    print $1, $2, $3 > file
}
' "$SOURCE"
```

В результирующие файлы записываются только первые три поля:

```text
event_id,user_name,amount
```

Поле `event_date` не записывается внутрь файла, поскольку оно хранится как значение partition key.

Проверяем созданные файлы:

```bash
for file in "$PARTDIR"/*.csv; do
  echo "===== $(basename "$file") ====="
  cat "$file"
done
```

Получены три файла:

```text
2026-10-14.csv
2026-10-15.csv
2026-10-16.csv
```

Каждый файл содержит по три строки.

### Автоматическая генерация SQL

Создаем SQL-файл:

```bash
SQLFILE=$HOME/hw3/load-partitions.sql
: > "$SQLFILE"
```

Для каждого автоматически созданного CSV генерируем команду загрузки в соответствующую партицию:

```bash
for FILE in "$PARTDIR"/*.csv; do
  DATE=$(basename "$FILE" .csv)

  printf "LOAD DATA LOCAL INPATH '%s' OVERWRITE INTO TABLE hw3.events_partitioned PARTITION (event_date='%s');\n"     "$FILE" "$DATE" >> "$SQLFILE"
done
```

Проверяем сгенерированный SQL:

```bash
cat "$SQLFILE"
```

Получили:

```sql
LOAD DATA LOCAL INPATH '/home/team28a/hw3/partition-load/2026-10-14.csv' OVERWRITE INTO TABLE hw3.events_partitioned PARTITION (event_date='2026-10-14');
LOAD DATA LOCAL INPATH '/home/team28a/hw3/partition-load/2026-10-15.csv' OVERWRITE INTO TABLE hw3.events_partitioned PARTITION (event_date='2026-10-15');
LOAD DATA LOCAL INPATH '/home/team28a/hw3/partition-load/2026-10-16.csv' OVERWRITE INTO TABLE hw3.events_partitioned PARTITION (event_date='2026-10-16');
```

### Выполнение загрузки

Выполняем автоматически сгенерированный SQL через Beeline:

```bash
beeline   -u 'jdbc:hive2://10.28.0.11:29000/default'   -n team28a   -f "$SQLFILE"
```

Все три команды `LOAD DATA LOCAL` завершились успешно.

Hive создал три статические партиции и загрузил соответствующие данные в HDFS-каталоги таблицы `events_partitioned`.

После загрузки сохраняем рабочие артефакты:

```text
~/hw3/events.csv
~/hw3/partition-load/
~/hw3/load-partitions.sql
```

## 18. Проверка партиций и данных

После выполнения автоматизированной загрузки проверяем, что Hive зарегистрировал все созданные партиции, а данные физически находятся в HDFS. Все команды выполняются на `team-28-nn`.

### Проверка зарегистрированных партиций

Через Beeline выполняем:

```bash
beeline   -u 'jdbc:hive2://10.28.0.11:29000/default'   -n team28a   -e 'SHOW PARTITIONS hw3.events_partitioned;'
```

Получили три партиции:

```text
event_date=2026-10-14
event_date=2026-10-15
event_date=2026-10-16
```

### Проверка структуры каталогов в HDFS

Проверяем каталог партиционированной таблицы:

```bash
hdfs dfs -ls   /user/hive/warehouse/hw3.db/events_partitioned
```

Получили три отдельных каталога:

```text
/user/hive/warehouse/hw3.db/events_partitioned/event_date=2026-10-14
/user/hive/warehouse/hw3.db/events_partitioned/event_date=2026-10-15
/user/hive/warehouse/hw3.db/events_partitioned/event_date=2026-10-16
```

### Проверка данных каждой партиции

Читаем данные напрямую из HDFS:

```bash
for DATE in 2026-10-14 2026-10-15 2026-10-16; do
  echo "===== event_date=$DATE ====="
  hdfs dfs -cat     "/user/hive/warehouse/hw3.db/events_partitioned/event_date=$DATE/*"
done
```

Для партиции `2026-10-14` получили:

```text
1,alice,120.50
2,bob,75.00
3,carol,210.25
```

Для партиции `2026-10-15`:

```text
4,dave,99.90
5,alice,310.00
6,bob,45.50
```

Для партиции `2026-10-16`:

```text
7,carol,180.00
8,dave,125.75
9,alice,260.40
```

Поле `event_date` не хранится внутри файлов, поскольку его значение определяется именем каталога партиции. Проверка подтверждает, что автоматизированная загрузка создала три партиции и корректно разместила данные по соответствующим HDFS-каталогам.

## 19. Финальная проверка сервисов

После завершения настройки Hive и загрузки данных выполняем финальную проверку состояния кластера. Для этого возвращаемся на edge-узел:

```bash
exit
```

### Проверка процессов

С edge-узла проверяем Java-процессы на всех трех внутренних машинах:

```bash
for host in team-28-nn team-28-00 team-28-01; do
  echo "===== $host ====="
  ssh -i ~/.ssh/team28a_internal team28a@$host     'JAVA_HOME=$HOME/apps/java $HOME/apps/java/bin/jps'
  echo
done
```

На `team-28-nn` работают:

```text
NameNode
DataNode
ResourceManager
NodeManager
JobHistoryServer
RunJar
RunJar
```

Два процесса `RunJar` соответствуют Hive Metastore и HiveServer2.

На `team-28-00` работают:

```text
SecondaryNameNode
DataNode
NodeManager
```

На `team-28-01` работают:

```text
DataNode
NodeManager
```

### Проверка PostgreSQL и Hive-портов

На `team-28-01` проверяем PostgreSQL:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-01   'ss -ltn | grep ":29432"'
```

PostgreSQL слушает адрес:

```text
10.28.0.13:29432
```

На `team-28-nn` проверяем Hive Metastore и HiveServer2:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn   'ss -ltn | grep -E ":(29083|29000|29002)([[:space:]]|$)"'
```

Активные порты Hive:

```text
29083
29000
29002
```

### Проверка HDFS

Для проверки HDFS, YARN и Hive снова подключаемся к `team-28-nn`:

```bash
ssh -i ~/.ssh/team28a_internal team28a@team-28-nn
```

Задаем переменные окружения:

```bash
export JAVA_HOME=$HOME/apps/java
export HADOOP_HOME=$HOME/apps/hadoop
export HIVE_HOME=$HOME/apps/hive
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$HIVE_HOME/bin:$PATH
```

Проверяем состояние HDFS:

```bash
hdfs dfsadmin -safemode get

hdfs dfsadmin -report   | grep -E     'Under replicated blocks:|Blocks with corrupt replicas:|Missing blocks:|Live datanodes'
```

Состояние HDFS:

```text
Safe mode is OFF
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0
Live datanodes (3)
```

Дополнительно проверяем файловую систему:

```bash
hdfs fsck /
```

Проверка файловой системы:

```text
Status: HEALTHY
The filesystem under path '/' is HEALTHY
```

### Проверка YARN

Проверяем зарегистрированные NodeManager:

```bash
yarn node -list -showDetails
```

ResourceManager зарегистрировал:

```text
Total Nodes:3
```

Все три NodeManager находятся в состоянии:

```text
RUNNING
```

Проверяем отсутствие выполняющихся приложений:

```bash
yarn application -list -appStates RUNNING
```

Запущенных YARN-приложений:

```text
Total number of applications (application-types: [], states: [RUNNING] and tags: []):0
```

### Проверка Hive

Через HiveServer2 выполняем:

```bash
beeline   -u 'jdbc:hive2://10.28.0.11:29000/default'   -n team28a   -e 'SHOW DATABASES; SHOW TABLES IN hw3; SHOW PARTITIONS hw3.events_partitioned;'
```

Hive возвращает:
```text
default
hw3
```

Таблицы:

```text
events_partitioned
events_staging
```

Партиции:

```text
event_date=2026-10-14
event_date=2026-10-15
event_date=2026-10-16
```

## 20. Итог

Apache Hive `4.0.0-alpha-2` развернут поверх существующего HDFS/YARN-кластера команды `team28a`. Для Hive Metastore используется отдельная база PostgreSQL `metastore`. Метаданные Hive не хранятся во встроенной Derby-базе, поэтому embedded-подход не используется.
