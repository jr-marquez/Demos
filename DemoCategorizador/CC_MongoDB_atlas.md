# Confluent Cloud Replication and MongoDB Atlas integration 
This is the high level architecture:
![GitHub Logo](https://github.com/jr-marquez/Demos/blob/main/DemoCategorizador/images/arqui.png)

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


INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (1, 'Fred', 'Smith','liber 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (2, 'Jorge', 'Perez','diez 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (3, 'Diego', 'Reyes', ' rambla 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (4, 'Mario', 'Baracus','barceloneta 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (5, 'Roque', 'Graseras','tranvia 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (6, 'Peter', 'Cash','dodisio 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (7, 'Douglas', 'Costa','sacramentos 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (8, 'Richard', 'Textex','palotes 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (9, 'Santi', 'Villa','rana 124');
INSERT INTO clientes (id_cliente, nombre, apellido, direccion) VALUES (10, 'Luis', 'Sandro','pirineos 124');
```
Login to ksqlDB on the cloud:
```bash
docker exec -it ksqldb-cli   ksql -u $CC_KSQLDB_API_KEY \
       -p $CC_KSQLDB_API_SECRET \
          $CC_KSQLDB_URL

ESTa MAL, es por postman
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
ksql> create stream movimientos( 
    id_movimiento INTEGER,
    id_cliente INTEGER,
    id_cuenta INTEGER,
    monto BIGINT,
    concepto VARCHAR,
    banco VARCHAR
    ) with (
        kafka_topic='movimientos', 
        value_format='json');
```
ksql> CREATE STREAM clientes WITH (
    kafka_topic = 'customers.public.clientes',
    value_format = 'avro'
);


create stream clientes_json with (  kafka_topic = 'clientes_json',
    value_format = 'json') as select * from clientes ;

ksql> CREATE TABLE t_clientes AS
    SELECT id_cliente,
           latest_by_offset(nombre) AS nombre,
           latest_by_offset(apellido) AS apellido,
           latest_by_offset(direccion) AS direccion
    FROM clientes
    GROUP BY id_cliente
    EMIT CHANGES;


ksql>  SET 'auto.offset.reset' = 'earliest';


create stream movimiento_ampliado_origen  with (KAFKA_TOPIC='movimiento_ampliado')
	as select 
		m.id_movimiento id_movimiento,
        m.id_cliente id_cliente1,
		AS_VALUE(m.id_cliente) id_cliente,
		m.id_cuenta id_cuenta,
		m.monto monto,
		m.concepto concepto,
		m.banco banco, 
		c.nombre nombre,
		c.apellido apellido,
		c.direccion direccion
	from movimientos m
	inner join t_clientes c on m.id_cliente  = c.id_cliente;



    create stream movimiento_ampliado_origen (
		id_movimiento INTEGER,
        id_cliente INTEGER,
        id_cuenta INTEGER,
        monto BIGINT,
        concepto VARCHAR,
        banco VARCHAR,
		nombre VARCHAR,
		apellido VARCHAR,
		direccion VARCHAR )
    with (KAFKA_TOPIC='movimiento_ampliado', VALUE_FORMAT='JSON') ;

CREATE STREAM categorizacion
    (concepto varchar key, 
    categoria varchar
    )
    WITH (kafka_topic='categorizacion', value_format='avro', partitions=6);

INSERT INTO categorizacion (concepto, categoria) VALUES ('Compra de entradas ticketmaster concierto ricky martin','ocio');
INSERT INTO categorizacion (concepto, categoria) VALUES ('compra en Restaurante Coca nro4','alimentacion');
INSERT INTO categorizacion (concepto, categoria) VALUES ('concepto delivery supermercados dya','alimentacion');
INSERT INTO categorizacion (concepto, categoria) VALUES ('pizzería muy buena hermanos','alimentacion');
INSERT INTO categorizacion (concepto, categoria) VALUES ('Cuenta social club deportivo torque','ocio');
INSERT INTO categorizacion (concepto, categoria) VALUES ('Recibo de agua y luz - Todo junto','Hogar');
INSERT INTO categorizacion (concepto, categoria) VALUES ('compra calzado para niños - Me Queda Chica S.A','Ropa y Complementos');
INSERT INTO categorizacion (concepto, categoria) VALUES ('Kiosko Bendita Golosina S.A','alimentacion');

create table t_categorizacion as
    SELECT concepto,
           latest_by_offset(categoria) AS categoria
    FROM categorizacion
    GROUP BY concepto
    EMIT CHANGES;

create stream movimiento_ampliado_final with (kafka_topic='movimiento_ampliado_final')
	as select 
		m.id_cliente `id_cliente`,
        m.id_cuenta `id_cuenta`,
        m.id_movimiento  `id_movimiento`,
		m.monto `monto`,
        m.concepto concepto1,
		AS_Value(m.concepto) `concepto`,
        cat.categoria `categoria`,
        m.banco `banco`, 
        m.nombre `nombre`,
		m.apellido `apellido`,
		m.direccion `direccion`
	from movimiento_ampliado_origen m
	left join t_categorizacion cat on m.concepto  = cat.concepto
	partition by m.concepto;