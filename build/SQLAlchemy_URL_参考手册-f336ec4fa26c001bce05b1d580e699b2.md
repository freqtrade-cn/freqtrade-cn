# SQLAlchemy 数据库连接 URL 参考手册

## 📋 基本格式

```
dialect[+driver]://user:password@host:port/database[?key=value&key=value...]
```

### 格式说明
- **dialect**: 数据库类型 (`mysql`, `postgresql`, `sqlite`, `oracle`, `mssql` 等)
- **driver**: 数据库驱动 (可选)
- **user**: 用户名
- **password**: 密码
- **host**: 主机地址
- **port**: 端口号
- **database**: 数据库名
- **key=value**: 额外参数

---

## 🗃️ SQLite 数据库

| 用途 | URL 示例 |
|------|----------|
| **内存数据库** | `sqlite:///:memory:` |
| **相对路径** | `sqlite:///example.db` |
| **绝对路径** | `sqlite:////absolute/path/to/database.db` |
| **Windows路径** | `sqlite:///C:\path\to\database.db` |
| **带参数** | `sqlite:///example.db?check_same_thread=false` |

---

## 🐬 MySQL 数据库

| 驱动 | URL 示例 |
|------|----------|
| **PyMySQL (推荐)** | `mysql+pymysql://user:password@localhost:3306/database` |
| **MySQLdb** | `mysql+mysqldb://user:password@localhost:3306/database` |
| **MySQL Connector** | `mysql+mysqlconnector://user:password@localhost:3306/database` |

### 常用配置参数

| 功能 | URL 示例 |
|------|----------|
| **UTF-8字符集** | `mysql+pymysql://user:password@localhost:3306/database?charset=utf8mb4` |
| **SSL连接** | `mysql+pymysql://user:password@localhost:3306/database?ssl_ca=/path/to/ca.pem` |
| **AWS RDS** | `mysql+pymysql://user:password@hostname.region.rds.amazonaws.com:3306/database` |
| **连接池** | `mysql+pymysql://user:password@localhost:3306/database?pool_size=20&max_overflow=0` |

---

## 🐘 PostgreSQL 数据库

| 驱动 | URL 示例 |
|------|----------|
| **psycopg2 (默认)** | `postgresql://user:password@localhost:5432/database` |
| **asyncpg (异步)** | `postgresql+asyncpg://user:password@localhost:5432/database` |
| **pg8000** | `postgresql+pg8000://user:password@localhost:5432/database` |

### 常用配置参数

| 功能 | URL 示例 |
|------|----------|
| **指定Schema** | `postgresql://user:password@localhost:5432/database?options=-csearch_path%3Dschema_name` |
| **SSL连接** | `postgresql://user:password@localhost:5432/database?sslmode=require` |
| **AWS RDS** | `postgresql://user:password@hostname.region.rds.amazonaws.com:5432/database` |
| **连接池** | `postgresql://user:password@localhost:5432/database?pool_size=20&max_overflow=0` |

---

## 🔶 Microsoft SQL Server

| 驱动 | URL 示例 |
|------|----------|
| **pyodbc (推荐)** | `mssql+pyodbc://user:password@server:1433/database?driver=ODBC+Driver+17+for+SQL+Server` |
| **pymssql** | `mssql+pymssql://user:password@server:1433/database` |

### 特殊配置

| 功能 | URL 示例 |
|------|----------|
| **Windows认证** | `mssql+pyodbc://server/database?driver=ODBC+Driver+17+for+SQL+Server&trusted_connection=yes` |
| **Azure SQL** | `mssql+pyodbc://user:password@server.database.windows.net:1433/database?driver=ODBC+Driver+17+for+SQL+Server&encrypt=yes` |

---

## 🏛️ Oracle 数据库

| 连接方式 | URL 示例 |
|----------|----------|
| **基本连接** | `oracle+cx_oracle://user:password@localhost:1521/database` |
| **TNS连接** | `oracle+cx_oracle://user:password@localhost:1521/?service_name=service` |
| **SID连接** | `oracle+cx_oracle://user:password@localhost:1521/?sid=ORCL` |

---

## 📮 Redis 数据库

| 配置 | URL 示例 |
|------|----------|
| **本地Redis** | `redis://localhost:6379/0` |
| **带密码** | `redis://:password@localhost:6379/0` |
| **用户名密码** | `redis://user:password@localhost:6379/0` |
| **SSL连接** | `rediss://user:password@localhost:6380/0` |
| **集群** | `redis://localhost:7000,localhost:7001,localhost:7002/0` |

---

## 🍃 MongoDB 数据库

| 配置 | URL 示例 |
|------|----------|
| **本地连接** | `mongodb://localhost:27017/database` |
| **带认证** | `mongodb://user:password@localhost:27017/database` |
| **副本集** | `mongodb://user:password@host1:27017,host2:27017,host3:27017/database?replicaSet=myReplicaSet` |
| **Atlas云** | `mongodb+srv://user:password@cluster.mongodb.net/database?retryWrites=true&w=majority` |

---

## 🔧 其他数据库

| 数据库 | URL 示例 |
|--------|----------|
| **ClickHouse** | `clickhouse://user:password@localhost:9000/database` |
| **CockroachDB** | `cockroachdb://user:password@localhost:26257/database?sslmode=require` |
| **Snowflake** | `snowflake://user:password@account.region.snowflakecomputing.com/database/schema?warehouse=warehouse&role=role` |
| **BigQuery** | `bigquery://project/dataset` |

---

## 💡 实用技巧

### 1. 环境变量配置
```python
import os
DATABASE_URL = os.getenv('DATABASE_URL', 'sqlite:///default.db')
```

### 2. 配置文件管理
```python
DATABASE_CONFIG = {
    'development': 'sqlite:///dev.db',
    'testing': 'sqlite:///:memory:',
    'production': 'postgresql://user:password@prod-server:5432/app'
}
```

### 3. 连接池优化
```python
# MySQL
'mysql+pymysql://user:password@localhost:3306/db?pool_size=20&pool_recycle=3600'

# PostgreSQL  
'postgresql://user:password@localhost:5432/db?pool_size=20&max_overflow=0'
```

### 4. SSL/TLS 配置
```python
# MySQL SSL
'mysql+pymysql://user:password@host:3306/db?ssl_ca=/path/to/ca.pem&ssl_verify_cert=true'

# PostgreSQL SSL
'postgresql://user:password@host:5432/db?sslmode=require&sslcert=/path/to/cert.pem'
```

### 5. 特殊字符处理
```python
from urllib.parse import quote_plus

password = "p@ssw0rd!"
encoded_password = quote_plus(password)
url = f"mysql+pymysql://user:{encoded_password}@localhost:3306/database"
```

---

## 📦 必需的 Python 包

| 数据库 | 驱动包 | 安装命令 |
|--------|---------|----------|
| **MySQL** | PyMySQL | `pip install PyMySQL` |
| **PostgreSQL** | psycopg2 | `pip install psycopg2-binary` |
| **SQL Server** | pyodbc | `pip install pyodbc` |
| **Oracle** | cx_Oracle | `pip install cx_Oracle` |
| **SQLite** | 内置 | 无需额外安装 |

---

## 🚀 快速开始示例

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# 创建引擎
engine = create_engine('postgresql://user:password@localhost:5432/mydb')

# 创建会话
Session = sessionmaker(bind=engine)
session = Session()

# 使用会话
# ... 你的数据库操作代码
```

---

**💡 提示**: 在生产环境中，建议将数据库连接信息存储在环境变量或配置文件中，而不是硬编码在代码中。
