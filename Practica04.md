### Accesibilidad con OSM y PgRouting

En esta práctica comenzar a revisar cómo desarrollar un modelo de de accesibilidad que vincule a cada nodo de la red a un conjunto de unidades de infraestructura. 
Además vamos a revisar modelos superficiales de costo agregado y cómo integrar variables que permitan modelar la accesibilidad basado en diferentes formas de traslado o medios de trnasporte.

Vamos a seguir trabajando con los puntos de teatros y descargaremos una nueva red de OSM con la herramienta OSMDownloader y Osm2Pgrouting. 
[1] Descarga la red correspondiente al polígono de la alcaldía Benito Juárez y llamale ways_bj
[2] Para la tabla de nodos y la de red crea los índices espaciales y no espaciales para las columnas: source, target y geom. 

````sql
CREATE INDEX ways_bj_geom_idx
  ON practica03.ways_bj
  USING GIST (geom);

CREATE index ways_bj_source_idx ON practica03.ways (source);
CREATE index ways_bj_target_idx ON practica03.ways (target);
````
[3] Agrega el identificador del nodo más cercano a la tabla de teatros. ¿Porqué es necesario volverlo a hacer?
[4] Ahora vamos a modelar el costo. Agraga la columna de tiempo en minutos y calculalo

En este ejercicio vamos a experimentar con la función pgr_dijkstraCost, primero probaremos la versión del algoritmo ONE TO MANY esto significa que va a calcular todas las rutas posibles de un nodo a muchos nodos.¿Qué diferencia encuentras con los resultados que arrojaría pgr_dijkstra?
https://docs.pgrouting.org/3.1/en/pgr_dijkstraCost.html 

````sql
SELECT * FROM pgr_dijkstraCost(
	'SELECT id, source, target, cost_s as cost, reverse_cost_s as reverse_cost FROM practica03.ways_bj',
	84605, (SELECT array(SELECT DISTINCT id FROM practica03.ways_vertices_pgr)));
  ````
 [5] Qué estamos haciendo al integrar la consulta: 
 ````sql
 SELECT array(SELECT DISTINCT id FROM practica03.ways_vertices_pgr))
 ````
 Vamos a visualizar los resultados 
 
 ````sql 
select n.id, costos.agg_cost, n.geom
from
(SELECT * FROM pgr_dijkstraCost(
	'SELECT id, source, target, cost_s as cost, reverse_cost_s as reverse_cost FROM practica03.ways_bj',
	84605, (SELECT array(SELECT DISTINCT id FROM practica03.ways_bj_vertices_pgr)))) as costos
join practica03.ways_bj_vertices_pgr n
on n.id = costos.end_vid
 ````
 
 [6] ¿Qué pasa si hacemos el grafo no-dirigido?
 
 ````sql 
select n.id, costos.agg_cost, n.geom
from
(SELECT * FROM pgr_dijkstraCost(
	'SELECT id, source, target, cost_s as cost, reverse_cost_s as reverse_cost FROM practica03.ways_bj',
	84605, (SELECT array(SELECT DISTINCT id FROM practica03.ways_bj_vertices_pgr)), FALSE)) as costos
join practica03.ways_bj_vertices_pgr n
on n.id = costos.end_vid
 ````

Ahora vamos a hacer el mismo ejercicio pero con los datos de teatros, ahora vamos a evaluar todos los puntos contra todos. 


 ````sql
SELECT * FROM pgr_dijkstraCost(
	'SELECT id, source, target, cost_s as cost, reverse_cost_s as reverse_cost FROM practica03.ways_bj',
	(SELECT array(SELECT closest_node FROM practica03.teatros)),
	(SELECT array(SELECT DISTINCT id FROM practica03.ways_bj_vertices_pgr))
)
 ````
Ahora para cada nodo de inicio (start_vid) tenemos el costo a todos los nodos finales. ¿Qué problema notas en la tabla?
Tenemos cada nodo de la red repetido tantas veces como teatros. Para el cálculo de accesibilidad que estamos haciendo, lo que necesitamos es que cada nodo quede asignado al teatro que tiene a un menor costo, de esa forma podemos crear una cobertura de los nodos con los y crear un mapa de accesibilidad por costo. Lo único que necesitamos es,  seleccionar los nodos que tengan menos distancia para cada nodo de origen:


````sql
SELECT DISTINCT ON (end_vid)
                    end_vid as nodo, start_vid as bodega, agg_cost
FROM
(SELECT * FROM pgr_dijkstraCost(
	'SELECT id, source, target, cost_s as cost, reverse_cost_s as reverse_cost FROM practica03.ways_bj',
	(SELECT array(SELECT closest_node FROM practica03.teatros)),
	(SELECT array(SELECT DISTINCT id FROM practica03.ways_bj_vertices_pgr))
)) as costos
ORDER  BY end_vid, agg_cost asc
````
[7]Para visualizarlos haz un join con las geometrías de los vertices

Aquí lo que hicimos fue asignar a cada teatro todos los nodos de la red que tengan el costo menorusando DISTINCT ON y ORDER BY , al ordenar nodo final y costo, el DISTNCT toma el de menor costo.

 
