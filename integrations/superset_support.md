# 支持Superset

[Apache Superset](https://superset.apache.org) 是一个现代数据探索和可视化平台。它使用[SQLAlchemy](https://docs.sqlalchemy.org/en/13/index.html)来查询数据。

虽然可以使用[Mysql Dialect](https://superset.apache.org/docs/databases/mysql)，但是它不支持`largeint`。所以我们开发了[StarRocks Dialect](https://github.com/StarRocks/starrocks/blob/main/contrib/sqlalchemy-connector)。



## 环境准备

- Python 3.x
- mysqlclient (pip install mysqlclient)
- [Apache Superset](https://superset.apache.org)


注意: 如果没有安装`mysqlclient`, 将抛出异常:
```
No module named 'MySQLdb'
```

## 安装

由于`dialect`还没有贡献给`SQLAlchemy`社区，需要使用源码进行安装。

如果你使用Docker安装`superset`，需要使用`root`用户安装`sqlalchemy-starrocks`。

安装通过[源码](https://github.com/StarRocks/starrocks/blob/main/contrib/sqlalchemy-connector)
```
pip install .
```
卸载
```
pip uninstall sqlalchemy-starrocks
```
## 使用

通过SQLAlchemy连接StarRocks，可以使用下述链接：

```
starrocks://<username>:<password>@<host>:<port>/<database>[?charset=utf8]
```

## 案例
### Sqlalchemy案例
建议使用python 3.x链接StarRocks数据库，比如：
```
from sqlalchemy import create_engine
import pandas as pd
conn = create_engine('starrocks://root:@x.x.x.x:9030/superset_db?charset=utf8')
sql = """select * from xxx"""
df = pd.read_sql(sql, conn)
```

### Superset Example
使用superset时，使用`Other`数据库连接，并且设置url为：
```
starrocks://root:@x.x.x.x:9030/superset_db?charset=utf8
```

