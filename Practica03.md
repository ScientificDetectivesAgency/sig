### Análisis de redes
``` sql
create extension pgrouting; 
create schema practicas03;
```

**EJERCICIO[1]:** De la red de calles de la CDMX, crea una tabla que se llame ways_coyoacan en el eschema practica03 



Calcular la topología https://docs.pgrouting.org/3.0/es/pgr_createTopology.html

``` sql
alter table practica03.ways_coyoacan add column source integer;
alter table  practica03.ways_coyoacan add column target integer;
select pgr_createTopology('practica03.ways_coyoacan', 0.001, 'the_geom', 'gid', 'source', 'target')
--select * from practica03.ways_coyoacan limit 10
``` 

**EJERCICIO[2]:** Agrega la columna de geometría en la tabla de teatros para poder visualizarla como puntos, filtra solo los de coyoacán y visualizalos sobre la red de calles


Ahora vamos a comenzar a jugar con los costos de traslado sobre la red, comencemos con algo simple. Usemos distancias

**EJERCICIO[3]:** Agrega la columna long y calcula la distancia en kilometros de cada uno de los arcos de la red, toma en cuenta la proyección de las capas y si es necesario castealas o reproyectalas  
**EJERCICIO[4]:** Ahora vamos a trabajar con tiempos de traslado basandonos en las velocidades mínimas permitidas registradas por OSM, para esto vas a usar la formula de V = d/tdespeja y calcula el tiempo.

¿Qué columna tiene los datos de velocidad? ¿cuál es la diferencia entre maxspeed_forward y maxspeed_backward

Ahora vamos a trabajar con los datos de OSM y los tipos de camino, exploremos la tabla configuracion_osm la columna tag_id del catalogo de configuración nos ayudará a obtener los datos del tipo de camino 

``` sql
select * from practica03.configuracion_osm co
``` 
Los catálogos de información nos ayudan a obtener etiquedas basado en información: 

``` sql
alter table practica03.ways_coyoacan add column tag_value text;
update practica03.ways_coyoacan a
set tag_value = b.tag_value
from practica03.configuracion_osm b
where a.tag_id = b.tag_id
``` 
¿Te da un error? ¿Porqué?*
¿Cómo codifica OSM los tipos de calles? https://wiki.openstreetmap.org/wiki/Key:highway

``` sql
select distinct (tag_value)  from practica03.ways_coyoacan    
``` 

Agregamos la columna para la función de costos

``` sql
alter table practica03.ways_coyoacan add column costo float;
``` 

Asignamos los costos a la columna: recuerda que a mayor valor de costo el algoritmo no va a elegi ir por ese camino, menor costo le dará prioridad

``` sql
update practica03.ways_coyoacan set costo =
case
    when tag_value like 'tertiary_link' then velocidad*0.05 
    when tag_value like 'tertiary' then velocidad*0.05 
    when tag_value like 'living_street' then velocidad*0.005
    when tag_value like 'residential' then velocidad*0.005
    else velocidad*5
  end;
``` 

Comencemos a calcular rutas 
https://docs.pgrouting.org/latest/en/pgr_dijkstra.html

``` sql
SELECT * FROM pgr_dijkstra(
  ' SELECT
      gid AS id,
      source::integer,
      target::integer,
      costo::double precision AS cost
    FROM practica03.ways_coyoacan',
  1799, 585, directed := false)
``` 
Ahora seleccionemos las geometrías
``` sql
 select c.gid, c.the_geom 
from practica03.ways c,
(SELECT * FROM pgr_dijkstra(
  ' SELECT
      gid AS id,
      source::integer,
      target::integer,
      costo::double precision AS cost
    FROM practica03.ways_coyoacan',
  1799, 585, directed := false)) as ruta
where c.gid = ruta.edge;
```
Como habrás observado al hacer el join con la tabla de edges o arcos, es posible dibujar la ruta que se trazó. Pero, ¿Cómo podemos comenzar a trabajar con puntos reales? Primero necesitamos convertir los puntos a evaluar a nodos de la red, esto para identificar los nodos de inicio y destino y que todo esté en los mismos términos. Para ello vamos a utilizar la siguiente consulta:  

```sql
alter table #tabla_puntos# add column closest_node bigint; 
update #tabla_puntos# set closest_node = c.closest_node
from  
(select b.id as #id_puntos#, (
  SELECT a.id
  FROM #tabla_vertices# As a
  ORDER BY b.geom <-> a.the_geom LIMIT 1
)as closest_node
from  #tabla_puntos# b) as c
where c.#id_puntos# = #tabla_puntos#.id
```
Analiza qué hace, investiga para qué funciona este operador <->

Vamos a trabajar con los datos de teatros que se encuentran en la base, recuerda que los datos deben estár en la misma proyección y acotados al área que se va a trabajar. Para este ejercicio debes elegir una alcaldía, recortar la red de calles y los puntos de teatros y sobre la tabla de teatros crear la columna closest_node que va a contener el nodo más cercano de la red. 

```sql
alter table teatros  add column closest_node bigint; 

update teatros set closest_node = c.closest_node
from  
(select b.id, (
  SELECT a.id
  FROM ways_vertices_pgr As a
  ORDER BY b.geom <-> a.the_geom LIMIT 1
)as closest_node
from  teatros b) as c
where c.id = teatros.id
```

