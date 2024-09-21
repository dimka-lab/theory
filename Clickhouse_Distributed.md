**Создание шардированных таблиц в ClickHouse с использованием движка Distributed**

---

### **Введение**

ClickHouse — это высокопроизводительная колоночная СУБД, которая поддерживает горизонтальное масштабирование через шардирование и репликацию. Для реализации шардирования используется движок **Distributed**, который распределяет запросы и данные по указанным шардам.

---

### **Шаг 1: Настройка кластера**

Перед созданием шардированных таблиц необходимо настроить конфигурацию кластера. Это делается в файле конфигурации `clusters.xml`, который обычно находится в `/etc/clickhouse-server/config.d/` или может быть добавлен в основной конфигурационный файл `config.xml`.

**Пример конфигурации кластера:**

```xml
<clickhouse>
    <remote_servers>
        <my_cluster>
            <shard>
                <replica>
                    <host>host1</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>host2</host>
                    <port>9000</port>
                </replica>
            </shard>
            <shard>
                <replica>
                    <host>host3</host>
                    <port>9000</port>
                </replica>
                <replica>
                    <host>host4</host>
                    <port>9000</port>
                </replica>
            </shard>
        </my_cluster>
    </remote_servers>
</clickhouse>
```

В этом примере кластер `my_cluster` состоит из двух шардов, каждый из которых имеет две реплики.

**Важно:** После изменения конфигурации сервера необходимо перезапустить ClickHouse Server.

---

### **Шаг 2: Создание локальных таблиц на каждом шарде**

На каждом сервере создайте локальную таблицу, используя движок **MergeTree** или его вариации.

**Пример создания локальной таблицы:**

```sql
CREATE TABLE default.local_table ON CLUSTER my_cluster
(
    event_date Date,
    event_time DateTime,
    user_id UInt32,
    event_type String,
    event_value Float32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);
```

**Объяснение:**

- **ON CLUSTER my_cluster**: Эта команда создаст таблицу на всех узлах кластера `my_cluster`.
- **ENGINE = MergeTree()**: Используется движок MergeTree, оптимизированный для больших объемов данных.
- **PARTITION BY**: Указывает, как будут разбиваться данные на партиции.
- **ORDER BY**: Определяет порядок сортировки данных внутри партиции.

---

### **Шаг 3: Создание Distributed таблицы**

Distributed таблица является логической таблицей, которая объединяет данные из локальных таблиц на разных шардах и репликах.

**Пример создания Distributed таблицы:**

```sql
CREATE TABLE default.distributed_table ON CLUSTER my_cluster AS default.local_table
ENGINE = Distributed(
    'my_cluster',     -- имя кластера из конфигурации
    'default',        -- база данных локальных таблиц
    'local_table',    -- имя локальной таблицы
    user_id           -- ключ шардирования
);
```

**Объяснение:**

- **AS default.local_table**: Структура таблицы совпадает с локальной таблицей `local_table`.
- **ENGINE = Distributed(...)**: Указывает, что это Distributed таблица.
- **'my_cluster'**: Имя кластера из конфигурации.
- **'default'**: База данных, в которой находятся локальные таблицы.
- **'local_table'**: Имя локальной таблицы.
- **user_id**: Ключ шардирования; определяет, на какой шард попадет строка.

---

### **Шаг 4: Вставка данных**

**Вариант 1: Вставка через Distributed таблицу**

Вы можете вставлять данные напрямую в Distributed таблицу — ClickHouse автоматически распределит данные по шардам.

**Пример:**

```sql
INSERT INTO distributed_table (event_date, event_time, user_id, event_type, event_value) VALUES
('2023-01-01', '2023-01-01 12:00:00', 123, 'click', 1.0),
('2023-01-02', '2023-01-02 13:00:00', 456, 'view', 2.0);
```

**Вариант 2: Вставка в локальные таблицы**

Можно вставлять данные непосредственно в локальные таблицы на конкретных узлах. Это полезно для загрузки данных в оффлайн-режиме или при миграции данных.

---

### **Шаг 5: Запросы к данным**

**Запрос через Distributed таблицу**

Вы можете выполнять запросы к Distributed таблице, и ClickHouse сам распределит запрос по шардам и объединит результаты.

**Пример:**

```sql
SELECT
    event_type,
    COUNT(*) AS event_count,
    AVG(event_value) AS average_value
FROM
    distributed_table
WHERE
    event_date >= '2023-01-01' AND event_date <= '2023-01-31'
GROUP BY
    event_type;
```

---

### **Полный пример**

**1. Создание локальной таблицы на всех узлах:**

```sql
CREATE TABLE default.local_table ON CLUSTER my_cluster
(
    event_date Date,
    event_time DateTime,
    user_id UInt32,
    event_type String,
    event_value Float32
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(event_date)
ORDER BY (event_date, user_id);
```

**2. Создание Distributed таблицы:**

