# Clase de introducción al uso de consultas espaciales en PosrgreSQL-PostGIS

El objetivo de esta práctica es entender la forma en que se deben realizar las consultas espaciales y no espaciales haciendo uso del lenguaje SQL (Structured Query Language). Todos los datos de la práctica pueden ser descargados del siguiente [link]()

En esta práctica vamos a explorar datos relacionados con las llamadas registradas por el servicio de 911, las áreas verdes y datos de población de la Ciudad de México. La CDMX es la ciudad más caoticas del páis, dode es posible encontrar un gran número de áreas verdes. Sin embargo, no todas pueden ser aprovechadas por la ciudadacía, gracias a los altos índices de delincuencia que se pueden identificar en sus alrededores. En este esjercicio nos centraremos a aborDar el terma de la seguridad ciudadana y el aprovechamiento de las áreas verdes disponibles.  

Para comenzar debes crear una base de datos que se llame practicas y un esquema que se llame practica1: 

``` sql
create database practicas;
create extension postgis;
create schema practica01;
```
¿Porqué usar esquemas? 

Permiten organizar los objetos de la base de datos, por ejemplo, las tablas en grupos lógicos para hacerlos más manejables.
Hacen posible que varios usuarios utilicen una base de datos sin interferir entre sí.

# PREPARANDO LOS DATOS

Para poder comenzar a trabajar con los datos, es necesario explorarlos e identificar en qué condiciones se encuentran para poder procesarlos y extraer información de ellos. Cuando se trabaja con datos tanto espaciales como no espaciales es necesario que los datos cumplan con tres condiciones importantes: contar como llaves primarias y secundarias, tener indices espaciales y definir la proyección espacial idonea para trabajar con ellos.


(1) Comenzaremos a trabajar con las tablas de manzanas_cdmx y datos_manzadas. Los datos de población y los poligonos de las manzanas se encuentran separados, es por eso que vamos a realizar una unión.  

``` sql
select * from practica01.manzanas_cdmx mc limit 10
select * from practica01.datos_manzanas mc limit 10
```

Ambas tablas comparten dos columnas la de id y la de manzana_cvegeo, un error común es utilizar siempre la columna de id para hacer las uniones entre tablas. En este caso esta columna no es la ideal, ya que no sabemos si efectivamente cada elemento dde cada tabla tiene una correspondencia entre si. Sin embargo, la columna manzana_cvegeo si la tiene y es por ello que será la columna que usaremos para realizar la union. Pero primero necesitamos homogeneizar los nombres de las columnas ya que la tabla con geometrías presnetan errores.


``` sql
alter table practica01.manzanas_cdmx rename column municipio_ to municipio_cvegeo;
alter table practica01.manzanas_cdmx rename column manzana_cv to manzana_cvegeo;
alter table practica01.manzanas_cdmx rename column entidad_cv to entidad_cvegeo;
``` 
Repite lo anterior para la tabla de municipio_cdmx 

Cada tabla debe tener una llave primaria y de ser el caso un índice espacial 

-- TAREA: investiga qué son y para qué sirven los índices espaciales y de qué tipos hay

Primero crearemos los índices espaciales de todas las tablas con geometría en la base de datos. Toma de referencia la siguiente sentencia: 

``` sql
create index nombre_indice_gix on esquema.tabla1 using GIST(columna_geometría);
```

``` sql
create index manzanas_cdmx_gix on practica01.manzanas_cdmx using GIST(geom);
```
Creamos los constraints necesarios para agregar una llave primaria PK.