Ahora imagina que eres el/la secretaria de cultura de la Ciudad de México y te interesa crear corredores culturales temáticos. En este caso de teatros que puedan encontrarser cerca y representen una mejor oferta cultural. Además, como secretari@ de cultura te interesa fomentar actividades económicas asociadas a dichas actividades como restaurntes o cafeterías. 

**EJERCICIO[5]:** En la unidad compartida encontrarás datos Directorio Estadístico Nacional de Unidades Económicas (DENUE), selecciona aquellos establecimientos que pertenecen a estas categorías (restaurantes y cafés) y crea una nueva tabla. A esta tabla también debes agregar y poblar la columna closest_node. 

Ahora vamos a crear rutas que recorran varios puntos en el menor tiempo posible, para ello vamos a utilizar el algortimo del agente viajero. 

### Agente viajero

El problema del Agente viajero o TSP plantea la siguiente pregunta: 
"Dada una lista de ciudades y las distancias entre cada par de ciudades, ¿cuál es la ruta más corta posible que visita cada ciudad exactamente una vez y regresa al ciudad de origen?"

Ahora vamos a llevarlo a nuestro problema
¿Cómo podemos trazar rutas que incluyan varios teatros cafés y restaurantes que estén cerca de estaciones de metro? 

En este ejercicio, debes seleccionar zonas donde te interese trazar 4 rutas que partan y terminen en una estacion del metro y recorran 2 teatros una cafetería y un reataurante. Con los identificadores de los nodos calcula la secuencia optima para recorrerlos conla siguiente consulta: 


```sql
SELECT seq, node, the_geom FROM pgr_TSP(
    $$
    SELECT * FROM pgr_dijkstraCostMatrix(
        'SELECT gid as id, source, target, cost FROM ways',
        (
            SELECT ARRAY[9769, 20604, 48398, 42533, 49819, 41141, 45520] node_array
        ),
        directed := false
    )
    $$,
    start_id := 9769,
    randomize := false
) as res
JOIN ways_vertices_pgr  ovp 
ON ovp.id = res.node;
```

Ahora podemos trazar rutas por pares de nodos 

```sql
CREATE TABLE ruta_tsp as
(WITH path_1_2 AS (
        SELECT  b.geom, a.*
		from (select node, edge as id, cost
						from pgr_bdDijkstra('SELECT  id::int4, source::int4 ,target::int4,  cost FROM  osmcdmx',
														 9769, 48398 ,FALSE)
												)  as a join osmcdmx b on a.id = b.id),
		path_2_3 AS (
        SELECT  b.geom, a.*
		from (select node, edge as id, cost
						from pgr_bdDijkstra('SELECT  id::int4, source::int4 ,target::int4,  cost FROM  osmcdmx',
														 48398, 42533 ,FALSE)
												)  as a join osmcdmx b on a.id = b.id),  
		path_3_4 AS (
        SELECT  b.geom, a.*
		from (select node, edge as id, cost
						from pgr_bdDijkstra('SELECT  id::int4, source::int4 ,target::int4,  cost FROM  osmcdmx',
														 42533, 45520,FALSE)
												)  as a join osmcdmx b on a.id = b.id),
												
        path_4_5 AS (
        SELECT  b.geom, a.*
		from (select node, edge as id, cost
						from pgr_bdDijkstra('SELECT  id::int4, source::int4 ,target::int4,  cost FROM  osmcdmx',
														 20604,9769, FALSE)
												)  as a join osmcdmx b on a.id = b.id)
SELECT * FROM path_1_2
union
SELECT * FROM  path_2_3
union
SELECT * FROM path_3_4
union
SELECT * FROM  path_4_5)
```

**EJERCICIO[5]:** Investiga cómo puedo trazar una línea teniendo la información que me da el agente viajero
**EJERCICIO[6]:** Investiga cómo calcular áreas de servicio y cómo funciona el algoritmo Dijkstra 

### Areas de servicio 

Para encontrar el area de servicio alrededor de un punto necesitamos conocer todos los nodos de la red que quedan a un menos de un determinado costo del nodo origen. Pgrouting provee una función para este tipo de análisis pgr_drivingDistance. Como ejemplo, vamos a encontrar el area de servicio de 10 kilometros  alrededor del nodo 22084:

``` sql
create table servicio as
select * from osmcdmx_vertices_pgr v,
(SELECT node FROM pgr_drivingDistance(
        'SELECT id, source, target,
         (st_length(geom)/1000) as cost
         FROM osmcdmx',
        22048, 10
      )) as service
where v.id = service.node;
```
Ahora pensemos en nuestro caso de estudio, qué pasa si lo que queremos es saber que hay a 10km de distancia sobre la red de todos los teoatros. 
primero vamos a probar construir un array con los identificadores del nodo mś cercano sobre la red 

```sql
select array(select closest_node from teatros)
```
Esta consulta te da en forma de array todos los nodos y siver como input para determinar el conjunto de puntos de origen en la siguiente consulta

```sql
create table servicio_array as
SELECT * FROM pgr_drivingDistance(
      'SELECT id, source, target,
       st_length(the_geom) as cost,
       FROM osmcdmx',
      (select array(select closest_node from teatros)), 0.5);
 ```
 
