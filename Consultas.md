Consultas

¿Cual es la longitud total de todas las carreteras expresadas en kilómetros?

select sum(ST_Length(geom))/1000 AS vialudades from mex_eje_vial;

select sum(ST_Length(ST_Transform(geom, 900913)))/1000 AS vialidades from mex_eje_vial;

¿Cual es la superficie de la ciudad Toluca en hectáreas?

SELECT
  ST_Area(geom)/10000 AS hectarias
FROM mex_municipio
WHERE nomgeo = 'Toluca';

SELECT
ST_Area(ST_Transform(geom, 900913))/10000 AS hectarias
FROM mex_municipio
WHERE nomgeo = 'Toluca';

SELECT
ST_Area(ST_Transform(geom, 900913))/1000000 AS kilometros2
FROM mex_municipio
WHERE nomgeo = 'Toluca';

¿Cual es el municipio con mayor superficie del estado?

SELECT
  nomgeo,
  ST_Area(geom)/10000 AS hectarias
FROM
  mex_municipio
ORDER BY hectarias DESC
LIMIT 1;

SELECT
  nomgeo,
  ST_Area(geom)/1000000 AS kilometros2
FROM
  mex_municipio
ORDER BY hectarias DESC
LIMIT 1;

¿Cuál es la longitud de las carreteras contenidas por completo dentro de cada municipio?

Este es un ejemplo de "unión espacial", ya que estamos utilizando datos de dos tablas (haciendo una unión) pero utilizando una condición de interacción espacial ("contained") como la condición de unión en lugar del enfoque relacional habitual de unión de la clave primaria:

SELECT
  m.nomgeo,
  sum(ST_Length(r.geom))/1000 as vialidades_km
FROM
  mex_eje_vial AS r,
  mex_municipio AS m
WHERE
  ST_Contains(m.geom,r.geom)
GROUP BY m.nomgeo
ORDER BY vialidades_km;

Crear una tabla con todas las carreteras de la ciudad Toluca

Este es un ejemplo de "superposición", que tomo dos tablas y extrae una tabla nueva que contiene un resultado de un recorte espacial. A diferencia de la "Unión espacial" del ejemplo anterior, esta consulta en realidad crea nuevas geometrías. Una superposición es como una unión espacial "turbo-cargada", y es útil para un trabajo de análisis mas exacto:

SELECT
  ST_Intersection(r.geom, m.geom) AS interseccion_geom,
  ST_Length(r.geom) AS vial_orig_length,
  r.*
FROM
  mex_eje_vial AS r,
  mex_municipio AS m
WHERE  m.nomgeo = 'Toluca' AND ST_Intersects(r.geom, m.geom);


¿Cual es la longitud en kilómetros de "Tollocán" en Toluca?

SELECT
  sum(ST_Length(r.geom))/1000 AS kilometros
FROM
  mex_eje_vial r,
  mex_municipio m
WHERE  r.nomvial = '20 de Noviembre' AND m.nomgeo = 'Toluca'
        AND ST_Contains(m.geom, r.geom) ;

¿Cual es el polígono de municipios mas grande que tiene un agujero?

SELECT gid, nomgeo, ST_Area(geom) AS area
FROM mex_municipio
WHERE ST_NRings(geom) > 1
ORDER BY area DESC LIMIT 1;

Los nombres de los municipios por los que cruza la calle 'Paseo Tollocan'

SELECT m.nomgeo
FROM mex_municipio m JOIN mex_eje_vial r
ON ST_Crosses(m.geom, r.geom)
WHERE r.nomvial = '20 de Noviembre';


¿Cuál es la distancia que existe entre Toluca y la Ciudad de México?

select st_distance(t.geom, cdmx.geom) as distance
from ciudades_mundo t, ciudades_mundo cdmx
where t.name = 'Toluca' and cdmx.name = 'Mexico City' and cdmx.iso_a2 = 'MX'

En KM

select st_distance(st_transform(t.geom, 900913), st_transform(cdmx.geom, 900913))/1000 as distance
from ciudades_mundo t, ciudades_mundo cdmx
where t.name = 'Toluca' and cdmx.name = 'Mexico City' and cdmx.iso_a2 = 'MX'

El problema es que no se puede usar un sistema de coordenadas proyectadas para obtener distancias sobre una esfera. Es necesario tener en cuenta la curvatura terrestre. Para eso, podemos usar el tipo geography. Al llamar a st_distance sobre datos de tipo geography, se usa la esfera para hacer cálculos, y se devuelve el resultado en metros.

La consulta correcta sería:

select st_distance(geography(o.geom), geography(d.geom))/1000 as distance
from ciudades_mundo o, ciudades_mundo d
where o.name = 'Toluca' and d.name = 'Mexico City' and d.iso_a2 = 'MX'


El numero de escuelas que hay en cada uno de los municipios del Estado de México:

select m.nomgeo, count(p.nombre_act) as escuelas from mex_municipio m join
denue_inegi_15_ p on st_contains(m.geom, p.geom) where p.nombre_act like '%scuelas%'
group by m.nomgeo order by escuelas desc

select st_distance(st_transform(o.geom, 900913), st_transform(d.geom, 900913))/1000 as distancia_plana,
st_distance(geography(o.geom), geography(d.geom))/1000 as distancia_esfera
from ciudades_mundo o, ciudades_mundo d
where o.name = 'Toluca' and d.name = 'Mexico City' and d.iso_a2 = 'MX'



¿Cuál es la superficie del Estado de México?

ST_Transform(geom, 900913))/1000000 from mex_entidad where nomgeo = 'México'

Agrega el área o superficie del Estado de México en  el campo área de la tabla mex_entidad

Agregar el campo

ALTER TABLE mex_entidad ADD COLUMN area real;

update mex_entidad set area=(
select ST_Area(ST_Transform(geom, 900913))/1000000 from mex_entidad where nomgeo = 'México'
)

Cuál es el área de influencia en 100 metros de la vialidad con identificador número 3

CREATE TABLE buffer1 as
SELECT
  1 as gid,
  ST_Transform(ST_Buffer(
  (SELECT ST_Transform(geom, 900913) FROM mex_eje_vial WHERE gid = 2), 100, 'endcap=round join=round'), 4326)
AS geom;



## Referencias
https://postgis.net/docs/manual-dev/postgis-es.html#idm2850
https://gis.stackexchange.com/questions/169422/how-does-st-area-in-postgis-work
