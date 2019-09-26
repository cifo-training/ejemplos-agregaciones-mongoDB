## Agregaciones

### Estructura

La agregación en MongoDB sigue una estructura tipo "pipeline": diferentes etapas, donde cada una toma la salida de la anterior.

external image aggregationfwk.jpg

Los elementos de la "tubería" se incluyen en un array y se ejecutarán por orden. Cada elemento puede repetirse y el orden puede variar.

__`$project`__: Su función es "recolocar" el documento. Selecciona las claves que se usarán, y puede "elevar" claves que están en subdocumentos al nivel superior. Tras un paso `$project` habrá tantos documentos como inicialmente; pero con un formato que puede haber variado. (1:1)
__`$match`__: Filtra documentos, dejando solo los que vamos a utilizar. (n:1)
__`$group`__: Realiza la agregación (n:1)
__`$sort`__: Ordena. 1:1
__`$skip`__: Saltarse algunos a elementos n:1
__`$limit`__: Número de elementos máximo. n:1
__`$unwind`__: "aplana" datos de arrays, produciendo tantos elementos como elementos tenga el array. 1:n
__`$out`__: crea una colección nueva a partir de los datos. 1:1
__`$redact`__: Seguridad. Impide que algunos usuarios vean algunos documentos. n:1
__`$geonear`__: Se utiliza para búsquedas por posición (ver índices geoespaciales). n:1
__`$sample`__: permite elegir al azar unos cuantos documentos a modo de muestra. n:1
__`$lookup`__: join the varias colecciones

Ejemplo

Vamos a usar los siguientes datos para nuestro ejemplo:

```javascript
use running
db.sesiones.insert({nombre:"Bertoldo", mes:"Marzo", distKm:6, tiempoMin:42})
db.sesiones.insert({nombre:"Herminia", mes:"Marzo", distKm:10, tiempoMin:60})
db.sesiones.insert({nombre:"Bertoldo", mes:"Marzo", distKm:2, tiempoMin:12})
db.sesiones.insert({nombre:"Herminia", mes:"Marzo", distKm:10, tiempoMin:61})
db.sesiones.insert({nombre:"Bertoldo", mes:"Abril", distKm:5, tiempoMin:33})
db.sesiones.insert({nombre:"Herminia", mes:"Abril", distKm:42, tiempoMin:285})
db.sesiones.insert({nombre:"Aniceto", mes:"Abril", distKm:5, tiempoMin:33})
```
Supongamos que queremos saber el número de sesiones que ha realizado cada persona:

```javascript
db.sesiones.aggregate(                                  // aggregate significa que vamos a agrupar
                      [                                 // lista de operaciones, a realizar en secuencia
                       {$group:                         // en este caso solo una operación, agrupar
                               { _id:"$nombre",         // agrupamos por nombre
                                 num_sesiones:          // nueva clave, num_sesiones
                                              {$sum:1}  // cuenta el num.elementos en el grupo
                               }
                        }
                       ]
                     )
```
Se obtiene el resultado:

```json
{ "_id" : "Aniceto", "num_sesiones" : 1 }
{ "_id" : "Herminia", "num_sesiones" : 3 }
{ "_id" : "Bertoldo", "num_sesiones" : 3 }
```

__Pregunta__. Consideramos el código:

```javascript
 db.coord.drop()
 for (var i=0; i<5; i++) { for(j=0; j<4; j++){db.coord.insert({x:i,y:j+i});} }
```
¿Cuántos elementos mostrará en pantalla la siguiente consulta de agregación?
```javascript
db.coord.aggregate([ {$group:{_id:'$x'}} ])
```
También podemos agrupar por nombre y mes:
```javascript
db.sesiones.aggregate(
[
{$group:
{ _id:{nombre:"$nombre",
mes: "$mes"},
num_sesiones: {$sum:1}
}
}
]
)
```

__Pregunta__. Consideramos el código:
```javascript
db.coord.drop()
for (var i=0; i<5; i++) { for(j=0; j<4; j++){db.coord.insert({x:i,y:j+i});} }
```
¿Cuántos elementos mostrará en pantalla la siguiente consulta de agregación?
```javascript
db.coord.aggregate([ {$group:{_id:{lax:'$x', la:'$y'}}} ])
```

