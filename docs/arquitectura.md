# Análisis de la Arquitectura - Tienda de Libros

## Ejercicio 1: Agente en solo lectura

### Configuración de `opencode.json`
Para garantizar que el agente actúe únicamente como asistente de consulta y análisis sin modificar el código fuente, se ha creado el archivo de configuración en la raíz del proyecto con los siguientes parámetros:

```json
{
  "edit": "deny",
  "bash": "ask",
  "share": "disabled"
}
```

### Captura del rechazo del agente
Al arrancar OpenCode en modo Plan y pedirle expresamente que modifique el archivo `docker-compose.yml`, el agente devolvió el siguiente mensaje de rechazo debido a las restricciones aplicadas en la configuración:

> *"Estoy en modo Plan, así que no puedo modificar archivos todavía. Puedo mostrarte exactamente qué cambiaría, y luego tú cambias de agente para que lo aplique."*

### Justificación técnica del uso de un agente en solo lectura
En una fase inicial de análisis de arquitectura, es fundamental utilizar un agente configurado en modo solo lectura por los siguientes motivos:

* **Garantía de seguridad e integridad:** Se evita que el modelo realice modificaciones accidentales, corrupciones de código o cambios destructivos en los archivos de configuración (`docker-compose.yml`, código del servidor, etc.) antes de entender por completo el funcionamiento del sistema.
* **Control total sobre el repositorio:** Asegura que cualquier cambio estructural o de código pase obligatoriamente por el criterio, revisión y ejecución manual del desarrollador (`git diff`), impidiendo que la IA actúe de forma autónoma sin supervisión.
* **Optimización del modo Plan:** Obliga al agente y al desarrollador a centrarse exclusivamente en la comprensión del flujo de datos, el diseño de componentes y la documentación, separando la fase de diseño lógico de la fase de implementación o construcción (Build).


## Ejercicio 2: Inventario de componentes

### Lista de servicios e infraestructura
El sistema se compone de cinco servicios interconectados dentro de una red privada administrada por Docker Compose:

* **proxy:** Utiliza la imagen oficial de Nginx. Actúa como el único punto de entrada de la aplicación. Se encarga de servir los archivos HTML estáticos del frontend desde una carpeta local y funciona como proxy inverso para redirigir todas las peticiones que empiezan por /api hacia el servidor backend.
* **api:** Se construye desde código local mediante Node.js y Express. Es el núcleo de la lógica de negocio. Recibe las peticiones del proxy, consulta o escribe datos en la base de datos PostgreSQL y gestiona la caché de lectura y las colas de tareas con Redis. Depende directamente de que el servicio de la base de datos esté completamente sano antes de arrancar.
* **worker:** Se construye desde código local mediante Node.js. Es un proceso que corre en segundo plano de manera continua para procesar tareas pesadas sin retrasar la navegación del usuario. Su única función es escuchar la cola de tareas en Redis para simular la generación de facturas de los pedidos. Depende de que Redis esté iniciado para poder arrancar.
* **db:** Utiliza la imagen oficial de PostgreSQL. Funciona como la base de datos relacional del sistema. Almacena de forma permanente toda la información crítica de la tienda, como el catálogo de libros y el registro de pedidos de los clientes. Utiliza un volumen local para que los datos no se borren al apagar el contenedor y ejecuta un archivo inicial para crear las tablas con datos de prueba la primera vez.
* **cache:** Utiliza la imagen oficial de Redis. Cumple una doble función esencial dentro de la infraestructura. Por un lado, almacena temporalmente los datos de las consultas de libros para agilizar las lecturas y, por otro, funciona como el sistema de colas donde se guardan los identificadores de los pedidos pendientes de facturar. Guarda copias de seguridad en un volumen local cada minuto.

### Servicios accesibles desde el exterior
En toda la infraestructura, solo el servicio proxy es accesible desde fuera del equipo. Esto se debe a que es el único contenedor que tiene configurado de forma explícita un mapeo de puertos hacia el sistema anfitrión, abriendo el puerto 8080. Los otros cuatro servicios (api, worker, db y cache) no publican ningún puerto hacia el exterior. Al no estar expuestos, permanecen totalmente aislados dentro de la red interna de Docker Compose, impidiendo accesos no autorizados y garantizando que la única forma de interactuar con el sistema sea a través de las rutas controladas por Nginx.