``` sql
ALTER TABLE practica01.manzanas_cdmx ALTER COLUMN "manzana_cvegeo" SET NOT NULL;
ALTER TABLE practica01.manzanas_cdmx ADD UNIQUE ("manzana_cvegeo");
ALTER TABLE practica01.manzanas_cdmx ADD PRIMARY KEY ("manzana_cvegeo");

NOTA: Para que una columna sea PK, debe tener relación con otras tablas y no aceptar valores nulos ni repetidos. Si son tablas que no se unirán con otras no es necesario tener un PK, hasta contar con una relación en la BD. En caso de tener valores repetidos puedes recurrir a las siguientes consultas:  

``` sql
select count(*), columna_identificador  from esquema.tabla  group by columna_identificador order by count(*) desc;
```
Para eliminar los duplicados corremos la siguiente consulta y creamos otra tabla
``` sql
create table practica1.manzanas_zmvm_sd as
select DISTINCT ON (columna_identificador) * from esquema.tabla
```
# PODEMOS COMENZAR A HACER UNIONES

Las funciones JOIN unen las tablas ya sea a través de una clave unica o con ayuda de funciones espaciales. Estas uniones nos permiten seleecionar los datos que utilizaremos de cada tabla a través de la siguiente sintaxis general. 

``` sql
select tabla1.columna, tabla2.* 
from tabla1 a
join tabla2 b
on a.id_llave = b.id_llave 
``` 

NOTA: Cuanto usamos el ".*" despues del nombre de la tabla, le estamos diciendo que queremos todas las columnas. Las letras a y b despues de los tablas de union 
son apodos que nos permitirán no tener que escribir los nombres completos de las tablas. 

Con la siguiente consulta, traeremos de la tabla de manzanas_cdmx la columna de geometría y de la tabla de datos_manzanas todas las columnas. Es importante notar
que esta no es una union espacial, ya que la estaremos haciendo a través de las claves cvegeo. 

``` sql
select mc.geom, dm.* 
from practica01.manzanas_cdmx mc 
join practica01.datos_manzanas dm 
on mc.manzana_cvegeo = dm.manzana_cvegeo
``` 



SELECT AddGeometryColumn ('practica01','llamadas911','geom',4326,'POINT',2);
UPDATE practica01.llamadas911 SET geom = ST_SetSRID(ST_MakePoint(longitud::float, latitud::float), 4326);


# TIPOS DE DATOS

Cada columna de una tabla de la base de datos debe tener un nombre y un tipo de dato. Se debe decidir qué tipo de datos se almacenará dentro de cada columna basado en la información que se espere extraer. Vamos a revisar las columnas y su tipo de datos para determinar cuales necesitamos cambiar para contestar   preguntas de investigación como:

1) ¿En qué alcaldía hay mayor número de delitos?
2) ¿Qué alcaldía tiene mayor número de áreas verdes? 

NOTA: Para esto usaremos funciones de agregación. Estas funciones nos permiten hacer operaciones sobre los registros en una columna y el numero de registros en la tabla.  
Como habrán notado ni la tabla de llamadas del 911 ni la de áreas verdes cuenta con un identificador de alcaldía o manzana, por lo que necesitaremos agregarlo a ambas.    

Pero primero, tenemos que generar las geometrías de la tabla llamadas 911 y darle el formato adecuado a cada columna para trabajar con ella 
sobre todo aquellas que contienen datos de fechas y horarios. 

``` sql
select column_name, data_type 
FROM
	information_schema.columns
WHERE
	table_schema = 'practica01'
	AND table_name = 'llamadas911';
```
Encontramos que todas las columnas son de tipo varchar, por lo que será necesario cambiar algunas como latitud y longitus a un valor numérico o codificar como fechas y horas las columnas que lo necesiten. Comenzaremos por modificar las columnas latitud y longitud ya que con ellas vamos a crear la geometría de los puntos

``` sql
ALTER TABLE practica01.llamadas911
ALTER COLUMN longitud TYPE numeric USING longitud::float;