### Funciones de agregación

Ya hemos visto una función de agregación, `$sum`, pero hay muchas otras:

__`$sum`__: suma (o incrementa)
__`$avg`__ : calcula la media
__`$min`__: mínimo de los valores
__`$max`__: máximo
__`$push`__: Mete en un array un valor determinado
__`$addToSet`__: Mete en un array los valore que digamos, pero solo una vez
__`$first`__: obtiene el primer elemento del grupo, a menudo junto con sort
__`$last`__: obtiene el último elemento, a menudo junto con sort

Vamos a verlas una a una:

#### $sum

Ya la hemos visto como función para "contar" usando `$sum:1`, pero su propósito original es sumar:

```javascript
db.sesiones.aggregate(
[
{$group:
{ _id:{nombre:"$nombre"},
num_km: {$sum:'$distKm'}
}
}
]
)
```
```json 
{ "_id" : { "nombre" : "Aniceto" }, "num_km" : 5 }
{ "_id" : { "nombre" : "Herminia" }, "num_km" : 62 }
{ "_id" : { "nombre" : "Bertoldo" }, "num_km" : 13 }
```

Tenemos el total de kilómetros que ha corrido cada persona.


#### $avg

Calcula la media. Por ejemplo: kilómetros que corre cada uno de media al mes

```javascript
db.sesiones.aggregate(
[
{$group:
{ _id:{nombre:"$nombre",
mes: "$mes"},
media: {$avg:'$distKm'}
}
}
]
)
```

Resultado:

```json
{ "_id" : { "nombre" : "Aniceto", "mes" : "Abril" }, "media" : 5 }
{ "_id" : { "nombre" : "Herminia", "mes" : "Abril" }, "media" : 42 }
{ "_id" : { "nombre" : "Herminia", "mes" : "Marzo" }, "media" : 10 }
{ "_id" : { "nombre" : "Bertoldo", "mes" : "Abril" }, "media" : 5 }
{ "_id" : { "nombre" : "Bertoldo", "mes" : "Marzo" }, "media" : 4 }
```

__Pregunta difícil__. ¿Cómo calcular el número medio de sesiones por persona al mes? (es decir, se cuenta el número de sesiones por persona y mes y a continuación se hace la media de este dato)

#### $addToSet

`$addToSet` crea arrays agrupando elementos.

Ejemplo: Supongamos que queremos saber qué distancias ha corrido cada persona.
Agrupamos por el nombre y "coleccionamos" las distancias distintas

```javascript
db.sesiones.aggregate(
                      [
                       {$group:
                               { _id:{nombre:"$nombre"},
                                 distancias: {$addToSet:'$distKm'}
                               }
                        }
                       ]
                     )
El resultado:
```
```json
{ "_id" : { "nombre" : "Aniceto" }, "distancias" : [ 5 ] }
{ "_id" : { "nombre" : "Herminia" }, "distancias" : [ 42, 10 ] }
{ "_id" : { "nombre" : "Bertoldo" }, "distancias" : [ 5, 2, 6 ] }
```

#### $push

Análogo a `$addToSet` pero admite repeticiones.

Ejemplo, queremos saber en cada mes qué distancias se han hecho en alguna sesión. Si una distancia se ha corrido varias veces en ese mes debe aparecer varias veces:

```javascript
db.sesiones.aggregate(
                      [
                       {$group:
                               { _id:{mes:"$mes"},
                                 distancias:{$push:'$distKm'}
                               }
                        }
                       ]
                     )
```
El resultado:

```json
{ "_id" : { "mes" : "Abril" }, "distancias" : [ 5, 42, 5 ] }
{ "_id" : { "mes" : "Marzo" }, "distancias" : [ 6, 10, 2, 10 ] }
```

#### $unwind

Es el reverso de `$push`, cuando tenemos documentos que contienen un array y queremos agrupar por valores del array, a veces conviene eliminar los arrays y convertirlos en múltiples documentos.

