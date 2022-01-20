
Supongamos que obtuvimos las coordenadas de varios puntos alrededor de un lago y las representamos como puntos, pero lo que necesitamos es un poligo que represente el lago. 
El primer paso es crear una linea a partir de los puntos de la tabla waypoints y para ello debemos considerar el orden en el que los juntemos, por lo que es necesario ordenarlos por id, usando la cláusula GROUP BY track_id, que agrupa los puntos de acuerdo a un identificador de recorrido.

``` sql
CREATE TABLE gps_tracks AS
SELECT
ST_MakeLine(geom) AS geom, track_id
FROM ( SELECT * FROM waypoints ORDER BY id) AS ordered_points
GROUP BY track_id;
``` 

¿Que tipo de geometría generamos al unir los puntos? Veamos:

``` sql
SELECT ST_asText(geom) from gps_tracks;
``` 

Si visualizas la capa en QGi.!!
Notarás que la linea que construimos contiene una auto-intersección y que no está cerrada (el primero y el último punto no coinciden), para resolver este problema, vamos a generar un nodo en la auto-intersección:

Pregunta # 1: ¿Qué hace el operador ST_UnaryUnion() y, más en general, ¿Qué hace una unión en Postgis?

``` sql
UPDATE gps_tracks SET geom = ST_UnaryUnion(geom)
```

ST_UnaryUnion(geom) para poligonos es un equivalente del dissolve en Arc, para lineas se usa esta funcion para agregar un nodo de unión entre ellas cuando intersectan. Para ver el resultado de la operación anterior, vamos a desbaratar la linea que creamos en sus componentes:

``` sql
SELECT  ST_asText(ST_GeometryN(geom,generate_series(1,ST_NumGeometries(geom)))) AS lines
FROM gps_tracks
select st_geometryn(geom, generate_series(1,st_numgeometries(geom))) from gps_tracks -- colection 
select st_numgeometries(geom) from gps_tracks
```

st_astext, Devuelve la epresentación de las geometrías en WKT ( Well-Known Text) sin estar codificado como SRID
st_geometryN, nos dice en caso de ser una colección de geometrias las N geometrías contenidas en esa tabla 
ST_NumGeometries, numero de geometrías en una colección 

Esto nos regresa un conjunto de objetos tipo Linestring (véanlo). Ahora que hemos separado el Multilinestring en sus componentes, vamos a desarmar éstas líneas en sus componentes básicos:

Ya que la geometría de líneas como tal no existe es necesario crearla a través de una subconsulta

``` sql
SELECT st_astext(ST_GeometryN(geom, generate_series(1,ST_NumGeometries(geom)))) FROM gps_tracks
``` 

``` sql
SELECT ST_AsText(ST_PointN(line_geom,generate_series(1, ST_NPoints(line_geom)))) point_geom
FROM (SELECT ST_GeometryN(geom, generate_series(1,ST_NumGeometries(geom))) AS line_geom 
	  FROM gps_tracks
) AS foo;
```
Nota que hicimos las dos consultas anidadas, es decir, primero hacemos la consulta anterior y luego la envolvemos con la consulta que nos regresa los puntos. Ahora, para poder visualizar el resultado en QGis, necesitamos agregar un id a los puntos y ponerlos en una tabla:

``` sql
CREATE TABLE waypoints_nuevos AS
SELECT 
   ST_PointN(
	  lines,
	  generate_series(1, ST_NPoints(lines))
   ) as geom
FROM (
	SELECT 
		ST_GeometryN(geom,
	generate_series(1,ST_NumGeometries(geom))) AS lines
	FROM gps_tracks
) AS foo;
``` 

``` sql
alter table waypoints_nuevos add column id serial;
``` 

Como podemos ver, creamos un punto el la auto-intersección. Ahora regresemos a la tabla gps_tracks y creemos un polígono:

``` sql
CREATE TABLE gps_lakes AS
SELECT
ST_BuildArea(geom) AS lake,
track_id
FROM gps_tracks;
```
Ahora pueden visualizar el lago en QGis. 
PREGUNTA #2: ¿Qué hubiera pasado si no ponemos un nodo en la auto-intersección?
ver en la documentación--> ¿Qué es un polígono?
