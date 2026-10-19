# [ExP 89] Fase 3 - Implementación de HTTPS con Nginx Reverse Proxy y Docker

## Resultados de aprendizaje

Aplicación práctica de RA4-C4 (certificación y comunicaciones seguras) y RA1-C1 (elección de mecanismos para proteger la transmisión).

---

Publica la aplicación existente mediante HTTPS, utilizando Nginx como reverse proxy y el certificado emitido por tu CA. La aplicación debe seguir funcionando y la base de datos debe conservar sus datos.

Al finalizar todas las actividades de la **Fase 2** tendrás:

- El cliente accede a la aplicación por `https://NOMBRE_DEL_PROYECTO` mediante el puerto **443**.
- Nginx utiliza **el certificado y la clave privada creados en la Fase 2**.
- El certificado se **verifica correctamente** frente a la CA del laboratorio, sin desactivar la comprobación de confianza.
- El acceso por HTTP se redirige a HTTPS.
- Nginx comunica las peticiones al servicio web interno y este mantiene su conexión con la base de datos, si la utiliza.
- Los contenedores web y de base de datos no publican innecesariamente sus puertos al host.
- La configuración y las evidencias son reproducibles, sin claves privadas ni contraseñas en GitHub.

> **Alcance:** en esta fase implantaremos y probaremos HTTPS de extremo a extremo. Las pruebas avanzadas de seguridad TLS, el análisis del tráfico cifrado en Suricata y la monitorización detallada corresponden a la **Fase 4**. No tienes que rehacer tu aplicación ni tu base de datos.

---

# Actividad 8 - Revisión del punto de partida

### 8.1. Comprobar el estado actual

Desde el directorio donde se encuentra tu proyecto Docker Compose, comprueba:

```bash
docker compose ps
docker compose ls
ss -tulnp
```

Comprueba también que la web responde con el procedimiento que utilizabas en MF0488. Por ejemplo:

```bash
curl -I http://IP_SERVIDOR:PUERTO_ACTUAL/
```

**Adapta la IP y el puerto:** no todos los proyectos publicaron la misma aplicación en el mismo puerto.

Identifica qué contenedor sirve la aplicación, cuál contiene la base de datos y si ya existe algún reverse proxy. Si el proyecto utiliza WordPress, anota cómo se conectan WordPress y MariaDB/MySQL.

### 8.2. Proteger el trabajo anterior

Antes de cambiar las redes, los puertos o el Compose:

1. Guarda una copia de las configuraciones actuales y realiza una copia de seguridad de la base de datos y de los datos persistentes. Puedes utilizar el procedimiento de backup de tu proyecto o una instantánea controlada de la VM.
2. Asegúrate de que sabes cómo volver a la configuración anterior si la nueva no funciona.
3. **No ejecutes `docker compose down -v`**: la opción `-v` elimina volúmenes y podría hacerte perder la base de datos o la instalación de WordPress.

### 8.3. Comprobar los certificados emitidos en la Fase 2

Deben estar instalados en Ubuntu:

```text
/etc/ssl/private/servidor.key
/etc/ssl/certs/servidor.crt
/etc/ssl/mf0489/ca.crt
```

Comprueba la existencia de los archivos y sus permisos **sin mostrar el contenido de las claves privadas**:

```bash
sudo ls -l /etc/ssl/private/servidor.key
ls -l /etc/ssl/certs/servidor.crt
ls -l /etc/ssl/mf0489/ca.crt
```

Verifica el certificado utilizando **el nombre DNS que elegiste en la Fase 2**:

```bash
DOMINIO="portal.proyecto.test"   # Sustituye por tu dominio

openssl verify \
  -CAfile /etc/ssl/mf0489/ca.crt \
  -purpose sslserver \
  -verify_hostname "$DOMINIO" \
  /etc/ssl/certs/servidor.crt
```

---

**Resultado esperado:** `/etc/ssl/certs/servidor.crt: OK`. Si el nombre no coincide con el SAN, corrige el certificado siguiendo el procedimiento de emisión de la Fase 2; no ignores el error.

### Evidencias

- Estado previo de los servicios
- Puertos actuales
- Respaldo preparado y validación `OK` del certificado

---