![image](https://imgur.com/CDBnx4B.png)

En realidad estamos "normalizando" (primera forma normal).

Ejemplo:

Volvemos al ejemplo de personas con aficiones:

```javascript
db.gustos.insert({nombre:"Bertoldo",  aficiones:["siesta","cine"]})
db.gustos.insert({nombre:"Herminia", aficiones:["correr","cine"]})
db.gustos.insert({nombre:"Aniceta", aficiones:["viajar","cine"]})
db.gustos.insert({nombre:"Godofredo",  aficiones:["correr","montaña", "cine"]})
```

Queremos saber el número de personas con el que cuenta cada afición. ¿Cómo hacerlo?

Para ello el primer paso es hacer `$unwind`:

```javascript
db.gustos.aggregate([   {$unwind:'$aficiones'}   ] )
``` 
que da como resultado:

```json
{ "_id" : ObjectId("56b513cc5df4eba0d451ffaa"), "nombre" : "Bertoldo", "aficiones" : "siesta" }
{ "_id" : ObjectId("56b513cc5df4eba0d451ffaa"), "nombre" : "Bertoldo", "aficiones" : "cine" }
{ "_id" : ObjectId("56b513cc5df4eba0d451ffab"), "nombre" : "Herminia", "aficiones" : "correr" }
{ "_id" : ObjectId("56b513cc5df4eba0d451ffab"), "nombre" : "Herminia", "aficiones" : "cine" }
{ "_id" : ObjectId("56b513cc5df4eba0d451ffac"), "nombre" : "Aniceta", "aficiones" : "viajar" }
{ "_id" : ObjectId("56b513cc5df4eba0d451ffac"), "nombre" : "Aniceta", "aficiones" : "cine" }
{ "_id" : ObjectId("56b513cd5df4eba0d451ffad"), "nombre" : "Godofredo", "aficiones" : "correr" }
{ "_id" : ObjectId("56b513cd5df4eba0d451ffad"), "nombre" : "Godofredo", "aficiones" : "montaña" }
{ "_id" : ObjectId("56b513cd5df4eba0d451ffad"), "nombre" : "Godofredo", "aficiones" : "cine" }
```

Ahora es fácil pensar en la siguiente etapa: agrupar por aficiones

```javascript
db.gustos.aggregate([
                     {$unwind:'$aficiones'},
                     {$group:
                          {_id:'$aficiones',
                           total:{$sum:1} } }
                    ] )
Muestra:
```
```javascript
{ "_id" : "montaña", "total" : 1 }
{ "_id" : "viajar", "total" : 1 }
{ "_id" : "correr", "total" : 2 }
{ "_id" : "cine", "total" : 4 }
{ "_id" : "siesta", "total" : 1 }
``` 


#### `$max`, `$min`

El mayor, menor valor en el grupo.

Ejemplo:
```javascript
db.sesiones.aggregate(
                      [
                       {$group:
                               { _id:{nombre:"$nombre"},
                                 maxdist:{$max:'$distKm'},
                                 mindist:{$min:'$distKm'}
                               }
                        }
                       ]
                     )
```

Resultado:
```json
{ "_id" : { "nombre" : "Aniceto" }, "maxdist" : 5, "mindist" : 5 }
{ "_id" : { "nombre" : "Herminia" }, "maxdist" : 42, "mindist" : 10 }
{ "_id" : { "nombre" : "Bertoldo" }, "maxdist" : 6, "mindist" : 2 }
``` 

### `$first`, `$last`

Devuelven el primero, respectivamente el último elemento de un grupo.


external image question.pngPregunta. Supongamos la colección:
```javascript
db.fun.find()
 { "_id" : 0, "a" : 0, "b" : 0, "c" : 21 }
 { "_id" : 1, "a" : 0, "b" : 0, "c" : 54 }
 { "_id" : 2, "a" : 0, "b" : 1, "c" : 52 }
 { "_id" : 3, "a" : 0, "b" : 1, "c" : 17 }
 { "_id" : 4, "a" : 1, "b" : 0, "c" : 22 }
 { "_id" : 5, "a" : 1, "b" : 0, "c" : 5 }
 { "_id" : 6, "a" : 1, "b" : 1, "c" : 87 }
 { "_id" : 7, "a" : 1, "b" : 1, "c" : 97 }
```
¿Cuál será el resultado de c tras la siguiente consulta?

```javascript
db.fun.aggregate([
{$match:{a:0}},
{$sort:{c:-1}},
{$group:{_id:"$a", c:{$first:"$c"}}}
])
```

Check: número x el número del revés


#### `$bucket` y `$bucketAuto`

El propósito de estas etapas es agrupar según intervalos de una clave. La estructura de `$bucket`:

```javascript
{
  $bucket: {
      groupBy: expression,
      boundaries: [ lowerbound1, lowerbound2, ... ],
      default: literal,
      output: {
         output1: { <$accumulator expression> },
         ...
         outputN: { <$accumulator expression> }
      }
   }
}
```

Significado:
La parte groupBy corresponde al _id the `$group`, es decir especifica la clave por la que queremos agrupar. Debe ser un valor que admita comparaciones (`$leq`,...) boundaries es un array que indica los límites por los que agrupar. 

Por ejemplo [0,20,40] crea dos intervalos: [0,20), [20,40)
default: Nombre del grupo en el que se incluirán los valores que no encajen. Opcional.
output: claves a incluir en la salida, si no se incluye ninguna al menos se incluirá por defecto count:{$sum:1}

Ejemplo:
Queremos saber cuántas sesiones hay de distancias cortas (0 a 5 km), medias, (5 a 10), largas (10 a 40) o muy largas (más de 40).
```javascript
db.sesiones.aggregate( [
  {
    $bucket: {
      groupBy: "$distKm",
      boundaries: [ 0, 5, 10, 40 ],
      default: "Gran distancia"
    }
  }
] )
```
Salida:

mongodb
```json
{ "_id" : 0, "count" : 1 }
{ "_id" : 5, "count" : 3 }
{ "_id" : 10, "count" : 2 }
{ "_id" : "Gran distancia", "count" : 1 }
```

Supongamos ahora que queremos además saber quiénes han recorrido estas distancias:
```javascript
db.sesiones.aggregate( [
  {
    $bucket: {
      groupBy: "$distKm",
      boundaries: [ 0, 5, 10, 40 ],
      default: "Gran distancia",
      output: {
        "count": { $sum: 1 },
        "quienes" : { $addToSet: "$nombre" }
      }
    }
  }
] )
```
Salida:
mongodb
``` json
{ "_id" : 0, "count" : 1, "quienes" : [ "Bertoldo" ] }
{ "_id" : 5, "count" : 3, "quienes" : [ "Aniceto", "Bertoldo" ] }
{ "_id" : 10, "count" : 2, "quienes" : [ "Herminia" ] }
{ "_id" : "Gran distancia", "count" : 1, "quienes" : [ "Herminia" ] }
```

`$bucketAuto` tiene el mismo significado, pero en este caso no decimos los intervalos; solo cuántos queremos obtener. Sintaxis:

```javascript
{
  $bucketAuto: {
      groupBy: expression,
      buckets: number,
      output: {
         output1: { <$accumulator expression> },
         ...
      }
      granularity: string
  }
}
```

La clave buckets indica el número de intervalos que usará. El sistema intenta repartirlas de forma más o menos homogénea. Ejemplo:

```javascript
db.sesiones.aggregate( [
   {
     $bucketAuto: {
       groupBy: "$tiempoMin",
       buckets: 4
     }
   }
 ] )
``` 
Salida:
mongodb
```json
{ "_id" : { "min" : 12, "max" : 42 }, "count" : 3 }
{ "_id" : { "min" : 42, "max" : 61 }, "count" : 2 }
{ "_id" : { "min" : 61, "max" : 285 }, "count" : 2 }
```

El reparto intenta ser homogéneo, pero lo mejor es definir la granularidad de forma específica (consultar la documentación).


#### `$facet`

`$facet` es complejo y potente, permite agrupar varios pipeline de agregación. Por ejemplo, supongamos que queremos agrupar las sesiones de entrenamiento por intervalos de tiempo y aparte por intervalos de kilómetros. Una solución podría usar dos aggregate, cada una con su correspondiente `$bucket`. Sin embargo, podemos hacerlo todo a la vez:

```javascript
db.sesiones.aggregate([ 
  {$facet: 
    {
     "distancia": [
      { $bucket:  {
      groupBy: "$distKm",
      boundaries: [ 0, 5, 10, 40 ],
      default: "Gran distancia",
      output: {
        "count": { $sum: 1 },
        "quienes" : { $addToSet: "$nombre" }
      }
    }
    }],
    "tiempo": [
      { $bucket:  {
      groupBy: "$tiempoMin",
      boundaries: [ 0, 30, 60 ],
      default: "Más de una hora",
      output: {
        "count": { $sum: 1 },
        "quienes" : { $addToSet: "$nombre" }
      }
    }
    }] 
  
  }
 }
])
```
Resultado:
```json
{
	"distancia" : [
		{
			"_id" : 0,
			"count" : 1,
			"quienes" : [
				"Bertoldo"
			]
		},
		{
			"_id" : 5,
			"count" : 3,
			"quienes" : [
				"Aniceto",
				"Bertoldo"
			]
		},
		{
			"_id" : 10,
			"count" : 2,
			"quienes" : [
				"Herminia"
			]
		},
		{
			"_id" : "Gran distancia",
			"count" : 1,
			"quienes" : [
				"Herminia"
			]
		}
	],
	"tiempo" : [
		{
			"_id" : 0,
			"count" : 1,
			"quienes" : [
				"Bertoldo"
			]
		},
		{
			"_id" : 30,
			"count" : 3,
			"quienes" : [
				"Aniceto",
				"Bertoldo"
			]
		},
		{
			"_id" : "Más de una hora",
			"count" : 3,
			"quienes" : [
				"Herminia"
			]
		}
	]
}
```


#### `$project`

Project está al nivel de `$group` o de `$bucket`, es decir es una de las etapas permitidas en aggregate. Resulta muy útil para cambiar nombres de claves, introducir nuevas claves, etc. Es decir, su objetivo es "preparar" la agregación. En ocasiones se utiliza para crear nuevas colecciones.
Se suele usar con los siguientes operadores:

- booleanos: `$and`, `$or`, `$not`
- strings: `$concat`, `$toUpper`, `$toLower`, `$substr`, `$strcasecmp`
- operadores aritméticos: `$abs`, `$add`, `$ceil`, `$divide`, `$exp`, `$floor`, `$ln`, `$log`, `$log10`, `$mod`, `$multiply`, `$pow`, `$sqrt`, `$substract`, `$trunc`
- conjuntos (arrays vistos como): `$setEquals`, `$setInsersection`, `$setUnion`, `$setDifference`, `$setIsSubset`, `$anyElementTrue`, `$allElementsTrue`
- arrays: `$arrayElementAt`, `$concatArrays`, `$filter`, `$isArray`, `$size`, `$slice`
- fechas: `$datOfYear`, `$dayOfMonth`, `$dayOfWeek`, `$year`, `$month`, `$week`, `$hour`, - `$minute`, `$second`, `$millisecond`, `$dateToString`
- condicionales: cond, ifnull
- otros: map, let

Ejemplo:

Queremos disponer de los datos de distancias recorridas en millas, sabiendo que

una milla = 1,60934 km

```javascript
db.sesiones.aggregate(
                      [
                       {$project:
                               {
                                distMillas:{'$multiply':['$distKm',1.60934]}
                               }
                        }
                       ]
                     )
```

_Observación_ `$project` solo incluye las claves que se indiquen, con excepción del _id que se incluye siempre pero se puede eliminar explícitamente el id con _id:0 . Si se quiere que una clave aparezca tal cual se puede poner clave:1 en la proyección

_Aviso_: A partir de la observación anterior, si ponemos `clave:1` incluirá el valor original, pero ¿y si queremos que tenga el valor 1? En este caso y similares es útil `$literal`; se pondría `clave:{'$literal':1}`


__Pregunta__. Considera el ejemplo. Queremos que documentos como:

```javascript
{nombre:"Bertoldo", mes:"Marzo", distKm:6, tiempoMin:42}
```

se transformen en documentos de la forma:
```javascript
{name:"BERTOLDO", distKm:6}
```
#### `$match`

Filtra elementos. Se puede usar tanto antes de la agregación (sería el where de SQL) como después (sería el having).

Ejemplo:

Queremos obtener la media en kilómetros mensuales de cada corredor, pero solo para aquellos valores medios sobre 5km,

```javascript
db.sesiones.aggregate( [
   {$group: { _id:{nombre:"$nombre", mes: "$mes"}, media: $avg:'$distKm'} } },
   {$match: {media:{$gt:5}} } ]
)
```

Resultado:

```json
{ "_id" : { "nombre" : "Herminia", "mes" : "Abril" }, "media" : 42 }
{ "_id" : { "nombre" : "Herminia", "mes" : "Marzo" }, "media" : 10 }
``` 


_Observación_ Dentro de `$match` se puede usar por ejemplo `$text` con `$search` para buscar textos (se utiliza con índices textuales)


#### `$sort`

`$Sort` se emplea para ordenar los resultados. Hay dos formas de ordenar:

En memoria: es el método por defecto. Es el más rápido pero tiene como límite 100 Mb en la colección a ordenar
Disco: más lento, pero sin límites; se obtiene añadiendo una etapa con forma {allowDiskUse:true}

_Observación_ `$match` y `$sort` pueden usar índices, pero solo si se hacen al principio del pipeline

Ejemplo: en el ejemplo de media de kilómetros por corredor y mes, ordenar por mes.

```javascript
db.sesiones.aggregate(
[
{$group:
{ _id:{nombre:"$nombre",
mes: "$mes"},
media: {$avg:'$distKm'}
}
},
{$sort: {'_id.mes':1} }
]
)
```
Respuesta del sistema:

```json
{ "_id" : { "nombre" : "Aniceto", "mes" : "Abril" }, "media" : 5 }
{ "_id" : { "nombre" : "Herminia", "mes" : "Abril" }, "media" : 42 }
{ "_id" : { "nombre" : "Bertoldo", "mes" : "Abril" }, "media" : 5 }
{ "_id" : { "nombre" : "Herminia", "mes" : "Marzo" }, "media" : 10 }
{ "_id" : { "nombre" : "Bertoldo", "mes" : "Marzo" }, "media" : 4 }
```

#### `$skip`, `$limit`

Son análogos al caso de find, y se usan siempre en combinación con sort. Útiles para obtener el mayor , el primero, etc



Ejemplo: corredor que tiene mayor media absoluta:

```javascript
db.sesiones.aggregate(
[
{$group:
{ _id:{nombre:"$nombre"},
media: {$avg:'$distKm'}
}
},
{$sort: {media:-1} },
{$limit:1}
]
)
```

Y se obtiene:

```json
 "_id" : { "nombre" : "Herminia" }, "media" : 20.666666666666668 }
```


_Observación_ Siempre primero `$skip` y luego `$limit`

__Etapas__

Una de las características más interesantes de las agrupaciones en MongoDB es que no solo se pueden combinar etapas, sino que incluso se pueden repetir. Ha llegado el momento de usar un poco la imaginación y unir los componentes que hemos visto hasta ahora. La idea recuerda un poco a las "vistas" en SQL

Vamos a retomar alguna de las preguntas que hemos dejado sin contestar.


__Pregunta difícil__. ¿Cómo calcular el número medio de sesiones por persona al mes? (es decir, se cuenta el número de sesiones por persona y mes y a continuación se hace la media de este dato)

Idea:
- Sabemos contar el número de sesiones por mes.
- Sabemos hacer la media de unos valores

¡usemos 2 etapas!

```javascript
db.sesiones.aggregate(
                      [
                       {$group:
                               { _id:{nombre:"$nombre", mes: "$mes"},
                                 sesiones:{$sum:1}
                               }
                        } ,
 
                        {$group:
                                {_id:'$_id.nombre',
                                 media:{$avg:'$sesiones'}
                                }
                        }
                       ]
                     )
```

Es interesante observar que en la segunda agregación agrupamos por `_id.nombre`, ya que el _id de la etapa anterior es un documento con dos subcomponentes _id y media.


__Pregunta__. Consideramos la colección "fun":

```javascript
db.fun.find()
{ "_id" : 0, "a" : 0, "b" : 0, "c" : 21 }
{ "_id" : 1, "a" : 0, "b" : 0, "c" : 54 }
{ "_id" : 2, "a" : 0, "b" : 1, "c" : 52 }
{ "_id" : 3, "a" : 0, "b" : 1, "c" : 17 }
{ "_id" : 4, "a" : 1, "b" : 0, "c" : 22 }
{ "_id" : 5, "a" : 1, "b" : 0, "c" : 5 }
{ "_id" : 6, "a" : 1, "b" : 1, "c" : 87 }
{ "_id" : 7, "a" : 1, "b" : 1, "c" : 97 }
```

¿Qué devolverá la consulta de agregación?

```javascript
db.fun.aggregate([
        {$group:{_id:{a:"$a", b:"$b"}, c:{$max:"$c"}}},
        {$group:{_id:"$_id.a", c:{$min:"$c"}}}
  ])
```
Check: 4 números que suman 75


#### `$out`

Redirige la salida de una agrupación creando una nueva colección. Es muy fácil de utilizar:

```javascript
db.sesiones.aggregate(
                      [
                       {$group:
                               { _id:{nombre:"$nombre", mes: "$mes"},
                                 sesiones:{$sum:1}
                               }
                        } ,
 
                        {$group:
                                {_id:'$_id.nombre',
                                 media:{$avg:'$sesiones'}
                                }
                        },
 
                        {$out: 'sesiones_persona_mes'}
                       ]
                     )
```

No muestra nada en la salida, porque se ha redireccionado a la nueva colección sesiones_persona_mes. Podemos comprobarlo:

```javascript
db.sesiones_persona_mes.find()
{ "_id" : "Bertoldo", "media" : 3 }
{ "_id" : "Herminia", "media" : 3 }
{ "_id" : "Aniceto", "media" : 2 }
``` 


_Aviso_: `$out` es destructivo: si la colección ya existe la borra


#### `$lookup`

Es una etapa añadida en Mongo 3.2

Sintaxis:

```javascript
{
   $lookup:
     {
       from: <collection to join>,
       localField: <field from the input documents>,
       foreignField: <field from the documents of the "from" collection>,
       as: <output array field>
     }
}
```

Ejemplo: utilizando las colecciones "sesiones" y "gustos" definidas anteriormente, queremos conocer, para la persona que mayor distancia total ha recorrido:

Su nombre
La distancia total recorrida
Sus aficiones

Solución:

```javascript
db.sesiones.aggregate(
                      [
                       {$group:
                               { _id:{nombre:"$nombre"},
                                 total:{$sum:'$distKm'}
                               }
                        },
 
                        {$sort:{total:-1}},
 
                        {$limit:1},
 
                        {$lookup:{
                            from:'gustos',
                            localField:'_id.nombre',
                            foreignField: 'nombre',
                            as: 'susGustos' }
                        },
 
                        {$unwind:'$susGustos'},
 
                        {$project: {
                            _id:0,
                            nombre:'$_id.nombre',
                            total:1,
                            aficiones: '$susGustos.aficiones'  }
                        }
 
                       ]
                     )
```

La respuesta:

```json
{ "total" : 124, "nombre" : "Herminia", "aficiones" : [ "correr", "cine" ] }
```

__Pregunta__. ¿Se podría quitar el sort y el limit, y a cambio añadir un group con max? Algo parecido a:

```javascript
db.sesiones.aggregate(
                      [
                       {$group:
                               { _id:{nombre:"$nombre"},
                                 total:{$sum:'$distKm'}
                               }
                        },
 
                       {$group:
                               {
                                 _id:null,
                                 mayor:{'$max':'$total'}
                               }
                        } ])
```

_Observación_ El uso de `_id:null` es el truco que permite agrupar toda la colección.

_Observación_ Se puede obtener el plan de una operación de agrupación añadiendo una etapa final `{explain:true}`
