# Practica Integradora de herramientas de Big Data

Durante esta practica la idea es emular un ambiente de trabajo, desde un área de innovación solicitan construir un MVP(Producto viable mínimo) de un ambiente de Big Data donde se deban cargar unos archivos CSV que anteriormente se utilizaban en un datawarehouse en MySQl, pero ahora en un entorno de Hadoop.

Desde la gerencia de Infraestructura no están muy convencidos de utilizar esta tecnología por lo que no se asigno presupuesto alguna para esta iniciativa, de forma tal que por el momento no es posible utilizar un Vendor(Azure, AWS, Google) para implementar dicho entorno, es por esto que todo el MVP se deberá implementar utilizando Docker de forma tal que se pueda hacer una demo al sector de infraestructura mostrando las ventajas de utilizar tecnologías de Big Data.

# Entorno Docker con Hadoop, Spark y Hive
![](imgs/intro-a.jpeg) ![](imgs/intro-ab.png) ![](imgs/intro-b.png) ![](imgs/intro-c.jpeg)

Se pesenta un entorno Docker con Hadoop (HDFS) y la implementación de:
* Spark
* Hive 
* HBase
* MongoDB
* Neo4J
* Zeppelin
* Kafka


Es importante mencionar que el entorno completo consume muchos recursos de su equipo, motivo por el cuál, se propondrán ejercicios pero con ambientes reducidos, en función de las herramientas utilizadas.

### Nota: Este proyecto fue desarrollado en Windows, por lo que se generó una máquina virtual en VirtualBox con Ubuntu (Docker no es compatible con Windows) en la que se instaló Docker y se utilizó PuTTY para conectar la máquina virtual con la original con Windows. Tambien se puede utilizar WinSCP para facilitar el manejo de archivos en Ubuntu. Los links se proveen a continuacion

>VirtualBox: https://www.virtualbox.org/wiki/Downloads
>PuTTY: https://www.putty.org/
>WinSCP: https://winscp.net/eng/download.php

Ejecute `docker network inspect` en la red (por ejemplo, `docker-hadoop-spark-hive_default`) para encontrar la IP en la que se publican las interfaces de hadoop. Acceda a estas interfaces con las siguientes URL:

```
Namenode: http://<IP_Anfitrion>:9870/dfshealth.html#tab-overview
Datanode: http://<IP_Anfitrion>:9864/
Spark master: http://<IP_Anfitrion>:8080/
Spark worker: http://<IP_Anfitrion>:8081/	
HBase Master-Status: http://<IP_Anfitrion>:16010
HBase Zookeeper_Dump: http://<IP_Anfitrion>:16010/zk.jsp
HBase Region_Server: http://<IP_Anfitrion>:16030
Zeppelin: http://<IP_Anfitrion>:8888
Neo4j: http://<IP_Anfitrion>:7474
```

Para implementar ejecute

	git clone https://github.com/javyleonhart/Practica_herramientasBigData.git

Una vez descargadas las herramientas que utilizaremos, entraremos al carpeta con el comando "cd"

	cd Practica_herramientasBigData

Con el siguiente levantaremos el contenedor correspondiente a la tarea que ejecutaremos

	sudo docker-compose -f docker-compose-vX.yml up -d

Reemplazar la X con el numero deseado

## 1) HDFS

Se puede utilizar el entorno docker-compose-v1.yml

	sudo docker-compose -f docker-compose-v1.yml up -d

Una vez creado el contendor, entraremos al namenode(contenedor) 

	sudo docker exec -it namenode bash
 .
 
	cd home
 
Crearemos el directorio Datasets con el comando mkdir

	mkdir Datasets

Dentro de Datasets

	cd Datasets

 crearemos una carpeta para cada csv que vamos a exportar con mkdir.
 
 	mkdir calendario
	mkdir canaldeventa
	mkdir cliente
	mkdir compra
	mkdir data_nvo
	mkdir empleado
	mkdir gasto
	mkdir producto
	mkdir proveedor
	mkdir sucursal
	mkdir tipodegasto
	mkdir venta
 
 Para esto se provee el archivo Paso00.sh, el cual hay que mover dentro de la carpeta donde se quiere utilizar, en este caso Datasets, con el siguiente comando

 	sudo docker cp <path><archivo> namenode:/home/Datasets/

Luego, salimos

	exit

Y copiaremos los archivos ubicados en la carpeta Datasets, dentro del contenedor "namenode"

	sudo docker cp <path><archivo> namenode:/home/Datasets/<archivo>

Se puede ejecutar el archivo "Paso01" provisto en los materiales para mover todos los archivos. Para usarlo, primero hay que darle permisos con chmod

	chmod u+x Paso01.sh

luego ejecutarlo

	./Paso01.sh


Ubicarse en el contenedor "namenode"


	sudo docker exec -it namenode bash


Crear un directorio en HDFS llamado "/data".


	hdfs dfs -mkdir -p /data


