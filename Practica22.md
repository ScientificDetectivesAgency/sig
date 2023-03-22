# SIG: Casos de la vida real 
# Episodio 01

Vamos a comenzar a trabajar con operaciones espaciales básicas como lo son los buffers, clips, disolve, entre otras.!! 

El primer paso es proyectar las geometrías de las manzanas para que sean compatibles con las estaciones del metro y el límite metropolitano
Es mejor proyectar las manzanas para que todo está en coordenadas planas, de ese modo los cálculos geométricos son mucho más rápidos

```sql 
 ALTER TABLE estaciones_metro
   ALTER COLUMN geom
   TYPE Geometry(MultiPoint, 32614)
   USING ST_Transform(geom, 32614);
```
**Nota:** Repite la misma operación para todas las capas que estén en coordenadas geográficas (SRID: 4326)

Ahora vamos a cortar las manzanas con el polígono del límite metropolitano (lo que se llama un clip, pues) y meter el resultado en la tabla  merge_manzanas:

```sql
create table manzanas_zmvm as(
select merge_manzanas.*
from merge_manzanas
inner join limite_metropolitano on
st_intersects(limite_metropolitano.geom,merge_manzanas.geom))
``` 
Lo que hicimos fue en realidad un inner join pero como condición utilizamos una relación espacial: st_intersects

Ahora creamos un índice espacial sobre la geometría

```sql
create index manzanas_zmvm_gix on manzanas_zmvm using GIST(geom);
```

vemos si los id son únicos

```sql
select count(*) from manzanas_zmvm  group by id order by count(*) desc;
```
¿Por qué no son únicos los ids?

como la tabla tiene unos id's repetidos, alteramos la columna para que sean únicos
```sql
CREATE SEQUENCE "manzanas_zmvm_id_seq";
update manzanas_zmvm set id = nextval('"manzanas_zmvm_id_seq"');
```

Creamos los constraints necesarios y agregamos un PK
```sql
ALTER TABLE manzanas_zmvm ALTER COLUMN "id" SET NOT NULL;
ALTER TABLE manzanas_zmvm ADD UNIQUE ("id");
ALTER TABLE manzanas_zmvm ADD PRIMARY KEY ("id");
```

Ahora podemos empezar a hacer algunas preguntas interesantes, por ejemplo:
```sql
--¿cuantas manzanas quedan a 500 metros de cada estación del metro?
select foo.* from
(with buf as (select st_buffer(estaciones_metro.geom,500.0) as geom , estaciones_metro.nombreesta as estacion from estaciones_metro)
select count(manzanas_zmvm.id), buf.estacion from manzanas_zmvm join buf on
st_intersects(buf.geom,manzanas_zmvm.geom)
group by buf.estacion) as foo;
```

Tambien podemos preguntar ¿Cuánta gente vive a 500 m de una estación del metro?
--Nota: La columna pob1 contiene la población de cada manzana

```sql
select foo.* from
(with buf as (select st_buffer(estaciones_metro.geom,500.0) as geom , estaciones_metro.nombreesta as estacion from estaciones_metro)
select sum(manzanas_zmvm.pob1), buf.estacion from manzanas_zmvm join buf on
st_intersects(buf.geom,manzanas_zmvm.geom)
group by buf.estacion) as foo;
```

--TAREA: ¿Cuántas personas no viven a 500 metros de una estación de metro?
--Hint: Tienes que sumar el resultado de la expresión de arriba y restarla de la población total como es el primer quiz, puedes hacerlo en dos querys

Ahora vamos a crear una tabla que contenga la clave de la manzana (cvegeo), la geometría de la manzana y la clave de la colonia
--en la que se encuentra la manzana:

```sql
select  manzanas_zmvm.id, manzanas_zmvm.cvegeo, colonias.id_colonia,manzanas_zmvm.geom
into manzanas_colonias
from manzanas_zmvm 
join colonias 
on st_intersects(colonias.geom, manzanas_zmvm.geom);
```

Finalmente, para entender algunas de las diferencias entre Arc y el modelo PostGis, agregaremos las manzanas en colonias a través del id de colonia,
lo que en Arc se conoce como 'dissolve'.

```sql
select st_union(geom), id_colonia into manzanas_union
from manzanas_colonias
group by id_colonia;
```

-- ¿Qué tipo de geometría obtenemos?-- 