# Actividad 9 - Adaptación de la arquitectura Docker

Actualiza el diagrama de la Fase 1 para indicar **los puertos y las redes reales** que se van a utilizar.

### Arquitectura de referencia

```text
                  CLIENTE / NAVEGADOR
                        │
           HTTP :80     │     HTTPS :443
                  └─────┼──────┘
                        ▼
              NGINX REVERSE PROXY
              Termina la sesión TLS
              servidor.crt + servidor.key
                        │
             HTTP interno (red web)
                        │
                        ▼
                 APLICACIÓN WEB
                        │
               (red de base de datos)
                        │
                        ▼
                  MARIA DB / MYSQL

Ubuntu Server: SSH + Suricata + Docker + registros
CA privada: permanece en el equipo de administración, fuera de la VM
```

### Decisiones obligatorias

**Documenta brevemente:**

1. **Qué contenedor actuará de reverse proxy:** uno nuevo de Nginx o el Nginx que ya existe, si realmente puede realizar esa función.
2. **Qué nombre, puerto y protocolo internos tiene tu aplicación:** por ejemplo, la imagen `wordpress:apache` habla HTTP en el puerto **80 del contenedor** aunque se publicase como `8080` en el host. **Una aplicación WordPress con PHP-FPM no habla HTTP por el puerto 9000**: en ese caso manten el Nginx HTTP existente como backend o adaptar ese mismo Nginx para que reciba HTTPS.
3. **Qué contenedores comparten la red web:** Nginx tiene que poder comunicarse con la aplicación utilizando el DNS interno de Docker.
4. **Qué contenedores comparten la red de base de datos:** si existe una BBDD, se comunicará con la aplicación, pero no tiene por qué ser accesible desde Nginx ni desde el exterior.
5. **Dónde termina TLS:** el tráfico cliente → Nginx será HTTPS; el tramo Nginx → aplicación puede seguir siendo HTTP interno, siempre que documentes este límite.

> No confundas el puerto interno del contenedor con el puerto publicado en Ubuntu. Tampoco confundas `expose` con `ports`: `expose` no publica el servicio en la interfaz del host.

---

**Resultado esperado:** un diagrama adaptado a tu implementación, no una copia de la arquitectura de referencia.

---

# Actividad 10 - Incorporación del reverse proxy

Modifica el `docker-compose.yml`. *No elimines los volúmenes de la aplicación o de la BBDD y no copies un Compose nuevo sin adaptarlo.*

### 10.1. Definir el servicio Nginx

Como referencia, un servicio puede tener esta estructura:

```yaml
services:
  reverse-proxy:
    image: nginx:stable-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
      - /etc/ssl/certs/servidor.crt:/etc/nginx/certs/servidor.crt:ro
      - /etc/ssl/private/servidor.key:/etc/nginx/certs/servidor.key:ro
    networks:
      - red_web
```

> Este fragmento no es un `docker-compose.yml` completo. Integrarlo con tus servicios, redes, credenciales externalizadas y volúmenes preexistentes. Si tus servicios o redes tienen otros nombres, utilízalos.

Investiga y explica brevemente:

- ¿Qué realiza `ports`?
- ¿Para qué sirve cada entrada de `volumes`?
- ¿Qué significa `:ro`?
- ¿Por qué Nginx necesita `servidor.key`, pero no `ca.key`?
- ¿Necesita el reverse proxy estar conectado directamente a la red de la base de datos?

**Importante:** el archivo `ca.crt` es necesario para que los **clientes** puedan verificar el certificado del servidor; Nginx no necesita recibir la clave privada de la CA ni instalarla para servir el certificado del servidor.

### 10.2. Revisar los puertos de los servicios anteriores

Si antes exponías WordPress en `8080:80` o una web en `80:80`, revisa si esa publicación todavía es necesaria.

**Objetivo:** el punto de entrada público de la aplicación será Nginx, en los puertos 80 y 443. La aplicación tiene que ser accesible por su red Docker interna, sin un segundo puerto web publicado al host salvo una necesidad justificada.

### 10.3. Verificar la sintaxis del Compose

Valida la configuración sin imprimir en la terminal las contraseñas ya sustituidas en las variables del Compose.

```bash
docker compose config --quiet
```