Copiar los archivos csv provistos a HDFS:

	hdfs dfs -put /home/Datasets/* /data


Para verificar si se ejecuto correctamente podemos entrar al hdfs namenoda mediante

	http://<IP_Anfitrion>:9870/dfshealth.html#tab-overview

En utilities/Browse the file system debe estar la carpeta data con los archivos

![](imgs/img1-a.png)
![](imgs/img1-b.png)

### Nota: Se recomienda guardar los csv en su respectiva carpeta y no todos juntos porque al usar Hive, se puede poblar una tabla con todos los csv en determinada carpeta. Por ejemplo, si tengo 2 o mas archivos de 'compra', al crear la tabla, con el comando LOCATION se pueden cargar a la tabla todos los csv presentes en dicha carpeta en vez de uno por uno con el LOAD DATA INPATH.

## 2) Hive

Para este paso se puede utilizar el entorno docker-compose-v2.yml

	sudo docker-compose -f docker-compose-v2.yml up -d

Con el comando anterior creamos un entorno con Hive

Crearemos tablas en Hive, a partir de los csv ingestados en HDFS.

Para esto, se puede ubicar dentro del contenedor correspondiente al servidor de Hive, y ejecutar desdea allí los scripts necesarios

	sudo docker exec -it hive-server bash

Una vez dentro, con el siguiente comando ejecutamos hive

	hive

Aqui dentro podremos crear las tablas, poblarlas y consultarlas con comandos SQL


Este proceso de creación y población las tablas debe poder ejecutarse desde Paso02.hql

Nota: Para ejecutar un script de Hive, requiere el comando:

	hive -f <script.hql>

Este script hay que ejecutarlo desde dentro del entorno de hive, para ello, desde la carpeta donde están los archivos

	sudo docker cp ./Paso02.hql hive-server:/opt/

Para ver si se cargo correctamente, podemos entrar a Hive y consultar la base de datos

![](imgs/img2-a.png)

## 3) Formatos de Almacenamiento

Las tablas creadas en el punto 2 a partir de archivos en formato csv, deben ser almacenadas en formato Parquet + Snappy.
Tener en cuenta además de aplicar particiones para alguna de las tablas. Para ello vamos a crear una segunda DB donde poblaremos las tablas a partir de las tablas de la primera DB. Primero agregamos unos comandos al CREATE TABLE

>STORED AS PARQUET - Con esta linea guardamos la informacion que cargamos en la tabla como .Parquet
>TBLPROPERTIES ('parquet.compression'='SNAPPY') - Con esta linea comprimimos el .parquet en formato Snappy

Al poblar las tablas con INSERT INTO, para que tome la informacion de la primera DB escribiremos, luego de los campos:

>FROM 'db1'.'tabla'

Si queremos particionar la tabla, al crearla agregamos

> PARTITIONED BY('columna' 'tipo de dato')

Si queremos insertar la data particionada, en el INSERT agregamos

> PARTITION('IdTipoGasto'<- columna '=1' <- condicion))

Se proporciona en los materiales el script Paso03.hql que corre dicho ejercicio y hace unos ejercicios de particion. Recordar de pasar el archivo dentro del contenedor de Hive para poder ejecutarlo

Si consultamos por la tabla gasto, verificamos que la tabla esta organizada por el IdTipoGasto que tomo de la primera DB

![](imgs/img3-a.png)

## 4) SQL

La mejora en la velocidad de consulta que puede proporcionar un índice tiene el costo del procesamiento adicional para crear el índice y el espacio en disco para almacenar las referencias del índice.
Se recomienda que los índices se basen en las columnas que utiliza en las condiciones de filtrado. El índice en la tabla puede degradar su rendimiento en caso de que no los esté utilizando.
Crear índices en alguna de las tablas cargadas y probar los resultados:

```
CREATE INDEX index_name
 ON TABLE base_table_name (col_name, ...)
 AS index_type
 [WITH DEFERRED REBUILD]
 [IDXPROPERTIES (property_name=property_value, ...)]
 [IN TABLE index_table_name]
 [ [ ROW FORMAT ...] STORED AS ...
 | STORED BY ... ]
 [LOCATION hdfs_path]
 [TBLPROPERTIES (...)]
 [COMMENT "index comment"];
```

Ejemplo:

```
hive> CREATE INDEX index_students ON TABLE students(id) 
 > AS 'org.apache.hadoop.hive.ql.index.compact.CompactIndexHandler' 
 > WITH DEFERRED REBUILD ;
```
Para cambiar el índice, podemos usar el siguiente comando 

>ALTER INDEX index_name ON table_name [PARTITION partition_spec] REBUILD;

Ejemplo:
```
hive> ALTER INDEX index_students ON students(idCohort) REBUILD; 
```
Para eliminar el índice

>DROP INDEX [IF EXISTS] index_name ON table_name;
```
hive> DROP INDEX IF EXISTS index_students ON students; 
```

A modo de ejemplo, trabajaremos en la BD integrador2 y realizaremos una consulta:

Primero entramos a la BD

	USE integrador2;

Luego realizamo la siguiente consulta y revisaremos el tiempo

	SELECT idsucursal, SUM(precio * cantidad) FROM venta GROUP BY idsucursal;

![](imgs/img4-a.png)

Luego crearemos un índice con el siguien comando

	CREATE INDEX index_venta_sucursal ON TABLE venta(IdSucursal) AS 'org.apache.hadoop.hive.ql.index.compact.CompactIndexHandler' WITH DEFERRED REBUILD;

Y realizaremos la misma consulta para ver cuanto tiempo tarda

![](imgs/img4-b.png)

Como podemos observar, la misma consulta redujo considerablemente el tiempo de ejecución luego de la creación del índice

![](imgs/img4-c.png)