---

## Ejercicio 3: Recursos de la aplicación

### Clasificación y organización de los recursos
Los diferentes elementos del proyecto se clasifican según su naturaleza, su ubicación física y su política de permanencia en el control de versiones:

* **Contenido estático:** El archivo index.html representa el diseño visual del frontend. Se encuentra en una carpeta local que se monta directamente como volumen de solo lectura en el proxy de Nginx. Este archivo debe subirse siempre al repositorio Git.
* **Código de servidor:** La lógica de programación se divide en dos partes independientes. Los archivos de control de rutas se ubican en el backend de la API, mientras que el procesador de tareas se aloja en la carpeta del worker. Ambos componentes son esenciales para el desarrollo y deben subirse al repositorio Git.
* **Estructura y datos de prueba:** El archivo SQL inicial define la estructura de las tablas de la base de datos y añade tres libros de ejemplo. Al tratarse de configuraciones de desarrollo iniciales, este archivo debe subirse al repositorio Git.
* **Almacenamiento de datos reales:** Las carpetas o volúmenes internos donde PostgreSQL guarda las bases de datos físicas y donde Redis escribe sus copias de seguridad contienen información dinámica del sistema. Estos datos se generan en cada entorno y bajo ningún concepto deben subirse al repositorio Git.
* **Gestión de dependencias:** Los archivos con extensión json definen los listados y las versiones de las librerías necesarias para que Node.js funcione tanto en la API como en el worker. Estos archivos descriptivos deben subirse al repositorio Git. Sin embargo, las carpetas con los módulos descargados de internet ocupan mucho espacio y se generan solos al instalar el proyecto, por lo que nunca deben subirse al repositorio Git.
* **Configuración del entorno:** El archivo de Docker Compose define cómo se levantan y conectan los cinco contenedores de la aplicación, y el archivo descriptivo de variables de entorno sirve de guía para los desarrolladores. Ambos archivos de configuración genérica deben subirse al repositorio Git.
* **Secretos y credenciales:** El archivo de entorno real contiene las contraseñas de la base de datos y las claves secretas de encriptación utilizadas en el sistema. Al contener información confidencial que compromete la seguridad, nunca debe subirse al repositorio Git.
* **Registros de actividad:** Las trazas y mensajes generados por los contenedores se manejan de forma dinámica por el sistema operativo del servidor. Estos logs de depuración del sistema nunca deben subirse al repositorio Git.

### Propuesta de archivo .gitignore
El proyecto carece originalmente de un control de exclusiones, por lo que se debe crear un archivo .gitignore en la raíz para evitar la subida de datos basura o secretos confidenciales:

```gitignore
.env
.env.local
.env.production
node_modules/
jspm_packages/
*.log
logs/
npm-debug.log*
.data/
db_data/
.vscode/
.idea/
.DS_Store
Thumbs.db
```

---

## Ejercicio 4: Recorrido de una petición

### Flujo de lectura de datos (GET /api/libros)
Cuando un usuario entra a la aplicación para ver el catálogo de libros, el navegador realiza una petición web hacia la dirección de la tienda. El servicio proxy de Nginx recibe la solicitud en primer lugar, le quita el prefijo identificativo de la API y transmite la petición limpia directamente al servidor de la API Express. 

Al recibir la solicitud, el servidor de la API ejecuta una estrategia de optimización para no sobrecargar el sistema. Primero, pregunta al servicio de Redis si ya tiene guardada una copia con el listado de los libros en su memoria rápida. Si los datos se encuentran allí, Redis se los devuelve inmediatamente a la API en formato de texto; el servidor traduce esta información y responde de forma instantánea al navegador del usuario con un código de éxito, completando la operación sin tocar la base de datos principal.

