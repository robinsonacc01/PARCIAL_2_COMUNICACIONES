# Informe Técnico: Despliegue Multi-Contenedor Orquestado con Docker Compose

**Asignatura:** Redes y Comunicaciones  
**Universidad Militar Nueva Granada**  

---

## 1. Arquitectura y Topología de Red

La infraestructura está compuesta por 5 contenedores orquestados mediante Docker Compose, aislados internamente y expuestos de manera segura al exterior mediante un Proxy Inverso (Nginx).

### Diagrama de Flujo y Redes
* **`frontend_net` (Red Bridge):** Interconecta únicamente a `nginx` y `joomla`.
* **`backend_net` (Red Bridge):** Interconecta a `nginx`, `joomla`, `database`, `jupyter` y `grafana`.

```text
[ Cliente / Navegador Web ]
             |
         (Puerto 80)
             v
   +------------------+
   |   nginx_proxy    | (Único servicio expuesto)
   +--------+---------+
            |
   +--------+-----------------------+-----------------------+
   | (frontend_net)                 | (backend_net)         |
   v                                v                       v
+------------+            +-------------------+    +-----------------+
| joomla_cms | <--------> |    postgres_db    | <--| jupyter_notebook|
+------------+            +---------+---------+    +-----------------+
                                    ^
                                    |
                          +---------+---------+
                          | grafana_dashboard |
                          +-------------------+
## 2. Análisis Detallado del Modelo OSI en la Solución

### Capa 7 (Aplicación)

Nginx actúa como punto único de entrada y añade cabeceras HTTP esenciales antes de reenviar cada petición a los contenedores internos. La cabecera `Host` conserva el nombre de dominio original solicitado por el cliente, para que Joomla genere enlaces correctos. `X-Forwarded-For` preserva la IP real del cliente, ya que de otro modo Joomla solo vería la IP interna de Nginx como origen de todas las peticiones. `X-Forwarded-Proto` informa si la conexión original era HTTP o HTTPS, dato necesario para que la aplicación construya URLs con el protocolo correcto.

Para Jupyter, la ruta `/jupyter/` requiere un mecanismo especial: el protocolo HTTP Upgrade. El kernel de Jupyter se comunica con el navegador mediante WebSockets, una conexión persistente bidireccional distinta a una petición HTTP normal. Nginx debe enviar las cabeceras `Upgrade` y `Connection: Upgrade` para que esa conexión se "actualice" de HTTP a WebSocket y el kernel pueda ejecutar código en tiempo real.

PostgreSQL utiliza su propio protocolo binario de aplicación, distinto de HTTP, basado en un intercambio de mensajes cliente-servidor sobre TCP, donde el cliente autentica, envía consultas SQL y el servidor responde con resultados tabulares. Joomla genera logs de acceso en formato de texto estructurado, con campos como fecha, IP de origen, método HTTP, ruta solicitada y código de respuesta, similar al formato combinado de Apache.

### Capa 4 (Transporte)

Cada servicio expone un puerto TCP específico dentro de la red interna de Docker. Nginx escucha en el puerto 80, único publicado hacia el host. PostgreSQL escucha en el puerto 5432, accesible solo desde la red backend. Jupyter expone el puerto 8888 internamente, y Grafana el puerto 3000, ambos alcanzables únicamente a través del proxy inverso.

Las conexiones entre Joomla y PostgreSQL son persistentes gracias al uso de connection pooling, lo que evita el costo de abrir y cerrar una conexión TCP por cada consulta SQL. De forma similar, Nginx mantiene conexiones keep-alive con los clientes y con los servicios internos, reduciendo la sobrecarga de establecer el saludo de tres vías de TCP en cada petición HTTP.

### Capa 3 (Red)

Docker crea una subred privada distinta para cada red bridge definida en el archivo de orquestación. Los contenedores conectados a `frontend_net` reciben direcciones IP en un rango, y los conectados a `backend_net` en otro rango distinto. El contenedor de base de datos solo pertenece a la red backend, por lo que no tiene una ruta de red hacia el exterior ni hacia los clientes externos, cumpliendo así el aislamiento exigido.

Docker incluye un servidor DNS embebido en la dirección interna 127.0.0.11, que resuelve automáticamente los nombres de los servicios, como `database` o `joomla`, hacia su dirección IP interna correspondiente. Gracias a esto, los contenedores se comunican entre sí usando el nombre del servicio en lugar de direcciones IP fijas, lo que simplifica la configuración y la hace portable entre máquinas.

El kernel del host administra las reglas de reenvío y traducción de direcciones, NAT, que permiten que el puerto 80 publicado llegue desde la red del host hasta la interfaz interna del contenedor de Nginx, usando reglas de iptables que Docker configura automáticamente al levantar los contenedores.

### Capa 2 (Enlace de Datos)

Cada contenedor recibe una interfaz de red virtual, del tipo veth, que funciona en pareja: un extremo vive dentro del contenedor y el otro se conecta a un puente virtual, del tipo bridge, creado por Docker en el host. Este puente actúa como un switch virtual que interconecta todas las interfaces de los contenedores pertenecientes a la misma red.

Cuando dos contenedores dentro del mismo puente necesitan comunicarse, antes de enviar tráfico usan el protocolo ARP para resolver qué dirección de hardware, o MAC, corresponde a la dirección IP destino. Esta resolución ocurre de forma completamente interna, dentro del puente virtual, sin salir nunca hacia la red física del host.

## 3. Flujo de Logs y Métricas hacia Grafana

Grafana obtiene su información mediante una fuente de datos de tipo PostgreSQL configurada de forma declarativa en la carpeta de aprovisionamiento. Esta fuente apunta directamente a la base de datos que usa Joomla, consultando sus tablas internas para extraer estadísticas de actividad, como número de sesiones, usuarios o contenido creado. De esta manera, cada vez que un usuario interactúa con el portal Joomla, esa actividad queda reflejada en las tablas de PostgreSQL, y Grafana simplemente ejecuta consultas SQL programadas contra esas tablas para generar los paneles, sin necesidad de ningún paso manual de configuración.

## 4. Guía de Verificación y Demostración

Primero, se abre un navegador y se visita la dirección del servidor en el puerto 80, lo que carga el portal institucional de Joomla a través del proxy Nginx. Se navega por algunas secciones del sitio para generar tráfico real y así poblar las tablas de actividad en la base de datos.

Segundo, se abre la ruta correspondiente a Grafana, se inicia sesión con las credenciales configuradas, y se verifica que el panel ya muestre gráficas pobladas con información proveniente de las consultas a PostgreSQL, sin haber creado manualmente ninguna fuente de datos ni panel.

Tercero, se abre la ruta correspondiente a Jupyter, se introduce el token de acceso, se localiza el cuaderno precargado en la carpeta de trabajo, y se ejecutan sus celdas de código en orden. El cuaderno se conecta a la base de datos PostgreSQL mediante la librería psycopg2 y muestra como resultado el listado de tablas creadas por Joomla, confirmando que la integración entre los cinco contenedores funciona correctamente de principio a fin.