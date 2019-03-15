
### Procesamiento de Nube de Puntos (LIDAR) con PostgreSQL pointcloud ###

Los sensores LIDAR producen rápidamente millones de puntos con un gran número de variables asociadas, esto vuelve compleja la tarea de almacenar de manera eficiente esa información y además poder acceder a ella. La extensión ``pointcloud`` de PostgreSQL tiene la capacidad de agregar un número de puntos determinado por el usuario en poligonos llamados ``patches`` y además conservar los atributos de cada variable para optimizar las consultas entre ellas y otros objetos espaciales.

 Las variables capturadas por los sensores LIDAR varían según el sensor y el proceso de captura. Por lo que además de contener valores x, y, z también contendrán docenas de variables como intensidad, número de retorno; valores rojo, verde y azul; tiempos de retorno; entre otros. Y un punto muy importante es que no hay coherencia en la forma en que se almacenan las variables: la intensidad puede almacenarse en un entero de 4 bytes o en un solo byte; x, y, z puede duplicarse o escalar enteros de 4 bytes.

Para lidiar con esta heterogeneidad en los datos PostgreSQL Pointcloud utiliza un documento llamado ``esquema`` que contiene los atributos de cada variable para describir el contenido de cualquier punto LIDAR en particular. Cada punto contiene una serie de dimensiones, y cada dimensión puede ser de cualquier tipo de datos, con escalas. El formato de documento de esquema utilizado por PostgreSQL Pointcloud es el mismo que utiliza la biblioteca ``PDAL``, lo que permite cargar los datos de la nube de puntos desde esta biblioteca a PostgreSQL.

El siguiente es un ejemplo de un schema usado por PDAL: 

```json 
"<?xml version="1.0" encoding="UTF-8"?>
<pc:PointCloudSchema xmlns:pc="http://pointcloud.org/schemas/PC/" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
 <pc:dimension>
  <pc:position>1</pc:position>
  <pc:size>2</pc:size>
  <pc:description>Representation of the pulse return magnitude</pc:description>
  <pc:name>Intensity</pc:name>
  <pc:interpretation>uint16_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>2</pc:position>
  <pc:size>1</pc:size>
  <pc:description>Pulse return number for a given output pulse. A given output laser pulse can have many returns, and they must be marked in order, starting with 1</pc:description>
  <pc:name>ReturnNumber</pc:name>
  <pc:interpretation>uint8_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>3</pc:position>
  <pc:size>1</pc:size>
  <pc:description>Total number of returns for a given pulse.</pc:description>
  <pc:name>NumberOfReturns</pc:name>
  <pc:interpretation>uint8_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>4</pc:position>
  <pc:size>1</pc:size>
  <pc:description>Direction at which the scanner mirror was traveling at the time of the output pulse. A value of 1 is a positive scan direction, and a bit value of 0 is a negative scan direction, where positive scan direction is a scan moving from the left side of the in-track direction to the right side and negative the opposite</pc:description>
  <pc:name>ScanDirectionFlag</pc:name>
  <pc:interpretation>uint8_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>5</pc:position>
  <pc:size>1</pc:size>
  <pc:description>Indicates the end of scanline before a direction change with a value of 1 - 0 otherwise</pc:description>
  <pc:name>EdgeOfFlightLine</pc:name>
  <pc:interpretation>uint8_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>6</pc:position>
  <pc:size>1</pc:size>
  <pc:description>ASPRS classification.  0 for no classification.  See LAS specification for details.</pc:description>
  <pc:name>Classification</pc:name>
  <pc:interpretation>uint8_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>7</pc:position>
  <pc:size>4</pc:size>
  <pc:description>Angle degree at which the laser point was output from the system, including the roll of the aircraft.  The scan angle is based on being nadir, and -90 the left side of the aircraft in the direction of flight</pc:description>
  <pc:name>ScanAngleRank</pc:name>
  <pc:interpretation>float</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>8</pc:position>
  <pc:size>1</pc:size>
  <pc:description>Unspecified user data</pc:description>
  <pc:name>UserData</pc:name>
  <pc:interpretation>uint8_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>9</pc:position>
  <pc:size>2</pc:size>
  <pc:description>File source ID from which the point originated.  Zero indicates that the point originated in the current file</pc:description>
  <pc:name>PointSourceId</pc:name>
  <pc:interpretation>uint16_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>10</pc:position>
  <pc:size>2</pc:size>
  <pc:description>Red image channel value</pc:description>
  <pc:name>Red</pc:name>
  <pc:interpretation>uint16_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>11</pc:position>
  <pc:size>2</pc:size>
  <pc:description>Green image channel value</pc:description>
  <pc:name>Green</pc:name>
  <pc:interpretation>uint16_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>12</pc:position>
  <pc:size>2</pc:size>
  <pc:description>Blue image channel value</pc:description>
  <pc:name>Blue</pc:name>
  <pc:interpretation>uint16_t</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>13</pc:position>
  <pc:size>8</pc:size>
  <pc:description>GPS time that the point was acquired</pc:description>
  <pc:name>GpsTime</pc:name>
  <pc:interpretation>double</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>14</pc:position>
  <pc:size>8</pc:size>
  <pc:description>X coordinate</pc:description>
  <pc:name>X</pc:name>
  <pc:interpretation>double</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>15</pc:position>
  <pc:size>8</pc:size>
  <pc:description>Y coordinate</pc:description>
  <pc:name>Y</pc:name>
  <pc:interpretation>double</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:dimension>
  <pc:position>16</pc:position>
  <pc:size>8</pc:size>
  <pc:description>Z coordinate</pc:description>
  <pc:name>Z</pc:name>
  <pc:interpretation>double</pc:interpretation>
  <pc:active>true</pc:active>
 </pc:dimension>
 <pc:metadata>
  <Metadata name="compression" type="string">dimensional</Metadata>
 </pc:metadata>
 <pc:orientation>point</pc:orientation>
 <pc:version>1.3</pc:version>
</pc:PointCloudSchema>
"
```