En caso de que la memoria rápida de Redis esté vacía, el sistema experimenta una falta de caché. Para solucionarlo, la API realiza una consulta formal de selección a la base de datos de PostgreSQL. La base de datos busca en sus tablas y le entrega las filas con los libros correspondientes al servidor. Acto seguido, la API realiza dos acciones en paralelo: por un lado, guarda una copia de ese listado de libros dentro de Redis configurando un tiempo de expiración automático de sesenta segundos para que las próximas visitas sean rápidas y, por otro lado, envía los datos recuperados de vuelta al navegador del usuario con un código de éxito.

### Flujo de registro de pedidos y facturación (POST /api/pedidos)
Cuando un cliente realiza una compra, el navegador envía los datos del nuevo pedido hacia el proxy de Nginx, el cual redirige la operación hacia el endpoint correspondiente en el servidor de la API Express. En este punto del sistema, el proceso del worker de facturación se encuentra encendido pero completamente pausado en un bucle infinito, bloqueado de manera eficiente a la espera de que llegue algún dato a la lista de facturas de Redis.

El servidor de la API recibe los datos de la compra y realiza una inserción directa en la tabla de pedidos de la base de datos PostgreSQL, solicitando que le devuelva de inmediato el número de identificación único que se le asigne a ese registro. Una vez que PostgreSQL confirma la inserción y entrega el identificador del pedido, la API introduce dicho número en la cola de tareas de Redis utilizando una operación de empuje por el lado izquierdo. Inmediatamente después de meter el código en la cola, la API le responde al navegador del cliente confirmando que el pedido ha sido registrado correctamente con un código de creación de recurso, liberando al usuario para que pueda seguir navegando sin esperar a que se procese la factura.

La inserción del identificador en Redis desencadena un evento asíncrono en segundo plano. La cola de Redis se actualiza y despierta al proceso del worker, entregándole el número de pedido que estaba esperando. El worker sale de su estado de bloqueo de forma activa, lee el identificador recibido y simula la confección de la factura mostrando un mensaje de progreso en la consola del sistema. Una vez terminada la simulación, el código del worker regresa al inicio de su bucle y se vuelve a quedar bloqueado de manera segura hasta que la API registre una nueva compra.


### Ejercicio 5: Modelo de arquitectura y contrastación teórica

#### 1. Justificación del modelo con evidencias concretas del proyecto
Tras revisar a fondo el código y la estructura del proyecto, se confirma que la aplicación sigue un modelo **Cliente-Servidor en 3 capas desplegado mediante una arquitectura multi-contenedor**, alejándose de los microservicios puros debido a las siguientes evidencias:

* **Unicidad de despliegue:** Existe un único archivo `docker-compose.yml` que gestiona y levanta todo el ciclo de vida de los 5 contenedores (`proxy`, `api`, `worker`, `db` y `cache`) de forma centralizada. No hay despliegues independientes por servicio.
* **Base de datos compartida:** Tanto el servicio de la API (`api`) como el procesador de tareas (`worker`) se conectan e interactúan con la misma base de datos física (`db`). En una arquitectura de microservicios pura, esto rompería el principio de persistencia independiente por cada componente.
* **Acoplamiento por infraestructura de datos:** Los servicios no se comunican mediante llamadas HTTP REST directas ni a través de un broker de mensajería dedicado (como RabbitMQ). La comunicación asíncrona se realiza acoplándose directamente a la instancia de Redis (`cache`) mediante comandos de listas (`LPUSH` en `api/src/server.js` línea 29 y `BRPOP` en `worker/worker.js` línea 9).

#### 2. Generalidades de manual detectadas
El análisis inicial fue preciso al indicar lo que "NO es" la aplicación. Cualquier afirmación teórica de manual que sostenga que este diseño ofrece *"desacoplamiento completo de componentes"* o *"escalabilidad e independencia absoluta de desarrollo"* es falsa aquí. Al compartir el esquema de la base de datos (`db/init.sql`), cualquier modificación en las tablas obliga a coordinar y modificar en paralelo el código de la API y del Worker, rompiendo esa supuesta independencia teórica.

#### 3. Ventajas e inconvenientes del diseño para una tienda pequeña

