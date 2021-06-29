# Confluent Cloud Replication and MongoDB Atlas integration 
This is the high level architecture:
![GitHub Logo](/images/arqui.png)

# Running the Demo
You'll need an account in CC and include API keys from cluster,SR and ksqlDB, please check variables.md , you'll need also an MongoDB atlas instance. 
Confluent and MongoDB should be in the same region.  

# Demo walkthrough:

Start by editing the variable.md file with your implementing: 

Create topics movimientos y customers.public.clientes y categorizacion using Confluent Cloud UI

Lets load first postgres data
```bash
docker exec -it postgres /bin/bash
psql -U postgres-user customers
CREATE TABLE clientes (id_cliente INT PRIMARY KEY, nombre TEXT, apellido TEXT, direccion TEXT);


INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (1, 'fred', 'smith','liber 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (2, 'jorge', 'perez','diez 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (3, 'diego', 'reyes', ' rambla 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (4, 'mario', 'baracus','barceloneta 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (5, 'roque', 'graseras','tranvia 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (6, 'peter', 'cash','dodisio 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (7, 'douglas', 'costa','sacramentos 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (8, 'richard', 'textex','palotes 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (9, 'santi', 'villa','rana 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (10, 'luis', 'sandro','pirineos 124');
```
Login to ksqlDB on the cloud:
```bash
docker exec -it ksqldb-cli   ksql -u $CC_KSQLDB_API_KEY \
       -p $CC_KSQLDB_API_SECRET \
          $CC_KSQLDB_URL


ksql> CREATE SOURCE CONNECTOR customers_reader WITH ( \
    'connector.class' = 'io.debezium.connector.postgresql.PostgresConnector',\
    'database.hostname' = 'postgres',\
    'database.port' = '5432',\
    'database.user' = 'postgres-user',\
    'database.password' = 'postgres-pw',\
    'database.dbname' = 'customers',\
    'database.server.name' = 'customers',\
    'table.whitelist' = 'public.clientes',\
    'transforms' = 'unwrap',\
    'transforms.unwrap.type' = 'io.debezium.transforms.ExtractNewRecordState',\
    'transforms.unwrap.drop.tombstones' = 'false',\
    'transforms.unwrap.delete.handling.mode' = 'rewrite'\
);
ksql> create stream movimientos( \
    id_movimiento INTEGER, \
    id_cliente INTEGER, \
    id_cuenta INTEGER, \
    monto BIGINT, \
    concepto VARCHAR \
    ) with (\ 
        kafka_topic='movimientos', \
        value_format='avro');
```

ksql> CREATE STREAM clientes WITH (
    kafka_topic = 'customers.public.clientes',
    value_format = 'avro'
);

ksql> CREATE TABLE t_clientes AS
    SELECT id_cliente,
           latest_by_offset(nombre) AS nombre,
           latest_by_offset(apellido) AS apellido,
           latest_by_offset(direccion) AS direccion
    FROM clientes
    GROUP BY id_cliente
    EMIT CHANGES;

ksql>  SET 'auto.offset.reset' = 'earliest';