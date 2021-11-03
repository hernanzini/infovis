# Trabajo final

## Análisis de Datos Científicos y Geográficos - Comisión: Ecom

> ### _Integrantes:_
>
> - Basilio, Claudio
> - Scornik, Carolina
> - Zini, Hernán

1. Descargamos los archivos de circuitos de [aquí](https://mapa2.electoral.gov.ar/descargas)
2. Descargamos los datos de la provincia del **CHACO** de [aquí](https://www.argentina.gob.ar/elecciones/resultados-del-recuento-provisional-de-las-elecciones-paso)
3. Descargamos el shape de Departamentos de la página del [Indec](https://datos.gob.ar/ar/dataset/jgm-servicio-normalizacion-datos-geograficos/archivo/jgm_8.16), y tomamos solo los polígonos de CHACO.
4. Pasamos todo a un servidor _Postgres_, y construimos una base de datos con las siguientes tablas: _Circuitos, Resultados, y Deptos_
5. Se unificaron los datos de circuitos para poder hacer los joins.
6. Creamos una nueva tabla con los totales de **votos positivos** para el cargo de **Diputados Provinciales** por partido, total general y porcentaje por partido por circuito y juntamos con circuitos para obtener la geometría.

> Para ello ejecutamos la siguiente consulta:

```sql
select * from "Circuitos" c
inner join
(
select v1.idCirc, V503, V501, V201, V71, VOtros
  ,(V503+V501+V201+V71+VOtros) total
	,(V503/(V503+V501+V201+V71+VOtros)*100) P503
	,(V501/(V503+V501+V201+V71+VOtros)*100) P501
	,(V201/(V503+V501+V201+V71+VOtros)*100) P201
	,(V71/(V503+V501+V201+V71+VOtros)*100) P71
	,(VOtros/(V503+V501+V201+V71+VOtros)*100) POtros
From
	(select r1."idcircuito" idCirc,
	SUM(cast(r1.votos as numeric)) V503 from resultados r1
	where r1."idagrupacion" = '503' and r1."tipovoto" = 'positivo' and r1."idcargo" = '6' -- 6 = DIPUTADO PROVINCIAL
	group by r1."idcircuito") as v1
inner join
	(select r1."idcircuito" idCirc,
	SUM(cast(r1.votos as numeric)) V501 from resultados r1
	where r1."idagrupacion" = '501' and r1."tipovoto" = 'positivo' and r1."idcargo" = '6'
	group by r1."idcircuito") as v2
	on v1.idCirc =  v2.idCirc
inner join
	(select r1."idcircuito" idCirc,
	SUM(cast(r1.votos as numeric)) V201 from resultados r1
	where r1."idagrupacion" = '201' and r1."tipovoto" = 'positivo' and r1."idcargo" = '6'
	group by r1."idcircuito") as v3
	on v1.idCirc =  v3.idCirc
inner join
	(select r1."idcircuito" idCirc,
	SUM(cast(r1.votos as numeric)) V71 from resultados r1
	where r1."idagrupacion" = '71' and r1."tipovoto" = 'positivo' and r1."idcargo" = '6'
	group by r1."idcircuito") as v4
	on v1.idCirc =  v4.idCirc
	inner join
	(select r1."idcircuito" idCirc,
	SUM(cast(r1.votos as numeric)) VOtros from resultados r1
	where r1."idagrupacion" not in('71','201','501','503') and r1."tipovoto" = 'positivo' and r1."idcargo" = '6'
	group by r1."idcircuito") as v5
	on v1.idCirc =  v5.idCirc
	) as DipCirc
	on c.circuito = DipCirc.idCirc
```

7. En _QGis_ usamos la funcion _"Base de datos"->"Administrador de Base de Datos"->"Ventana SQL"_ para crear una capa a partir de la consulta anterior.
   Luego, a partir de esta capa, generamos los mapas de los 4 principales partidos con los porcentajes por circuito.
   Usamos clasificación (para que genere un color por cada elemento) y seleccionamos graduación de colores según el color del partido.

- Lista 503 "Yo Cambio" -> amarillo
- Lista 501 "Frente de Todos" -> azul
- Lista 201 "Frente Integrador" -> rojo oscuro
- Lista 71 "Partido Obrero" -> verde

8. Unimos el shape de Departamentos con el de Circuitos y calculamos los totales y porcentajes de cada partido por departamento con la siguiente consulta:

```sql
select d.departamen,d.geom
	, (sum(dpc.v503)/ sum(dpc.total)*100) P503
	, (sum(dpc.v501)/ sum(dpc.total)*100) P501
	, (sum(dpc.v201)/ sum(dpc.total)*100) P201
	, (sum(dpc.v71)/ sum(dpc.total)*100) P71
from public."Deptos" d, public."DiputadosPorCircuito" dpc
where st_intersects(d.geom, ST_Centroid(dpc.geom))
group by d.departamen, d.geom ;
```

> Utilizamos la función [st_intersects](https://postgis.net/docs/ST_Intersects.html), para condicionar los resultados a los departamentos que intersecten con el centroide de cada circuito.

9. En _QGis_ repetimos paso 7 para crear una capa con los resultados obtenidos en el punto anterior, y luego generamos un gráfico por cada partido, con las mismas consideraciones de circuitos (colores y gradientes).

10. Obtenemos los porcentajes totales por partido, capturamos las imágenes de los mapas y armamos la presentación final con los resultaods. _Fin_