ALTER TABLE practica01.llamadas911
ALTER COLUMN latitud TYPE numeric USING latitud::float;
``` 

Ya que estamos aquí, creamos la geometría.!!! 
Para ello vamos a usar la función AddGeometryColumn que acepta los siguientes atributos: 
``` sql
SELECT AddGeometryColumn ('esquema','nombre_tabla','columna_geometría',SRID,'TIPODEGEOMETRIA', 2);
SELECT AddGeometryColumn ('practica01','llamadas911','geom',4326,'POINT',2);
```
Después poblamos esta columnas con los datos de localización: 

``` sql
UPDATE esquema.tabla SET columna_geom = ST_SetSRID(ST_MakePoint(columna_longitud, columna_latitud), NUM_SRID);
UPDATE practica01.llamadas911 SET geom = ST_SetSRID(ST_MakePoint(longitud, latitud), 4326);
``` 
Exploremos los datos en el espacio visualizalos en QGIS. ¿Qué observas? al parecer no todos son parte de la CDMX. 
Vamos a extraer solo los que caen dentro de la ciudad con ayuda de una intersección espacial y crearemos una nueva tabla: 

``` sql
create table practica01.llamadas911_cdmx as 
select a.*
from practica01.llamadas911 a
join practica01.municipio_cdmx b 
on st_intersects(a.geom, b.geom)
``` 

# Trabajando con timestamp (fechas y horas )

Todos los sistemas operativos tiene la capacidad de decodificar el tiempo, Timestamp es un formato reconocido internacionalmente para representar fechas y horas que permite manipular este tipo de dato de mejor forma para su filtrado.  


Cambiaremos las columnas fecha_creacion y fecha_cierre a date y hora_creacion y hora_cierre a time. 

### FROM VARCHAR TO DATE
``` sql
select * from practica01.llamadas911_cdmx l limit 100

SELECT TO_DATE(fecha_creacion, 'YYYY-MM-DD') as fecha_prueba from practica01.llamadas911_cdmx;
SELECT TO_DATE(fecha_cierre, 'DD-MM-YY') as fecha_prueba from practica01.llamadas911_cdmx;

ALTER TABLE practica01.llamadas911_cdmx
ALTER COLUMN fecha_creacion TYPE date USING TO_DATE(fecha_creacion, 'YYYY-MM-DD');

ALTER TABLE practica01.llamadas911_cdmx
ALTER COLUMN fecha_cierre TYPE date USING TO_DATE(fecha_cierre, 'DD-MM-YY');
```
¿Qué diferencia encuentras en el formato de fecha de fecha_creacion y fecha_cierre?

### FROM VARCHAR TO TIME
``` sql
SELECT to_timestamp(hora_creacion , 'HH24:MI:SS' )::time as hora_creacion , 
       to_timestamp(hora_cierre, 'HH24:MI:SS' )::time as hora_cierre 
FROM practica01.llamadas911_cdmx;


ALTER TABLE practica01.llamadas911_cdmx
ALTER COLUMN hora_creacion TYPE time USING to_timestamp(hora_creacion , 'HH24:MI:SS' )::time;

ALTER TABLE practica01.llamadas911_cdmx
ALTER COLUMN hora_cierre TYPE time USING to_timestamp(hora_cierre, 'HH24:MI:SS' )::time;
```

### CONCATENANDO FECHA Y HORA EN TIMESTAMP
``` sql
ALTER TABLE practica01.llamadas911_cdmx add column fh_cierre timestamp; 
update practica01.llamadas911 set fh_cierre = fecha_cierre + hora_cierre; 

ALTER TABLE practica01.llamadas911_cdmx add column fh_creacion timestamp; 
UPDATE practica01.llamadas911 set fh_creacion = fecha_creacion + hora_creacion;
```

# Trabajando con funciones de agregación

Recordemos nuestras preguntas de investigación..!! 
1) ¿En qué alcaldía hay mayor número de delitos?
2) ¿Qué alcaldía tiene mayor número de áreas verdes? 

Para constestarlas usaremos "funciones de agregación", estas funciones permiten hacer operaciones sobre los registros en una columna o el numero de registros de la tabla. Dicho de otra manera, calculan un único resultado a partir de un conjunto de valores de entrada.
Para ambas preguntas necesitamos tener las columnas de manzanas_cvegeo y municipio_cvegeo, ya que estaremos haciendo la agregación sobre estas claves.   

``` sql
alter table practica01.areas_verdes2020 add column manzana_cvegeo varchar(16); 
alter table practica01.areas_verdes2020 add column municipio_cvegeo varchar(5); 

