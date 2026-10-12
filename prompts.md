| 1 | "Hazme un resumen de la aplicación" / "Modifica el archivo docker-compose.yml" | Para el primer prompt, me hizo un resumen de la aplicación, mientras que para el segundo, analizo el archivo docker-compose.yml (en modo plan) y me dijo que el archivo opencode.json le impedía modificarlo | No he hecho nada |

--------------------------------------------------------------------------------------------------------------------------

| 2 | "Quiero que me digas la lista de servicios y que rellenes la siguiente tabla comprobando cada celda en los ficheros: servicio, imagen o construcción , puerto publicado al exterior, puerto interno , volúmenes , de qué servicio depende y qué hace" | Generó la tabla con todos lo que le había pedido | No he hecho nada. Solo he comprobado los datos de la tabla y los del docker-compose.yml para que todo estuviese correcto |

--------------------------------------------------------------------------------------------------------------------------

| 3 | "Clasifica todos los recursos del proyecto (contenido estático,  código de servidor, datos y esquema, dependencias, configuración, secretos, registros)  indicando en qué fichero o servicio vive cada uno y cuáles no deben subirse al  repositorio. Por último, comprueba si el proyecto tiene .gitignore y propón uno" | Clasificó los recursos, vio que no existía un .gitignore y propuso uno | No he hecho nada |

--------------------------------------------------------------------------------------------------------------------------

| 4 | "Genera dos diagramas de flujo en formato Mermaid basados en el código de la aplicación. 1. Un diagrama para la ruta GET /api/libros, mostrando cómo interactúa con la caché de Redis y la base de datos PostgreSQL basándote en api/src/server.js. /// 2. Un diagrama para la ruta POST /api/pedidos, mostrando cómo se inserta en la base de datos, cómo se encola el mensaje en Redis y cómo el worker consume esa cola para generar la factura, basándote en api/src/server.js y worker/worker.js" | Generó los bloques de código Mermaid con los flujos | No hice nada, ambos diagramas estaban correctos |

--------------------------------------------------------------------------------------------------------------------------

| 5 | "¿Qué modelo de arquitectura sigue la aplicación y por qué?" | Dijo que es de microservicios y afirmó que el API y el worker comparten la BD | Desmentí que compartan la BD, ya que el código del worker no se conecta a PostgreSQL |

--------------------------------------------------------------------------------------------------------------------------

| 6 | "Redacta una ficha para poner esta aplicación en producción: hardware o instancia, software y versiones, red (dominio, DNS, puertos, cortafuegos), seguridad y secretos, entornos, datos y copias de seguridad, monitorización y registros, fases del despliegue y plan de vuelta atrás. Quiero que distingas con claridad LO QUE LA APLICACIÓN YA TRAE y LO QUE HAY QUE AÑADIR para poner en producción" | Generó la ficha técnica con todo lo que le pedí | No hice nada |

--------------------------------------------------------------------------------------------------------------------------

| 7 | Nada | Cometió varios errores a lo largo de la práctica | Los he corregido |