### 10.4. Poner en marcha los cambios

Cuando el Compose sea correcto, comprueba los servicios

```bash
docker compose ps
docker compose logs --tail=50 reverse-proxy
```

---

### Evidencias

- Compose adaptado
- Sintaxis válida
- Contenedores activos
- Puertos publicados
- Redes justificadas

---

# Actividad 11 - Configuración de Nginx

Configura Nginx para poder tener un reverse proxy.

### 11.1. Preparar el archivo de configuración

En tu repositorio de trabajo, crea el archivo:

```text
nginx/default.conf
```

### 11.2. Requisitos obligatorios de Nginx

Configuración de Nginx:

1. Bloque `server` para escuchar en **80** y redirigir a HTTPS.
2. Bloque `server` para escuchar en **443** con TLS.
3. Mismo `server_name` utilizado en el SAN del certificado.
4. Referencias a los archivos montados dentro del contenedor:

   ```text
   /etc/nginx/certs/servidor.crt
   /etc/nginx/certs/servidor.key
   ```

5. Protocolos **TLS 1.2 y TLS 1.3** (no SSL ni TLS 1.0/1.1).
6. Una directiva `proxy_pass` hacia **el nombre del servicio web dentro de Docker** y su puerto interno.
7. Las cabeceras necesarias para que la aplicación conozca el `Host`, la IP del cliente y el protocolo original HTTPS.

### 11.3. Esqueleto a completar

En lugar de copiar una configuración terminada, investiga las directivas necesarias y completa los espacios marcados.

```nginx
server {
    listen 80;
    server_name NOMBRE_DEL_PROYECTO;

    # TODO: redirigir permanentemente a la URL https
}

server {
    listen 443 ssl;
    server_name NOMBRE_DEL_PROYECTO;

    # TODO: ssl_certificate
    # TODO: ssl_certificate_key
    # TODO: ssl_protocols

    location / {
        # TODO: proxy_pass http://SERVICIO_WEB:PUERTO_INTERNO;
        # TODO: conservar el Host original
        # TODO: transmitir IP original del cliente
        # TODO: transmitir la cadena X-Forwarded-For
        # TODO: transmitir si el cliente llegó por HTTP o HTTPS
    }
}
```

*Puedes consultar la documentación de Nginx o utilizar el comando `nginx -h` para localizar las herramientas de comprobación.*

> No escribas rutas del sistema Ubuntu en `ssl_certificate` ni `ssl_certificate_key`: Nginx se ejecuta dentro de un contenedor y debe utilizar las rutas **internas** que has montado con Docker.

### 11.4. Verificar Nginx

Comprueba la sintaxis de la configuración de Nginx

```bash
docker compose exec reverse-proxy nginx -t
```

---

### Evidencias

- `default.conf`
- Resultado correcto de `nginx -t`
- Comprobación de que Nginx está escuchando en 443

---

# Actividad 12 - Aplicación y la base de datos

Una conexión HTTPS puede estar correctamente configurada y, aun así, mostrar una web rota.

### 12.1. Comprobar el acceso interno

Verifica que Nginx puede resolver el nombre del servicio de la aplicación y comunicarse con su puerto interno. Si aparece `502 Bad Gateway`, revisa primero el nombre del servicio, el puerto y las redes Docker compartidas.

Si la web utiliza una BBDD, comprueba que la conexión aplicación → BBDD sigue funcionando y que persisten los datos que ya tenías. **No utilices `proxy_pass http://wordpress:9000` si el contenedor es `wordpress:fpm`: el puerto 9000 utiliza FastCGI, no HTTP.** En ese escenario puedes terminar TLS en el Nginx que ya atiende PHP-FPM o situar un proxy TLS delante de ese Nginx HTTP, justificando la elección.

### 12.2. Si tu aplicación es WordPress

WordPress puede necesitar reconocer que **el cliente se ha conectado por HTTPS**, aunque Nginx reenvíe internamente la petición por HTTP.

Comprueba que tu proxy establece correctamente:

```nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Host $host;
```

> La **imagen oficial de WordPress para Docker** incorpora lógica para reconocer `X-Forwarded-Proto` en su configuración predeterminada. Si has modificado `wp-config.php` o utilizas otra imagen, revisa este comportamiento. En arquitecturas con un Nginx intermedio antes de PHP-FPM comprueba, además, que la información del protocolo HTTPS llega hasta WordPress.

