# Инструкция по развертыванию Apache Hive

## Дано
- Развернутый на 3 нодах кластер HDFS
- Все ноды проименованы (tmpl-nn, tmpl-dn-00, tmpl-dn-00)

Подключаемся к неймноде и переходим на пользователя с правами suders.

Итак, устанавливаем СУБД postgresql.
```
sudo apt install postgresql
```

Переключаемся на пользователя postgres, создаем и настраиваем базу данных.
```
sudo -i -u postgres
psql
CREATE DATABASE metastore;
CREATE USER hive WITH PASSWORD 'hiveMegaPass';
GRANT ALL PRIVILEGES ON DATABASE "metastore" TO hive;
ALTER DATABASE metastore OWNER TO hive;
\q
```

Открываем конфиг postgres и указываем, что нужно слушать адрес нашей неймноды.
```
sudo nano /etc/postgresql/16/main/postgresql.conf
```

```
# добавляем в файл следующую строку
listen_addresses = tmpl-nn
```

Далее открываем конфиг для настроек безопасности:
```
sudo nano /etc/postgresql/16/main/pg_hba.conf
```
И вписываем туда следующие строки:
```
# в секции IPv4 local connections
host metastore hive 192.168.1.1/32 password
host metastore hive <ip-адрес вашей jump-node>/32 password
```

Перезапускаем postgres:

```
sudo systemctl restart postgresql
```

И подключаемся к jump-node. Теперь нам нужно установить клиента postgresql.
```
sudo apt install posgtresql-client-16
```

Переключаемся на пользователя hadoop и выполняем следующие команды:
```
# скачиваем hive
wget https://archive.apache.org/dist/hive/hive-4.0.0-alpha-2/apache-hive-4.0.0-alpha-2-bin.tar.gz
# разархивируем
tar -xzvf apache-hive-4.0.0-alpha-2-bin.tar.gz

# скачиваем драйвер hive
cd apache-hive-4.0.0-alpha-2-bin/lib
wget https://jdbc.postgresql.org/download/postgresql-42.7.4.jar
```

Теперь необходимо отредактировать конфиги:
```
nano ../conf/hive-site.xml
```
В этот файл записываем следующее:
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
		<value>5433</value>
		<description>TCP port number to listen on, default 10000</description>
	</property>
	<property>
		<name>javax.jdo.option.ConnectionURL</name>
		<value>jdbc:postgresql://${nn}:5432/metastore</value>
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
		<value>${hive-password}</value>
	</property>
</configuration>
```

Добавим переменные окружения.
```
nano ~/.profile
```
```
# вписываем это в файл
export HIVE_HOME=/home/hadoop/apache-hive-4.0.0-alpha-2-bin
export HIVE_CONF_DIR=$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/*
export PATH=$PATH:$HIVE_HOME/bin
```

Применяем изменения.
```
source ~/.profile
```

Создаем в dfs нужные директории и зададим для них права.
```
hdfs dfs -mkdir -p /user/hive/warehouse
hdfs dfs -mkdir -p /tmp
hdfs dfs -chmod g+w /user/hive/warehouse
hdfs dfs -chmod g+w /tmp
```

Инициализируем DWH.
```
cd ..
bin/schematool -dbType postgres -initSchema
```

Запускаем hive.
```
hive --hiveconf hive.server2.enable.doAs=false --hiveconf hive.security.authorization.enable=false --service hiveserver2 1>> /tmp/hs2.log 2>> /tmp/hs2.log &
```

Подключаемся.
```
beeline -u jdbc:hive2://tmpl-jn:5433 -n scott -p tiger
```


__Автор:__ Сангаджиев Эренцен


