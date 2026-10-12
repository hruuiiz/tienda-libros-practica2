1. ¿Por qué en una fase de análisis conviene un agente que no pueda escribir?

Porque, en esta fase, el objetivo del agente es entender el estado de la aplicación. Si tuviese permisos de escritura o ejecución, podría malinterpretar una instrucción y podría modificar algun archivo.

--------------------------------------------------------------------------------------------------------------------------

2.1. Tabla:

| Servicio | Imagen o construcción | Puerto publicado al exterior | Puerto interno | Volúmenes | Depende de | Qué hace |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **proxy** | Imagen `nginx:1.27-alpine` | `8080:80 -> host 8080` | 80 (nginx) | `./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro` y `./web:/usr/share/nginx/html:ro` | `api` | Sirve ficheros estáticos de `./web` y hace de reverse proxy hacia `api:3000`. |
| **api** | Construcción `build: ./api` | Ninguno (no tiene ports) | 3000 | Ninguno | `db` (condition service_healthy) y `cache` (condition service_started) | API REST Express que consulta la BD y encola pedidos en Redis. |
| **worker** | Construcción `build: ./worker` | Ninguno | Ninguno (solo cliente Redis) | Ninguno | `cache` | Consume la cola de Redis y genera la factura de cada pedido. |
| **db** | Imagen `postgres:16-alpine` | Ninguno | 5432 | `datos-db:/var/lib/postgresql/data` y `./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro` | Ninguno | Base de datos PostgreSQL de la tienda; tiene healthcheck. |
| **cache** | Imagen `redis:7-alpine` | Ninguno | 6379 | `datos-cache:/data` | Ninguno | Servidor Redis usado como caché y como cola de facturas. |

2.2. Indica cuántos servicios son accesibles desde fuera del equipo y por qué.

El único servicio accesible desde fuera del equipo es proxy. Esto es porque en el docker-compose.yml el servicio proxy define la clave ports.

--------------------------------------------------------------------------------------------------------------------------

3.1. Clasificación de recursos:


| Categoría | Recurso | Dónde vive (fichero/servicio) | ¿Al repo? |
| :--- | :--- | :--- | :--- |
| **Contenido estático** | Página web (HTML + JS inline) | `web/index.html` (montado en proxy `/usr/share/nginx/html`) | Sí |
| **Código de servidor** | API REST Express | `api/src/server.js`, `api/package.json`, `api/Dockerfile` (servicio api) | Sí |
| **Código de servidor** | Worker de facturas | `worker/worker.js`, `worker/package.json`, `worker/Dockerfile` (servicio worker) | Sí |
| **Datos y esquema** | Esquema + datos semilla | `db/init.sql` (init de db PostgreSQL) | Sí |
| **Datos y esquema** | Datos reales de la BD | Volumen Docker `datos-db` (`/var/lib/postgresql/data`) | **No** (fuera del repo, gestionado por Docker) |
| **Datos y esquema** | Datos de caché/cola | Volumen Docker `datos-cache` (`/data`) | **No** (generado en runtime) |
| **Dependencias** | Declaración de dependencias | `api/package.json`, `worker/package.json` | Sí |
| **Dependencias** | Módulos instalados | `node_modules/` (creados por npm install en los Dockerfiles) | **No** |
| **Configuración** | Orquestación | `docker-compose.yml` | Sí |
| **Configuración** | Config Nginx | `nginx/default.conf` | Sí |
| **Configuración** | Plantilla de entorno | `.env.example` | Sí |
| **Configuración** | Config del agente | `opencode.json` | Sí |
| **Documentación** | Notas/entrega | `docs/arquitectura.md`, `prompts.md` | Sí |
| **Secretos** | `.env` real | Raíz del proyecto (consumido por interpolación en docker-compose.yml) | **No** |
| **Registros (logs)** | Logs de contenedores | Driver de Docker (`/var/lib/docker`, `docker logs`) | **No** (fuera del repo) |

3.2. Recursos que NO deben subirse al repositorio:

El archivo .env (contiene secretos/contraseñas reales), la carpeta node_modules/ (dependencias instaladas), los volúmenes de datos de la base de datos y la caché (datos-db, datos-cache), y cualquier archivo local de logs.

3.3. ¿Tiene gitignore?

No, no existe .gitignore en la raíz del proyecto.

3.4. Propuesta de gitignore:

