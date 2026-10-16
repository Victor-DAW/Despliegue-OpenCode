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
### Resumen de OpenCode
Flujo de funcionamiento

1. Frontend (web/index.html): página estática que hace fetch('/api/libros') y pinta el listado de libros.
2. Nginx: sirve el HTML y actúa como proxy inverso, redirigiendo /api/* a http://api:3000/.
3. API (api/src/server.js):
 - GET /salud -> healthcheck simple.
 - GET /libros -> lee de Redis; si no está en caché, consulta PostgreSQL y la guarda 60 s (cache-aside).
 - POST /pedidos -> inserta un pedido en PostgreSQL y encola su id en Redis (cola:facturas).
4. Worker (worker/worker.js): bloquea con brPop sobre cola:facturas y "genera la factura" de cada pedido (actualmente solo registra en consola).

Datos

db/init.sql crea las tablas libros y pedidos (con clave foránea y timestamp) y siembra 3 libros de ejemplo: El Quijote, Rayuela y La colmena.

Configuración

- Puertos: el proxy publica el 8080 -> contenedor 80. La API interna usa el 3000.
- Variables de entorno en .env (ver .env.example): POSTGRES_PASSWORD, JWT_SECRET. La API además usa DATABASE_URL y REDIS_URL.
- opencode.json configura OpenCode con edit: deny, bash: ask y share: disabled (agente en modo consulta).

Observaciones

- Hay dependencia de arranque: la API espera a que la db esté sana (service_healthy).
- El worker arranca con service_started en Redis (no espera healthcheck).
- No existe web como servicio propio: el HTML se monta como volumen en el proxy.

### Pido que modifique el archivo docker-compose.yml y guarde los cambios
Estoy en modo Plan, así que no puedo modificar archivos todavía. Puedo mostrarte exactamente qué cambiaría, y luego tú cambias de agente para que lo aplique.

## 3. Recursos de la aplicación

### Clasificación de recursos e inventario

---

### Propuesta de `.gitignore`

Tras comprobar con la terminal (`ls -la`), se valida que el proyecto requiere la configuración del siguiente archivo `.gitignore` optimizado para el entorno de desarrollo con Docker y Node.js:

```gitignore
# Entorno y variables de entorno
.env
.env.local
.env.production

# Dependencias
node_modules/
jspm_packages/

# Registros - logs
*.log
logs/
npm-debug.log*
yarn-debug.log*

# Datos, volúmenes y bases de datos locales
.data/
db_data/
*.sqlite

# IDEs, editores y sistema operativo
.vscode/
.idea/
.DS_Store
```
***

### 💡 Notas que debes tener en cuenta:
1. **Consistencia:** He copiado textualmente la estructura por bloques (`# Entorno...`, `# Dependencias...`) y la jerarquía que se llega a distinguir en las líneas finales de código de tu captura.
2. **Adaptación de nombres:** Si las carpetas de datos de tu base de datos o de tu app real se llaman de otra forma (por ejemplo, en lugar de `db_data/` se llama `mysql_data/`), cámbialo en la tabla y en el `.gitignore` para que coincida fielmente con lo que tienes en tu proyecto.

<FollowUp>
¿Quieres que preparemos el contenido para **algún otro apartado** de este mismo archivo `docs/arquitectura.md`, como el inventario de componentes del ejercicio 2?
</FollowUp>

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