> Revisa además las URL de WordPress (dirección de WordPress y dirección del sitio). Si siguen apuntando a `http://IP:PUERTO`, pueden aparecer redirecciones a HTTP, bucles o recursos mixtos. Actualízalas de forma controlada a la URL `https://TU_DOMINIO` cuando corresponda, conservando una copia de seguridad.


Comprueba:

- Página principal.
- Navegación por algún enlace interno.
- Página de administración o login, si corresponde.
- Acceso efectivo a los datos de la base de datos, si tu aplicación los utiliza.

### 12.3. Si tu aplicación NO es WordPress

Adapta el `proxy_pass` a su servicio y puerto reales. Comprueba que genera enlaces correctos y que entiende, cuando lo necesite, las cabeceras reenviadas por el proxy. Si ya tenías un Nginx sirviendo archivos estáticos, podrás mantenerlo como backend o justificar cómo lo has adaptado para ejercer también de proxy.

---

### Evidencias

- Navegación funcional
- Conexión con la BBDD cuando exista
- Ausencia de errores 502
- Persistencia de los datos anteriores.

---

# Actividad 13 - Comprobación HTTPS desde un cliente

Las pruebas deben demostrar que la comunicación es válida **sin ignorar la confianza del certificado**.

### 13.1. Preparar el acceso por nombre

El certificado está emitido para el nombre indicado en su SAN.

- Añade una entrada a `hosts` en el **cliente** para relacionar el dominio y la IP de Ubuntu.
- Utiliza temporalmente `curl --resolve`, sin modificar `hosts`.

> No basta con abrir `https://IP_SERVIDOR` si el certificado solo incluye un SAN de tipo DNS. La IP necesitaría aparecer también como SAN de tipo `IP` para validarse por dirección.

### 13.2. Comprobar la redirección HTTP → HTTPS

Desde un cliente que pueda alcanzar la VM:

```bash
DOMINIO="portal.proyecto.test"      # Adapta el nombre
IP_SERVIDOR="192.168.X.X"          # IP real de Ubuntu Server

curl -I \
  --resolve "${DOMINIO}:80:${IP_SERVIDOR}" \
  "http://${DOMINIO}/"
```

**Resultado esperado:** una respuesta de redirección (habitualmente 301 o 308) cuya cabecera `Location` apunte a `https://DOMINIO/...`.

### 13.3. Verificar el acceso HTTPS

Copia al **cliente** únicamente el certificado público de tu CA (`ca.crt`). No copies `ca.key` ni la clave privada del servidor.

Adapta la ruta siguiente a la ubicación de `ca.crt` **en el equipo cliente**:

```bash
CA_CERT="./ca.crt"

curl -v \
  --cacert "$CA_CERT" \
  --resolve "${DOMINIO}:443:${IP_SERVIDOR}" \
  "https://${DOMINIO}/" \
  -o /dev/null
```

Verifica

- El certificado es aceptado frente a la CA indicada.
- El nombre DNS coincide con el SAN.
- Se establece TLS y la aplicación responde.
- El comando termina sin un error de verificación.

**No se aceptará `curl -k` ni `--insecure` como prueba de funcionamiento seguro.**

### 13.4. Verificar el servidor con OpenSSL

Desde un cliente que tenga `ca.crt`:

```bash
openssl s_client \
  -connect "${IP_SERVIDOR}:443" \
  -servername "$DOMINIO" \
  -CAfile "$CA_CERT" \
  -verify_hostname "$DOMINIO" \
  -verify_return_error \
  -brief \
  </dev/null
```

Localiza el resultado correcto de verificación del certificado y la versión TLS negociada. Si aparece un error de confianza o de identidad, investígalo y corrígelo.

### 13.5. Comprobar que los servicios originales continúan disponibles

Busca peticiones de prueba y errores relevantes. Si el tráfico no llega desde el cliente, revisa los puertos publicados y la configuración del firewall de Ubuntu, si está activo.

```bash
docker compose ps

docker compose logs --tail=30 reverse-proxy
```

---

### Evidencias