Para este ejercicio vamos a crear una base de datos, cargar una nube de puntos de una zona de la Ciudad de México y calcularemos las alturas promedio de los edificios en esa área, así como visualizar los resultados.

En este link puedes descargar los datos con los que vas a trabajar
https://github.com/ScientificDetectivesAgency/pgpointcloud/raw/master/practica_nubesdepuntos.zip

<a href="https://github.com/ScientificDetectivesAgency/pgpointcloud/raw/master/practica_nubesdepuntos.zip
"><img src="https://cdn3.iconfinder.com/data/icons/delivery-and-logistics-1/24/logistics-24-512.png></a>

Primero es necesario crear una base de datos con extension espacial, que ademas tenga las extensiones pointcloud y pointcloud_postgis

```sql
create extension postgis;
create extension pointcloud;
create extension pointcloud_postgis;
```

Ahora vamos a cargar la nube de puntos en la base de datos, en la carpeta con los archivos necesarios disponible en este repositorio, encontraras los archivos ``edificios.las`` (datos LIDAR) y pipeline.json. Si abres este archivo se vera así:


```json
{
  "pipeline":[
    {
      "type":"readers.las",
      "filename":"edificios.las", <-- nombre del archivo que se va a cargar en la base de datos
      "spatialreference":"EPSG:32614" <--- Sistema de referencia de origen del archivo .las
    },
    {
      "type":"filters.chipper",
      "capacity":1500
    },
    {
      "type":"writers.pgpointcloud",
      "connection":"host='localhost' dbname='CAMBIA EL NOMBE Y PON EL DE TU BASE' user='postgres' password='postgres'", <--- Conexión a la 																base de datos 
      "table":"pcpatches_1500", <--- Nombre de la tabla dentro de la base de datos
      "compression":"dimensional",
      "srid":"32614" <-- Sistema de referencia de destino
    }
  ]
}
```
**Cambia los parametros donde sea necesario**


Ahora desde la terminal copia y pega la siguiente linea de código:
 
        pdal pipeline --input pipeline.json

Esto cargará en la tabla que se indica en el pipeline el archivo edificios.las y creará el esquema de las variables. También carga el archivo edificios.shp

	```pc_get() Regresa los valores en todas las dimensiones de un array```
	```pc_explode() Convierte el Patch en un set de varios puntos y permite consultar la infomación asociada a las 		           variables almacenada en ellos.```	
	```pc_intersects() Intersecta un objeto geométrico con un patch

Primero vamos a familiarizarnos con las funciones que vamos a utilizar y las consultas que se pueden hacer

```sql
--(3) Ahora queremos obtene las variables enteriores solo donde el 'ReturnNumber' sea
--igual a 1
select foo.*
--(2)De la subconsulta a vamos a tomar la información de los patches que corresponde a 
--'ReturnNumber', el valor 'Z' y los pcpoints dentro del patch
from
(select pc_get(pc_explode(a.pa), 'ReturnNumber') as return_number, 
       pc_get(pc_explode(a.pa), 'Z') as z,
	   pc_explode(a.pa) as pcpoint

--(1)En esta parte de la consulta seleccionamos toda la información de los patches 
--que intersectan con el buffer de los edificios llamamos a la subconsulta "a"
from
(select p.* from pcpatches p, (select * from edificios where id = 35) as e
 where pc_intersects(pa, st_buffer(e.geom, 2))) as a) as foo
where  return_number = 1
```
Ahora calcularemos las alturas

```sql
--(4)Calculamos las alturas mínima y máxima, la diferencias entre ambas y un último campo
--que calcula la diferencia entre la altura promedio y la altura mínima o de la base
select id_ed, min(z), max(z), max(z)-min(z) as alt, avg(z)-min(z) as avg_alt
--(3)Seleccionamos todos los campos
from
(select e.*

--(2)llamamos los mismo campos que en la consulta anterior
from
(select foo.id_ed, pc_get(pc_explode(foo.pa), 'ReturnNumber') as return_number, 
       pc_get(pc_explode(foo.pa), 'Z') as z,
	   pc_explode(foo.pa) as pcpoints

--(1)Hacemos lo mismo que en la consulta anterior pero ahora de la tabla completa de edificios
from
(select e.id as id_ed, p.*
from pcpatches p, edificios  e
where pc_intersects(pa, st_buffer(e.geom, 2))) as foo) as e
where  return_number = 1) as foo
group by id_ed;
```

**EJERCICIO 1:** Contesta lo siguiente: 

1. Investiga qué es un PcPoint y un PcPatch y cuáles son las funciones disponibles en pointcloud PostgreSQL para el uso de cada uno de ellos. 
2. ¿Qué es un pdal pipeline y para qué sirve?
	Puedes consultar a la documentación en la siguiente liga: https://github/pgpointcloud/pointcloud
3. Propón un caso de estudio que se pueda resolver con el almacenamiento de nubes de puntos en bases de datos y cómo lo resolverías.