| Ventajas | Inconvenientes |
| :--- | :--- |
| **Simplicidad y rapidez de despliegue:** Con un solo comando se levanta todo el entorno, facilitando el mantenimiento en proyectos pequeños sin complicar la infraestructura. | **Punto único de fallo:** Si la base de datos o el Redis fallan, tanto la API como el Worker dejan de funcionar por completo al depender de las mismas instancias. |
| **Bajo consumo de recursos:** Al estar centralizado, consume menos memoria y abarata los costes de alojamiento en servidores modestos. | **Efecto dominó en los cambios:** Modificar el esquema de datos obliga a actualizar y coordinar el código de varios componentes a la vez. |

---

## Ejercicio 6: Ficha de requerimientos de implantación

### 1. Hardware o instancia
* **Ya lo trae:** Aplicación contenerizada con 5 servicios Alpine orquestados con Docker Compose y persistencia en volúmenes locales (datos-db y datos-cache).
* **Añadir para producción:** 
  * Servidor VPS o VM con un mínimo de 2 vCPU, 4 GB RAM y 40-80 GB SSD.
  * Migrar PostgreSQL y Redis a servicios gestionados en la nube para eliminar el punto único de fallo.
  * Definir límites de recursos (mem_limit y cpus) por contenedor y configurar sincronización horaria mediante NTP.

### 2. Software y versiones
* **Ya lo trae:** Versiones fijadas en las imágenes de Docker (Nginx 1.27, Node 20, PostgreSQL 16 y Redis 7). Dependencias declaradas en package.json.
* **Añadir para producción:** 
  * Generar y versionar el archivo package-lock.json en api/ y worker/ para congelar las dependencias.
  * Cambiar el comando npm install por npm ci en los archivos Dockerfile para asegurar instalaciones idénticas.
  * Sustituir las etiquetas de imagen por identificadores únicos (SHA/Digest) y configurar escaneo de vulnerabilidades con Trivy o Grype.

### 3. Red (Dominio, DNS, puertos y cortafuegos)
* **Ya lo trae:** Único puerto publicado al host (8080 en el proxy). Los servicios api, db y cache están aislados dentro de la red interna de Compose. Nginx gestiona el proxy inverso hacia la API eliminando el prefijo /api.
* **Añadir para producción:** 
  * Registrar un dominio y configurar los registros DNS A/AAAA apuntando a la IP pública del servidor.
  * Implementar cifrado TLS/HTTPS en los puertos 80/443 usando certificados Let's Encrypt o un balanceador de carga.
  * Configurar un cortafuegos para cerrar el puerto 8080 al exterior y restringir el acceso SSH únicamente a direcciones IP de administración.

### 4. Seguridad y secretos
* **Ya lo trae:** Parametrización de credenciales mediante variables de entorno (POSTGRES_PASSWORD y JWT_SECRET) con una plantilla de ejemplo .env.example.
* **Añadir para producción:** 
  * Crear de inmediato el archivo .gitignore para evitar la subida accidental del archivo .env real al repositorio.
  * Reemplazar el archivo .env plano por un gestor de secretos dedicado como Docker Secrets o HashiCorp Vault.
  * Implementar código de autenticación real en las rutas de la API, ya que la librería jsonwebtoken está declarada pero nunca se importa ni se utiliza.
  * Configurar los contenedores para que se ejecuten con usuarios no-root (directiva USER en Dockerfile) y activar Rate Limiting en Nginx.

### 5. Entornos
* **Ya lo trae:** Un solo archivo docker-compose.yml que unifica la definición de los servicios y la variable NODE_ENV=production configurada en la API.
* **Añadir para producción:** 
  * Separar las configuraciones creando un archivo específico para el despliegue final (ej. docker-compose.prod.yml).
  * Alojar las imágenes construidas en un registro privado (Docker Hub, AWS ECR o GitHub Packages) etiquetadas por versión.
  * Diseñar un pipeline automatizado de CI/CD que cubra las fases de construcción, ejecución de tests, escaneo y despliegue.