- Redirección HTTP
- Validación con `curl --cacert`
- Validación con OpenSSL
- Aplicación accesible por HTTPS desde un cliente

---

# [ExP 89] Fase 3 - Documentación

Actualiza GitHub para que otra persona pueda reproducir el **despliegue de Nginx y Docker Compose**, sin necesidad de disponer de tus claves privadas.

### Estructura orientativa

```text
docker-compose.yml
.env.example
.gitignore
nginx/
└── default.conf

docs/
└── mf0489/
    ├── arquitectura.md
    ├── pki.md
    ├── https-docker.md
    └── evidencias/
        └── fase3/
```

**Conserva los archivos y carpetas reales de tu proyecto:** esta estructura es orientativa, no una orden para eliminar ni reemplazar los recursos anteriores.

### Contenido de `https-docker.md`

1. Arquitectura final y punto de terminación TLS.
2. Servicios, redes y puertos antes y después de la modificación.
3. Decisiones tomadas en Docker Compose y Nginx.
4. Rutas de los certificados y permisos aplicados, sin mostrar claves privadas.
5. Cómo resolviste el nombre DNS en el cliente de laboratorio.
6. Pruebas de HTTP → HTTPS y de confianza TLS.
7. Pruebas de funcionamiento de la aplicación y de persistencia de datos.
8. Incidencias reales encontradas y solución aplicada.
9. Instrucciones de despliegue y verificación reproducibles.

### Reglas de seguridad en GitHub

- **No publiques** `/etc/ssl/private/servidor.key`, `ca.key`, contraseñas, `.env` con valores reales, copias de seguridad de la BBDD ni datos personales.
- Publica `docker-compose.yml`, `nginx/default.conf`, `.env.example` y documentación sin secretos.
- No construyas una imagen Docker que incorpore la clave privada mediante `COPY`.
- No copies `ca.key` a Ubuntu ni la incluyas en la futura OVA del CTF.
- Si una clave privada se ha publicado, cambia esa clave y retira el secreto del historial accesible: **añadir `.gitignore` después no soluciona la exposición**.
- Utiliza certificados y datos **exclusivos del laboratorio**. No reutilices claves o contraseñas de servicios reales en la futura máquina CTF.

---

# [ExP 89] Fase 3 - Extra

## Fuentes técnicas de consulta

- [NGINX — Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [NGINX — ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- [Docker Hub — Imagen oficial de WordPress](https://hub.docker.com/_/wordpress)
- [WordPress — HTTPS y reverse proxy](https://developer.wordpress.org/advanced-administration/security/https/)

## Ayuda por si algo falla

| Hallazgo | Check | Hint |
|---|---|---|
| Nginx no arranca | `docker compose logs reverse-proxy` | Error de sintaxis, certificado ausente o clave inaccesible |
| `nginx -t` falla | Revisar `default.conf` y rutas internas del contenedor | Nginx no ve las rutas `/etc/ssl/...` del host si no se montan |
| `502 Bad Gateway` | Servicio/puerto de `proxy_pass` y redes Docker | Nginx no alcanza al backend |
| `bind: address already in use` | `ss -tulnp` y `docker compose ps` | Otro servicio sigue publicando 80 o 443 |
| `hostname mismatch` | SAN del certificado y dominio utilizado en el cliente | Conectar por IP no equivale a verificar un nombre DNS |
| `unable to get local issuer certificate` | Ruta y contenido de `ca.crt` en el cliente | Tu CA no está en la confianza usada por la prueba |
| Redirecciones infinitas en WordPress | Cabecera `X-Forwarded-Proto` y URL de WordPress | WordPress interpreta la petición interna como HTTP |
| Aparecen recursos `http://` | URL de la aplicación y enlaces almacenados | Puede existir contenido mixto |
| WordPress no conecta a MariaDB | Redes, nombre DNS del servicio BBDD y datos de conexión | El proxy no reemplaza la comunicación web ↔ BBDD |
| `curl` no conecta a la VM | IP, red de la VM, puertos y firewall | Puede no existir conectividad de red entre cliente y servidor |

> Documenta el **error real, qué investigaste y cómo lo resolviste**. Evita presentar una captura con un servicio funcional sin explicar los problemas que hayas tenido.
