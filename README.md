# HiveHW3


Задача:
Описать или реализовать автоматизированное развертывание Apache Hive таким образом, чтобы была возможность одновременного его использования более чем одним клиентом (т.е. при установке не использовать embedded подход), описать или реализовать автоматизированную загрузку данных в партиционированную таблицу Hive. Рекомендуется использовать версию дистрибутива Hive 4.0.0. alpha2

Считается, что уже выполнены инструкции, данные в ДЗ1 (https://github.com/VarakinMikhail/HadoopHW1) и ДЗ2 (https://github.com/VarakinMikhail/HadoopHW2)

Данные:

узел для входа 176.109.81.242 

jn 192.168.1.106 

nn 192.168.1.107 

dn-00 192.168.1.109 

dn-01 192.168.1.108

Подключимся к ноде:
```
ssh team@176.109.81.242
```

И перейдем на NameNode:
```
ssh tmpl-nn
```
Установим postgres:
```
sudo apt install postgresql
```

Переключимся на пользователя postgres и создадим базу данных, которая будет использоваться metastore:
```
sudo -i -u postgres
```
```
psql
```
```
CREATE DATABASE metastore;
```
```
CREATE USER hive with password 'hiveMegaPass';
```
Даем все привилегии пользователю hive:
```
GRANT ALL PRIVILEGES ON DATABASE "metastore" to hive;
```

И передаем владение базой новому пользователю hive:
```
ALTER DATABASE metastore OWNER TO hive;
```

Теперь можно выйти из SQL:
```
\q
```

И вернуться обратно к пользователю team:
```
exit
```
И теперь нужно отредактировать конфиг:
```
sudo vim /etc/postgresql/16/main/postgresql.conf
```

Там указываем listen_addresses = '*' и указываем порт 5433

И конфиг, связанный с безопасностью:
```
sudo vim /etc/postgresql/16/main/pg_hba.conf
```

В нем добавляем строки
```
host	metastore	hive	192.168.1.107/32	password
host	metastore	hive	192.168.1.106/32	password
```

Для NameNode и JumpNode соответственно

Перезагружаем postgresql:
```
sudo systemctl restart postgresql
```

Переходим на JumpNode пользователя team, и устанавливаем клиент postgres:
```
sudo apt install postgresql-client-16
```

Переключаемся на пользователя hadoop:
```
sudo -i -u hadoop
```

И скачиваем дистрибутив hive:
```
wget https://archive.apache.org/dist/hive/hive-4.0.0-alpha-2/apache-hive-4.0.0-alpha-2-bin.tar.gz
```

Распакуем его:
```
tar -xzvf apache-hive-4.0.0-alpha-2-bin.tar.gz
```

Нужно скачать драйвер, чтобы подключаться к postgres:
Для этого перейдем в директорию:
```
cd apache-hive-4.0.0-alpha-2-bin/lib/
```

И загрузим драйвер:
```
wget https://jdbc.postgresql.org/download/postgresql-42.7.4.jar
```

Теперь отредактируем конфиги:
```
vim ../conf/hive-site.xml
```
Запишем туда следующее:
```
<configuration>
    <property>
        <name>hive.server2.authentication</name>
        <value>NONE</value>
    </property>
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
    <property>
        <name>hive.server2.thrift.port</name>
        <value>5432</value>
        <description>TCP port number to listen on, default 10000</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:postgresql://tmpl-nn:5433/metastore</value>
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
        <value>hiveMegaPass</value>
    </property>
</configuration>
```

Теперь добавим переменные окружения в профиль:
```
vim ~/.profile
```
Вставим туда следующие строки:
```
export HIVE_HOME=/home/hadoop/apache-hive-4.0.0-alpha-2-bin
export HIVE_CONF_DIR=$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/*
export PATH=$PATH:$HIVE_HOME/bin
```

Применим переменные окружения:
```
source ~/.profile
```

Создадим папку для DWH (на которую ссылались в конфиге):
```
hdfs dfs -mkdir -p /user/hive/warehouse
```

Выдаем права для папок:
```
hdfs dfs -chmod g+w /tmp
```
```
hdfs dfs -chmod g+w /user/hive/warehouse
```

Возвращаемся на папку назад:
```
cd ..
```

Инициализируем БД:
```
bin/schematool -dbType postgres -initSchema
```

Теперь можно запускать сервис:
```
nohup hive --service hiveserver2 \
--hiveconf hive.server2.enable.doAs=false \
--hiveconf hive.security.authorization.enabled=false \
>> /tmp/hs2.log 2>&1 &
```
И подключиться к нему:
```
beeline -u jdbc:hive2://tmpl-jn:5432 -n scott -p tiger
```