### 6. Datos y copias de seguridad
* **Ya lo trae:** Persistencia configurada en volúmenes de Docker. Creación de tablas y carga de 3 libros de prueba controlada por el archivo db/init.sql (solo se ejecuta en la inicialización). Redis configurado con snapshot periódico RDB.
* **Añadir para producción:** 
  * Automatizar la ejecución de copias de seguridad cifradas de PostgreSQL de manera programada (pg_dump) y almacenarlas en un repositorio externo.
  * Integrar una herramienta de migraciones de base de datos (como node-pg-migrate) para aplicar cambios en el esquema sobre bases de datos ya existentes.
  * Activar el modo AOF (Append Only File) en Redis para asegurar la persistencia inmediata de los pedidos pendientes en la cola de facturas ante apagones.

### 7. Monitorización y registros
* **Ya lo trae:** Centralización de salidas stdout/stderr mediante el driver de logs nativo de Docker. Endpoint de comprobación en la ruta GET /salud y directiva restart: unless-stopped.
* **Añadir para producción:** 
  * Configurar políticas de rotación de archivos de log en el demonio de Docker para prevenir la saturación del almacenamiento del servidor.
  * Implementar la recolección centralizada de logs y métricas de infraestructura utilizando herramientas como Prometheus, Grafana, Loki o ELK.
  * Establecer alertas automáticas para eventos críticos como la caída de un contenedor, el agotamiento de memoria o el crecimiento inusual de la cola de facturas.

### 8. Fases del despliegue y plan de vuelta atrás (Rollback)
* **Ya lo trae:** Secuencia de arranque controlada por dependencias explícitas (depends_on con condiciones de salud para la base de datos).
* **Añadir para producción:** 
  * **Fases del despliegue:** Generar backup de seguridad de los datos -> Ejecutar migraciones del esquema -> Levantar contenedores actualizados -> Realizar pruebas de humo sobre el endpoint /salud antes de derivar el tráfico de usuarios.
  * **Plan de vuelta atrás:** Mantener almacenada localmente la versión anterior de la imagen para poder restaurar el servicio de forma inmediata mediante un comando docker compose up -d especificando el tag previo en caso de detectar fallos.


---

## Ejercicio 7: Caza de afirmaciones falsas

A lo largo del proceso de análisis, el agente incurrió en contradicciones y generalizaciones de manual que no corresponden con el código real del proyecto. Se recogen tres de ellas:

1. **Afirmación falsa del agente:** *"La aplicación está completamente desacoplada mediante una arquitectura de microservicios independientes."*
   - **Fichero y línea que la desmienten:** `docker-compose.yml` (líneas de definición de servicios) y `api/src/server.js` junto a `worker/worker.js`.
   - **Afirmación correcta:** La aplicación es un sistema Monolítico Distribuido o Cliente-Servidor en 3 capas. No son microservicios porque tanto la API como el Worker comparten la misma base de datos física (`db`) y se acoplan directamente a ella, rompiendo la independencia de persistencia.

2. **Afirmación falsa del agente:** *"La comunicación entre la API y el Worker se realiza de forma asíncrona mediante un patrón Pub/Sub (Publicación/Suscripción) de Redis."*
   - **Fichero y línea que la desmienten:** `worker/worker.js`, Línea 9.
   - **Afirmación correcta:** El proyecto no utiliza Pub/Sub (que enviaría el mensaje en tiempo real a todos los suscritos y se perdería si el worker estuviera apagado). En su lugar, utiliza una **cola de tareas basada en listas** mediante el comando bloqueante `brPop` sobre la clave `cola:facturas`. Es un modelo de productor-consumidor clásico donde el worker se bloquea a la espera de elementos.

3. **Afirmación falsa del agente:** *"El proyecto cuenta con un sistema de autenticación seguro implementado mediante JSON Web Tokens (JWT) para proteger las rutas."*
   - **Fichero y línea que la desmienten:** `api/package.json` (línea de dependencia de `jsonwebtoken`) y `api/src/server.js` (todo el archivo).
   - **Afirmación correcta:** Aunque la librería `jsonwebtoken` está declarada en las dependencias del `package.json`, **nunca es importada (`require`) ni utilizada** en el código del servidor Express para proteger la ruta `POST /api/pedidos` ni ninguna otra. Actualmente, cualquier usuario puede realizar peticiones sin autenticarse.