alter table practica01.llamadas911_cdmx add column manzana_cvegeo varchar(16); 
alter table practica01.llamadas911_cdmx add column municipio_cvegeo varchar(5); 
``` 

Si visualizamos las tablas encontraremos estas nuevas columnas pero con valores nulos, vamos a poblarlas a partir de una intersección espacial. 
Primero lo haremos para las llamadas de 911
NOTA: Estas consultas pueden tardar un momento así que se paciente =) ...! Mientras esperamos, vamos a visualizar los datos y hacernos algunas preguntas. 

``` sql
update practica01.llamadas911_cdmx a 
set manzana_cvegeo = b.manzana_cvegeo 
from practica01.manzanas_cdmx b
where st_intersects(a.geom, b.geom);

update practica01.llamadas911_cdmx a 
set municipio_cvegeo = b.municipio_cvegeo 
from practica01.municipio_cdmx b
where st_intersects(a.geom, b.geom); 
``` 

Verifiquemos que se hayan actualizado correctamente la información

``` sql
select * from practica01.llamadas911 l where manzana_cvegeo is null;
select * from practica01.llamadas911 l where municipio_cvegeo is null;
```

¿Porqué tenemos valores nulos en la clave de manzana y no en la de municipio?
EJERCICIO: Intenta hacerlo para la tabla de áreas verdes

### Filtrado de los datos

Después de haber pre-procesado los datos, regresemos(de nuevo) a nuestra primer pregunta de investigación:
1) ¿En qué alcaldía hay mayor número de delitos?

Para poder contestarla tenemos que seleccionar solo aquellas llamadas que pudieran estar relacionadas con delitos que afecten sus visitantes. 

Vamos a enlistar los distintos valores de la columna incidente_c4, que tiene información sobre el tipo de delito para identificar aquellos delitos que podrían afectar a un potencial visitante de las áreas verdes.
``` sql
select distinct (incidente_c4) from practica01.llamadas911 
``` 
Filtremos los datos

``` sql
create table practica01.delitos_av_911 as
select *
from practica01.llamadas911_cdmx lc where 
incidente_c4 = 'Accidente-Choque con Lesionados' or
incidente_c4 = 'Accidente-Choque con Prensados' or
incidente_c4 = 'Accidente-Choque sin Lesionados' or
incidente_c4 = 'Cadáver-Arma Blanca' or
incidente_c4 = 'Cadáver-Arma de Fuego' or
incidente_c4 = 'Detención Ciudadana-Agresión' or
incidente_c4 = 'Detención Ciudadana-Atropellamiento' or
incidente_c4 = 'Detención Ciudadana-Delitos Sexuales' or
incidente_c4 = 'Detención Ciudadana-Robo' or
incidente_c4 = 'Disturbio-Disparos' or
incidente_c4 = 'Robo-Pasajero Transporte Público' or
incidente_c4 = 'Robo-Repartidor' or 
incidente_c4 = 'Robo-Transeúnte' or
incidente_c4 = 'Robo-Vehiculo con Violencia' or
incidente_c4 = 'Robo-Vehículo sin Violencia' or
incidente_c4 = 'Sexuales-Abuso' or
incidente_c4 = 'Sexuales-Acoso' or
incidente_c4 = 'Sexuales-Acoso en transporte público' or
incidente_c4 = 'Sexuales-Otros/Vejaciones' or
incidente_c4 = 'Sexuales-Violación';
``` 

### Comencemos a obtener información

Con la función count() podemos contar cuantos delitos hay por delegación y por kilometro cuadrado 

``` sql
select a.*, b.municipio_nombre
from
(select count(*) as delitos, municipio_cvegeo 
from practica01.delitos_av_911 a
group by municipio_cvegeo ) as a
join practica01.municipio_cdmx b 
on a.municipio_cvegeo = b.municipio_cvegeo
order by delitos desc;
``` 
``` sql
select a.*, b.geom, b.municipio_nombre, st_area(b.geom::geography)/1000000 as area_m2, delitos/(st_area(b.geom::geography)/1000000) as delito_area
from
(select count(*) as delitos, municipio_cvegeo 
from practica01.delitos_av_911 a
group by municipio_cvegeo ) as a
join practica01.municipio_cdmx b 
on a.municipio_cvegeo = b.municipio_cvegeo
order by delito_area desc;
``` 





