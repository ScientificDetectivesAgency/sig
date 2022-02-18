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