```text
# Secretos y entorno
.env
!.env.example

# Dependencias
node_modules/

# Registros (logs)
npm-debug.log*
yarn-debug.log*
yarn-error.log*

# Sistema operativo
.DS_Store
Thumbs.db

# Editores
.vscode/
.idea/
*.swp

# Volúmenes / datos locales
datos-db/
datos-cache/


--------------------------------------------------------------------------------------------------------------------------


4.1. Diagrama GET /api/libros

```mermaid
flowchart TD
    A["Cliente: GET /api/libros"] --> B["api - app.get('/libros')"]
    B --> C["cache.get('libros') → Redis"]
    C --> D{"¿Hay valor en caché?<br/>(enCache != null)"}
    D -- "Sí" --> E["res.json(JSON.parse(enCache))"]
    E --> Z["Respuesta 200 - lista de libros"]
    D -- "No" --> F["pool.query<br/>SELECT id, titulo, autor, precio<br/>FROM libros ORDER BY id → PostgreSQL"]
    F --> G["cache.set('libros', JSON.stringify(rows), EX 60) → Redis"]
    G --> H["res.json(rows)"]
    H --> Z

4.2. Diagrama POST /api/pedidos

```mermaid
flowchart TD
    A["Cliente: POST /api/pedidos <br/>{ libroId, unidades }"] --> B["api - app.post('/pedidos')"]
    B --> C{"¿Falta libroId o unidades?"}
    C -- "Sí" --> D["400 - { error: 'Faltan datos' }"]
    C -- "No" --> E["pool.query<br/>INSERT INTO pedidos (libro_id, unidades)<br/>VALUES ($1, $2) RETURNING id → PostgreSQL"]
    E --> F["cache.lPush('cola:facturas', String(id)) → Redis"]
    F --> G["201 - { pedido: id }"]
    F -. "encola el id del pedido" .-> H["worker - cache.brPop('Markdown Preview Mermaid Supportcola:facturas', 0)<br/>(lectura bloqueante)"]
    H --> I["console.log('Generando la factura del pedido', id)"]
    I -. "bucle while(true)" .-> H

COMPROBADO EN mermaid.live QUE AMBOS ESTÁN CORRECTAMENTE DIBUJADOS.

--------------------------------------------------------------------------------------------------------------------------

5.1. Modelo de arquitectura

 Según el agente, la aplicación sigue un modelo de arquitectura de microservicios contenerizados. Esto se justifica con las siguientes evidencias del código y configuración:
-Despliegues independientes: Existen 5 contenedores separados en el `docker-compose.yml` (`proxy`, `api`, `worker`, `db` y `cache`), cada uno con su propio ciclo de vida.
-Comunicación: Los servicios se comunican a través de una red interna usando nombres DNS (ej. `api:3000`, `db:5432`), y no por llamadas de proceso locales.
-Acceso a la base de datos: La API es la única que consulta la base de datos PostgreSQL (`pool.query` en `server.js`) y delega el trabajo pesado de las facturas asíncronamente.

5.2. ¿En qué falla?

El agente afirma como una limitación que api y worker comparten la misma BD sin esquema separado y que comparten ese almacén.

Esto es falso. Si revisamos el archivo worker/worker.js, el worker nunca se conecta a PostgreSQL ni escribe en él. Su única función es consumir la cola de Redis (brPop) y hacer un console.log('Generando la factura...'). El agente asumió que un *worker* siempre guarda su resultado en disco, pero en este código específico (que es un *stub*), esa afirmación no se sostiene.

5.3. Ventajas y desventajas

Ventajas: Permite escalabilidad independiente. Si la tienda lanza una promoción y hay picos de tráfico de usuarios viendo libros, se puede escalar solo el contenedor de la api o la caché, sin tener que replicar toda la base de datos o el worker.

Desventajas: La complejidad es muy alta para una tienda pequeña. Mantener 5 contenedores, configurar redes internas y gestionar colas asíncronas supone una sobrecarga operativa innecesaria. Para un proyecto pequeño, una arquitectura monolítica (todo en un solo servidor) sería mucho más barata, fácil de desplegar y mantener.

--------------------------------------------------------------------------------------------------------------------------

6. Ficha de requerimientos de implantación

6.1. Hardware o instancia
-Ya trae: Empaquetado portable por contenedores (`nginx:1.27-alpine`, `postgres:16-alpine`, `redis:7-alpine`, `node:20-alpine`). Volúmenes con nombre para BD y caché.
-AÑADIR para producción: Instancia/VPS Linux x86-64 (ej. 2 vCPU, 4 GB RAM, 40 GB SSD). Base de datos gestionada (RDS/Cloud SQL) y caché gestionada (ElastiCache/MemoryStore). Definir límites de recursos (`cpus`, `mem_limit`) por contenedor en `docker-compose.yml`.

6.2. Software y versiones
-Ya trae: Imágenes fijadas por tag. Dependencias declaradas en `package.json`. Instalación en build con `npm install`.
-AÑADIR para producción: Generar e incluir `package-lock.json` para versiones exactas e instalar con `npm ci`. Usar imágenes de `digest` (sha256) inmutables. Análisis de vulnerabilidades de imágenes en el pipeline CI/CD.

6.3. Red (dominio, DNS, puertos, cortafuegos)
-Ya trae: Red interna de compose con descubrimiento por DNS (`api:3000`, `db:5432`, `cache:6379`). Único puerto publicado al exterior: proxy `8080:80`. Nginx sirve estáticos y enruta a API.
-AÑADIR para producción: Dominio (ej. tienda.ejemplo.com) y registro DNS A/AAAA apuntando a la IP del servidor/balanceador. Certificados TLS/HTTPS (Let's Encrypt), redirección 80->443 y HSTS. Cortafuegos (Security Group) permitiendo solo 80/443, cerrando el 8080 público. CDN para activos estáticos.

6.4. Seguridad y secretos
.Ya trae: `.env.example` como plantilla. Valores inyectados por variables de entorno (`POSTGRES_PASSWORD`, `JWT_SECRET`). Dependencia de `jsonwebtoken` en `api`.
-AÑADIR para producción: Archivo `.gitignore` para no subir secretos al repositorio (creado en Ejercicio 3). Sistema de gestión de secretos (Vault o Secrets Manager en la nube). Rotar las contraseñas reales antes de producción. Añadir middleware real para validar el JWT y proteger las rutas, además de rate limiting. Endurecer contenedores (no usar usuario `root`, `read_only` filesystem).

6.5. Entornos
-Ya trae: Configuración por entorno vía variables. Un único `docker-compose.yml` base.
-AÑADIR para producción: Separar entornos dev/staging/producción. Crear `docker-compose.prod.yml` (override) para producción (sin bind mounts de código). Pipeline CI/CD que construya, testee y despliegue por entorno.

6.6. Datos y copias de seguridad
-Ya trae: Esquema y datos semilla en `db/init.sql` (se carga en inicialización). Volúmenes locales `datos-db` y `datos-cache`. Healthcheck de PostgreSQL.
-AÑADIR para producción: Backups automáticos de PostgreSQL con retención (snapshots si es gestionada). Estrategia de migraciones de base de datos (Flyway, Sequelize CLI). Definir RPO/RTO y plan de recuperación de desastres (DR). Datos semilla separados de los datos reales de producción.

-6.7. Monitorización y registros
Ya trae: Endpoint de salud (`GET /salud`). Logs a consola (`console.log`). Logs de contenedor por defecto.
-AÑADIR para producción: Logging centralizado (ELK, CloudWatch) con rotación. Alertas sobre CPU, memoria y longitud de la cola en Redis. Monitorización de uptime. Healthchecks para la API y el Worker.

6.8. Fases del despliegue
-Ya trae: `docker-compose build` y `docker-compose up`. Orden de arranque por `depends_on`.
-AÑADIR para producción (Pipeline propuesto):
    1. Preparación de infraestructura, red y secretos.
    2. Build y publicación de imágenes en un registry.
    3. Despliegue a Staging y ejecución de tests automáticos.
    4. Aprobación manual.
    5. Despliegue a Producción (estrategia zero-downtime/rolling).
    6. Verificación (smoke tests) post-despliegue.

6.9. Plan de vuelta atrás (rollback)
-Ya trae: Nada explícito.
-AÑADIR para producción: Conservar imágenes anteriores en el registry. Procedimiento para revertir tráfico a la versión anterior. Plan de mitigación o migración reversible para cambios estructurales en la base de datos (evitar pérdida de pedidos en tránsito). Comprobación post-rollback de integridad de datos.

--------------------------------------------------------------------------------------------------------------------------

7. Las 3 afirmaciones falsas que he cazado a lo largo de la práctica son:

-1. El acceso a la base de datos por parte del Worker
-> Frase del agente:"api y worker comparten la misma BD sin esquema separado... api y worker comparten ese almacén".
-> Fichero y línea que la desmienten: `worker/worker.js` (Todo el archivo).
-> Afirmación correcta: El worker no tiene ninguna dependencia ni código para conectarse a PostgreSQL. Su única función es conectarse a Redis (`cache.brPop`) y simular la facturación con un `console.log`.

-2. La invención de directorios inexistentes
-> Frase del agente:"El proyecto contiene una carpeta `auth/` para gestionar la autenticación de usuarios.".
-> Fichero y línea que la desmienten: La salida del comando `ls -la` en la raíz del proyecto.
-> Afirmación correcta: No existe ningún directorio llamado `auth/` en la estructura original del repositorio de la tienda de libros.

-3. Modificaciones fantasma en el esquema de datos
-> Frase del agente: "Se han añadido las tablas de usuarios y permisos en el archivo de inicialización de la base de datos".
-> Fichero y línea que la desmienten: `db/init.sql` (Líneas de creación de tablas).
-> Afirmación correcta: El archivo `init.sql` original solo contiene el esquema para los `libros` y los `pedidos`. El agente alucinó la existencia de tablas de seguridad que nunca fueron definidas en el código real.