```sql
CREATE TABLE default.distributed_table ON CLUSTER my_cluster AS default.local_table
ENGINE = Distributed(
    'my_cluster',
    'default',
    'local_table',
    user_id
);
```

**3. Вставка данных:**

```sql
INSERT INTO distributed_table (event_date, event_time, user_id, event_type, event_value) VALUES
('2023-01-01', '2023-01-01 12:00:00', 123, 'click', 1.0),
('2023-01-02', '2023-01-02 13:00:00', 456, 'view', 2.0);
```

**4. Запрос данных:**

```sql
SELECT
    event_type,
    COUNT(*) AS event_count,
    AVG(event_value) AS average_value
FROM
    distributed_table
WHERE
    event_date BETWEEN '2023-01-01' AND '2023-01-31'
GROUP BY
    event_type;
```

---

### **Дополнительные детали**

#### **Шардирование**

Ключ шардирования (`user_id` в нашем примере) определяет, на какой шард попадут данные. Выбор правильного ключа шардирования важен для:

- **Равномерного распределения данных** по шардам.
- **Минимизации перекрестных запросов** между шардами.

#### **Репликация**

Если у вас настроена репликация, ClickHouse обеспечивает синхронизацию данных между репликами в рамках одного шарда. Это повышает отказоустойчивость и обеспечивает доступность данных.

#### **Параметры конфигурации**

- **insert_distributed_sync**: Если установить в `1`, вставки в Distributed таблицу будут синхронными, что гарантирует запись данных на все шарды до завершения операции.
- **insert_distributed_timeout**: Устанавливает таймаут для синхронных вставок.
- **prefer_localhost_replica**: Если включено, запросы будут предпочитать локальную реплику для минимизации сетевой нагрузки.

---

### **Практические советы**

- **Убедитесь, что все узлы имеют одинаковую конфигурацию таблиц**, чтобы избежать ошибок при выполнении запросов.
- **Выбирайте ключ шардирования с осторожностью**, основываясь на характеристиках ваших данных и запросов.
- **Используйте функционал DDL-запросов на кластере**, чтобы автоматически создавать таблицы на всех узлах:

```sql
CREATE TABLE database_name.table_name ON CLUSTER my_cluster
(
    -- описание столбцов
)
ENGINE = ...;
```

- **Мониторьте состояние кластера** с помощью системных таблиц:

```sql
SELECT * FROM system.clusters WHERE cluster = 'my_cluster';
SELECT * FROM system.tables WHERE name = 'local_table';
```

- **При необходимости используйте секционирование данных (partitioning)** для улучшения производительности при работе с историческими данными.

---

### **Пример с использованием Python и ClickHouse-driver**

Если вы хотите вставлять данные программно, можно использовать официальный Python-клиент `clickhouse-driver`.

**Установка библиотеки:**

```bash
pip install clickhouse-driver
```

**Пример кода:**

```python
from clickhouse_driver import Client

client = Client('host1')  # Подключение к одному из узлов кластера

# Вставка данных в Distributed таблицу
data = [
    ('2023-01-01', '2023-01-01 12:00:00', 123, 'click', 1.0),
    ('2023-01-02', '2023-01-02 13:00:00', 456, 'view', 2.0),
]

client.execute(
    'INSERT INTO default.distributed_table (event_date, event_time, user_id, event_type, event_value) VALUES',
    data
)

# Выполнение запроса
result = client.execute('''
    SELECT
        event_type,
        COUNT(*) AS event_count,
        AVG(event_value) AS average_value
    FROM
        default.distributed_table
    WHERE
        event_date BETWEEN '2023-01-01' AND '2023-01-31'
    GROUP BY
        event_type
''')

print(result)
```

---

### **Заключение**

Использование шардирования и движка **Distributed** в ClickHouse позволяет эффективно масштабировать систему для обработки больших объемов данных. Создание локальных таблиц на шардах и объединение их с помощью Distributed таблицы предоставляет прозрачный механизм для работы с распределенными данными.

**Ключевые моменты:**

- **Настройка кластера** в конфигурационных файлах ClickHouse.
- **Создание локальных таблиц** на каждом шарде с одинаковой структурой.
- **Создание Distributed таблицы**, которая объединяет локальные таблицы и обеспечивает прозрачный доступ к данным.
- **Вставка и запрос данных** через Distributed таблицу так же, как и с обычной таблицей.

---

**Дополнительные ресурсы:**

- [Документация ClickHouse по Distributed таблицам](https://clickhouse.com/docs/ru/engines/table-engines/special/distributed)
- [Руководство по настройке кластеров в ClickHouse](https://clickhouse.com/docs/ru/operations/cluster-setup)
- [Практики шардирования и репликации](https://clickhouse.com/docs/ru/architecture/capabilities)

---

Надеюсь, это поможет вам начать работать с шардированием в ClickHouse!
