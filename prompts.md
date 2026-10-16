## Ejercicio 2
Actúa en modo solo lectura. Analiza el archivo @docker-compose.yml y genera una tabla Markdown con el inventario de componentes del proyecto. La tabla debe incluir exactamente las siguientes columnas para cada servicio definido: Servicio, Imagen o Construcción, Puerto publicado, Puerto interno, Volúmenes, Dependencias, Función. Al final, indícame claramente cuáles de estos servicios son accesibles externamente desde fuera del equipo y justifica el motivo analizando los puertos mapeados.

## Ejercicio 3
Actúa como un experto en arquitectura de software. Analiza los archivos y la estructura de este proyecto y realiza las siguientes tareas: 1. Clasifica todos los recursos de la aplicación en estas categorías (Contenido estático, Código de servidor, Datos y esquema, Dependencias, Configuración, Secretos, Registros de logs). 2. Indica para cada recurso en qué fichero o servicio vive. 3. Identifica explícitamente cuáles de estos recursos NO deben subirse bajo ningún concepto al repositorio Git. 4. Comprueba si el proyecto cuenta actualmente con un archivo .gitignore. Si no existe o está incompleto, propón un archivo .gitignore adecuado y optimizado para esta tecnología.

## Ejercicio 4
Analiza detalladamente las líneas de código de api/src/server.js y worker/worker.js que gestionan GET /api/libros y POST /api/pedidos. Genera los diagramas Mermaid para ambas peticiones (incluyendo la caché en el GET; y la cola de facturas junto al worker en el POST), pero asegúrate estrictamente de que no haya alucinaciones. Cada paso del diagrama debe corresponderse con una función, middleware o llamada real en esos archivos. Si encuentras alguna ambigüedad entre lo que suele ser un 'estándar de manual' y lo que está programado, cíñete al código real.

Vamos a comprobar y corregir los diagramas que me has dado frente al código real: 1. Revisa de nuevo api/src/server.js. ¿El flujo de GET /api/libros comprueba primero la caché? ¿Qué pasa exactamente si hay un 'miss' (fallo de caché)? Corrige el código Mermaid si omitiste algún paso intermedio. 2. Revisa worker/worker.js. ¿Cómo recibe el worker los datos de la cola? ¿Es mediante un 'polling' o un evento? Asegúrate de que el diagrama de POST /api/pedidos refleje fielmente este mecanismo exacto de la cola de facturas. Dame los bloques de código Mermaid definitivos y depurados listos para el visor.

## Ejercicio 5
¿Qué modelo de arquitectura sigue la aplicación y por qué?"

## Ejercicio 6
Ayúdame a redactar una ficha de requerimientos de implantación para poner esta aplicación en producción. Necesito que desgloses la información en los siguientes apartados, distinguiendo claramente lo que la aplicación ya trae de lo que hay que añadir para producción: Hardware o instancia; Software y versiones; Red (dominio, DNS, puertos, cortafuegos); Seguridad y secretos; Entornos; Datos y copias de seguridad; Monitorización y registros; Fases del despliegue y plan de vuelta atrás.

Vuelca todo este contenido de la ficha de requerimientos de implantación (los 8 apartados con sus secciones de 'Ya lo trae' y 'Añadir para producción') dentro del archivo docs/arquitectura.md, justo debajo del ejercicio 5.

