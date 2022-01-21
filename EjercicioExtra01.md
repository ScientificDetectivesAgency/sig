# Ejercicio extra

Todos los datos de la práctica pueden ser descargados del siguiente [link](https://drive.google.com/drive/folders/1hzKPipsvvtnzqzkHqaQQ9VLM45J4P6X7?usp=sharing)

En esta práctica vamos a comenzar a trabajar con dos tipos de datos, los espciales y los no espaciales para obtener información reelevante de una zona en específico. En este caso una parte de la Zona Metropolitana del Valle de México y el censo de población. 


``` sql
create database extras;
create extension postgis;
create schema ej01;
```

# PREPARANDO LOS DATOS

Primero vamos a explorar las tablas de datos y lo que contienen y seleccionaremos el área de interés, verificaremos que los identificadores cumplan con las condiciones necesarias, determinaremos las claves primerias, crearemos los indices necesarios y finalmente definiremos la proyección idonea para trabajar con estos datos. 
 
``` sql
select * from practica1.pob_manzanas_edomex_cdmx limit 10
```

En la carpeta de datos se encuentra el diccionario de datos donde vienen el significado de cada una de las columnas, para este ejercicio solo trabajaremos con el total de población por manzana por lo que para optimizar los tiempos de consulta solo trabajaremos con las columnas: id, manzana_cvegeo, pob01  

``` sql
select id, manzana_cvegeo, pob01 from practica1.pob_manzanas_edomex_cdmx
```
Habiendo creado la consulta con la que seleccionaremos las columnas de interés ahora es necesario unirla con la tabla de geometrías para poder filtar los datos a través de una intersección espacial. Pero, ¿Qué columna nos va a ayudar a realizar esta tarea? para ellos como en el primer ejercicio vamos a explorar los datos espaciales

``` sql
select * from practica1.manzanas_edomex_cdmx limit 10
``` 
Seguro notaron que las columnas no coinciden con las de la tabla anterior, por lo que las vamos a renombrar

``` sql
alter table practica1.manzanas_edomex_cdmx rename column municipio_ to municipio_cvegeo;
alter table practica1.manzanas_edomex_cdmx rename column manzana_cv to manzana_cvegeo;
alter table practica1.manzanas_edomex_cdmx rename column manzana_no to manzana_nombre;
alter table practica1.manzanas_edomex_cdmx rename column entidad_cv to entidad_cvegeo;
``` 

Ambas tablas comparten dos columnas la de id y la de manzana_cvegeo, un error común es utilizar siempre la columna de id para hacer las uniones entre tablas. en este caso esta columna no es la ideal, para esta tarea ya que no sabes si efectivamente cada elemento de los datos de cada tabla tiene una correspondencia entre si. Sin embargo, la columna manzana_cvegeo si la tiene y es por ello que será la columna que usaremos para realizar la union de ambas tablas a través de un inner join. 


``` sql
select a.geom, b.*
from practica1.manzanas_edomex_cdmx a
join practica1.pob_manzanas_edomex_cdmx b
on a.manzana_cvegeo = b.manzana_cvegeo
``` 

Ahora que tenermos la union de ambas tablas, para no tener que crear una nueva la filtraremos, intersecctandola con el límite metropolitano para quedarnos solo con las manzanas que se encuentran dentro de el haicendo usa de una SUBCONSULTA.

``` sql
select c.*
from (select a.geom, b.* from practica1.manzanas_edomex_cdmx a join practica1.pob_manzanas_edomex_cdmx b on a.manzana_cvegeo = b.manzana_cvegeo) as c
join practica1.limite_metropolitano d
on st_intersects (c.geom, d.geom)
```

Notaran que al hacer la consulta no regresa ningún dato, esto es porque al hacer el filtrado por una intersección es necesario que ambas capas estén en la misma proyección. Y ya que vamos a hacer mediciones sobre las capas en distancias reproyectaremos la que sea necesaria a metros

``` sql
 ALTER TABLE practica1.manzanas_edomex_cdmx
   ALTER COLUMN geom
   TYPE Geometry(MultiPolygon, 32614)
   USING ST_Transform(geom, 32614);
``` 

Ahora al volver a correr la consulta anterior si nos arroja datos y entonces podemos crear la tabla final de manzanas_zmvm 

``` sql
create table practica1.manzanas_zmvm as 
select c.*
from (select a.geom, b.* from practica1.manzanas_edomex_cdmx a join practica1.pob_manzanas_edomex_cdmx b on a.manzana_cvegeo = b.manzana_cvegeo) as c
join practica1.limite_metropolitano d
on st_intersects (c.geom, d.geom);
```

Verificamos que nuestra clave no este duplicada

``` sql
select count(*), manzana_cvegeo  from manzanas_zmvm  group by manzana_cvegeo order by count(*) desc;
```

Para eliminar los duplicados corremos la siguiente consulta y creamos otra tabla

``` sql
create table practica1.manzanas_zmvm_sd as
select DISTINCT ON (manzana_cvegeo) * from practica1.manzanas_zmvm
```

Eliminamos la tabla anterior y renombramos la nueva 

Ahora creamos un índice espacial sobre la geometría
-- TAREA: investiga qué son y para qué sirven los índices espaciales y de qué tipos hay

``` sql
create index manzanas_zmvm_gix on practica1.manzanas_zmvm using GIST(geom);
```

Creamos los constraints necesarios y agregamos un PK

``` sql
ALTER TABLE practica1.manzanas_zmvm ALTER COLUMN "manzana_cvegeo" SET NOT NULL;
ALTER TABLE practica1.manzanas_zmvm ADD UNIQUE ("manzana_cvegeo");
ALTER TABLE practica1.manzanas_zmvm ADD PRIMARY KEY ("manzana_cvegeo");
```

Ya podemos comenzar a hacerle preguntas a la base de datos
¿cuantas manzanas quedan a 500 metros de cada estación del metro?

NOTA: Otro modo de hacer subconsultas es usando un with

``` sql
select foo.* from
(with 
  buf as (select st_buffer(practica1.estaciones_metro.geom,500.0) as geom , 
  				 practica1.estaciones_metro.nombreesta as estacion 
  				 from practica1.estaciones_metro)
select count(manzanas_zmvm.id), buf.estacion 
	from manzanas_zmvm 
	join buf 
	on st_intersects(buf.geom,manzanas_zmvm.geom)
	group by buf.estacion) as foo;
```
Tambien podemos preguntar ¿Cuánta gente vive a 500 m de una estación del metro?
NOTA: La columna pob01 contiene la población de cada manzana

``` sql
select foo.* from
(with 
  buf as (select st_buffer(practica1.estaciones_metro.geom,500.0) as geom , 
  				 practica1.estaciones_metro.nombreesta as estacion 
  				 from practica1.estaciones_metro)
select sum(a.pob01::int), buf.estacion 
	from practica1.manzanas_zmvm a
	join buf 
	on st_intersects(buf.geom,a.geom)
	group by buf.estacion) as foo;
  ```
