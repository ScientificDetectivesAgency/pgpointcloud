
### Procesamiento de Nube de Puntos con pointcloud ###


Para el procesamiento de nubes de puntos el uso de bases de datos espaciales, es una herrmienta útil ya que ayuda a organizar la información de manera que sea posible agrupar por patches el numero de puntos necesarios para su indexación. Conserbando información como el numero de retorno para el caso de datos LIDAR o el valor Z. Para ello además vamos a usar PDAL una librería hecha en C++ para el manejo de nubes de puntos y la extension de Postgres pointcloud. 

Primero es necesario crear una base de datos con extension espacial, que ademas tenga las extensiones pgpointcloud y pointcloud_postgis

create extension postgis;
create extension pointcloud;
create extension pointcloud_postgis;

-- Ahora vamos a cargar la nube de puntos en postgis, en la carpeta practica_nubesdepuntos encontraras un archivo con extensión .las
-- y un archivo que se llama pipeline.txt, este archivo abrelo y encontraras lo siguiente: 

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


--EJERCICIO 1: Investiga dentro de postgres qué es un patch en la extension pointcloud y las funciones disponibles
--para su uso. 
-- Qué es un pdal pipeline y para qué sirve
-- Puedes acudir a la documentación en la siguiente liga: https://github/pgpointcloud/pointcloud

Ahora desde la terminal copia y pega la siguiente linea de código:
 
        pdal pipeline --input pipeline.txt

--con ello se creará en tu base de datos la tabla que corresponte a la nube de puntos de edificios y podrás comenzar a hacer consultas ----espaciales. En este ejercicio calcularemos las alturas de los edificios en la zona donde se tomó la nube de puntos. Por lo que deberás
--cargar el archivo .shp del mismo nombre a la base 

-- pc_get() Return values of all dimensions in an array

```sql
select e.*
from
(select pc_get(pc_explode(foo.pa), 'ReturnNumber') as return_number, 
       pc_get(pc_explode(foo.pa), 'Z') as z,
	   pc_explode(foo.pa) as exp
from
(select p.*
from pcpatches p, 
     (select * from edificios where id = 35) as e
where pc_intersects(pa, st_buffer(e.geom, 2))) as foo) as e
where  return_number = 1
```
