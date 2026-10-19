# [ExP 89] Fase 4 - Validación de seguridad, TLS y monitorización

## Resultados de aprendizaje

RA1-C1 (evaluación de mecanismos criptográficos), RA3-C3 (verificación de certificados) y RA4-C4 (comprobación de comunicaciones seguras).

---

Actua como responsable de seguridad y **audita lo que se ha construido** demostrando que el servidor utiliza una configuración TLS adecuada y que, aunque el contenido de las conexiones HTTPS esté cifrado, puedes seguir monitorizando la actividad mediante **Suricata y los registros de Nginx**.

Al finalizar todas las actividades de la **Fase 4** tendrás:

- El certificado presentado **realmente por Nginx** es el que corresponde al dominio elegido y se valida con la CA del laboratorio.
- El servidor acepta **TLS 1.2 y TLS 1.3**, cuando el cliente los admite, y no tiene habilitados TLS 1.0 ni TLS 1.1.
- Sabes identificar la **versión TLS y cipher suite negociadas** en una conexión real.
- Has comprobado con tráfico de laboratorio la diferencia entre la visibilidad de **HTTP y HTTPS** desde la interfaz monitorizada.
- Suricata puede registrar actividad relacionada con conexiones HTTPS, **sin atribuirle visibilidad sobre las URL o el contenido cifrado**.
- Los registros del reverse proxy permiten ver las solicitudes **después de terminar TLS**.
- Puedes **correlacionar una prueba controlada** utilizando dos fuentes: Suricata y Nginx.
- La web, la base de datos y las medidas de protección de claves siguen funcionando.

> **Alcance:** todas las pruebas se realizarán sobre **tu propia VM y tu red de laboratorio**. No se requieren escaneos de terceros, ataques reales, robo de credenciales ni herramientas de explotación. **No desactives la verificación TLS** para hacer que una prueba dé resultado positivo.

---

# Actividad 15 - Establecemiento de una línea base de HTTPS | Entorno B

### 15.1. Comprobar el estado de la infraestructura

```bash
docker compose ps
sudo ss -ltnp
sudo systemctl status suricata --no-pager
```

Comprueba qué servicio publica los puertos `80` y `443`. Si no existe otra publicación web innecesaria hacia el host; la base de datos permanecerá en la red interna prevista.

Confirma que Nginx carga su configuración:

```bash
docker compose exec reverse-proxy nginx -t
```

*Adapta `reverse-proxy`* si en tu Compose utilizaste otro nombre.

### 15.2. Preparar el cliente de pruebas

Las comprobaciones de red se ejecutarán, en lo posible, desde **Kali**. De esta manera podrás determinar qué observa Suricata en la interfaz de red.

Ejemplo de variables, **en el equipo cliente**:

```bash
DOMINIO="portal.proyecto.test"    # Sustituye por el nombre usado en tu SAN
IP_SERVIDOR="192.168.X.X"          # IP real de Ubuntu Server
CA_CERT="./ca.crt"                 # Ruta del certificado PÚBLICO de tu CA en el cliente
```

*No copies `ca.key` ni `servidor.key` al cliente. Solo se necesita `ca.crt`*

### 15.3. Verificar la conexión HTTPS real

Localiza **la versión TLS**, la **cipher suite** y el **resultado de la verificación**. No basta con comprobar el archivo `servidor.crt` almacenado en Ubuntu: esta vez hay que verificar el certificado que presenta **el servicio HTTPS en funcionamiento**.

```bash
curl -v \
  --cacert "$CA_CERT" \
  --resolve "${DOMINIO}:443:${IP_SERVIDOR}" \
  "https://${DOMINIO}/" \
  -o /dev/null
```

```bash
openssl s_client \
  -connect "${IP_SERVIDOR}:443" \
  -servername "$DOMINIO" \
  -CAfile "$CA_CERT" \
  -verify_hostname "$DOMINIO" \
  -verify_return_error \
  -brief </dev/null
```

**Registra el resultado en la siguiente tabla**

| Prueba | Resultado real | Evidencia |
|---|---|---|
| Contenedores y reverse proxy activos | | |
| Puerto 443 accesible desde el cliente | | |
| Certificado y dominio válidos | | |
| Verificación de confianza satisfactoria | | |
| Versión TLS negociada | | |
| Cipher suite negociada | | |
| Aplicación funcionando | | |

---

### Evidencias

- Estado de servicios
- `nginx -t`
- Conexión `curl` validada
- Salida de `openssl s_client` con los parámetros negociados.

---

# Actividad 16 - Auditoría y ajuste de TLS

### 16.1. Revisar los protocolos permitidos

Revisa que la configuración de Nginx está realmente cargada, no únicamente el archivo del repositorio.

```bash
docker compose exec reverse-proxy nginx -T
```

Busca la directiva `ssl_protocols` en el bloque aplicable a tu servidor HTTPS. No incluyas en GitHub una salida completa si contiene datos sensibles.

**Requisito:** la configuración permitirá TLS 1.2 y TLS 1.3 y excluir SSLv2, SSLv3, TLS 1.0 y TLS 1.1.

Como referencia, la directiva podría tener esta estructura `ssl_protocols TLSv1.2 TLSv1.3;`

### 16.2. Negociar dos versiones TLS

Desde el cliente, prueba a establecer una conexión forzando TLS 1.2:

```bash
openssl s_client \
  -connect "${IP_SERVIDOR}:443" \
  -servername "$DOMINIO" \
  -CAfile "$CA_CERT" \
  -verify_hostname "$DOMINIO" \
  -verify_return_error \
  -tls1_2 -brief </dev/null
```

Repite la comprobación utilizando TLS 1.3:

```bash
openssl s_client \
  -connect "${IP_SERVIDOR}:443" \
  -servername "$DOMINIO" \
  -CAfile "$CA_CERT" \
  -verify_hostname "$DOMINIO" \
  -verify_return_error \
  -tls1_3 -brief </dev/null
```

Registra para cada conexión la versión negociada, la cipher suite y si la validación del certificado resulta satisfactoria.

> Si alguna conexión no se establece, **investiga el motivo**: la versión de OpenSSL, las políticas del cliente y la configuración del servidor pueden influir en el resultado. Un fallo del cliente por sí solo no demuestra que el servidor haya deshabilitado un protocolo.

### 16.3. Comprobar un protocolo obsoleto

Intenta conectar utilizando TLS 1.1 **solo contra tu servidor de laboratorio**:

```bash
openssl s_client \
  -connect "${IP_SERVIDOR}:443" \
  -servername "$DOMINIO" \
  -tls1_1 -brief </dev/null
```

La conexión **no se completará** con TLS 1.1. Analiza el mensaje de error y comprueba la configuración activa de Nginx para justificar la conclusión. Dependiendo de la versión y la política de OpenSSL, el cliente puede rechazar el intento antes de que exista una negociación útil.

### 16.4. Valorar los algoritmos negociados

A partir de las pruebas anteriores responde:

1. ¿Qué versión TLS se negoció en cada conexión válida?
2. ¿Qué cipher suite aparece en TLS 1.2 y en TLS 1.3?
3. ¿Qué protección aporta la cipher suite negociada?
4. ¿La cipher suite de la sesión es lo mismo que el algoritmo usado para firmar el certificado? Explica la diferencia.
5. ¿Qué comportamiento se observa al intentar usar TLS 1.1?

### 16.5. Comprobar la persistencia de la configuración

Si has modificado Nginx:

```bash
docker compose exec reverse-proxy nginx -t
docker compose exec reverse-proxy nginx -s reload
```

| Protocolo | Resultado real | ¿Cumple el requisito? |
|---|---|---|
| TLS 1.1 | | |
| TLS 1.2 | | |
| TLS 1.3 | | |

---

### Evidencias

- Fragmento de configuración activa
- Pruebas TLS 1.2 y TLS 1.3
- Prueba negativa de TLS 1.1
- Explicación de las cipher suites negociadas

---

# Actividad 17 - Observación con Suricata en HTTP y HTTPS

### 17.1. Identificar la interfaz monitorizada

Identifica la interfaz de entrada de las peticiones enviadas **desde el cliente** (por ejemplo, `ens33` o `enp0s3`). Comprueba que coincide con la interfaz que realmente monitoriza Suricata.

```bash
ip -br address
sudo systemctl status suricata --no-pager
```

### 17.2. Observar una petición HTTP

Abre dos terminales: **Terminal A (Ubuntu Server):**

```bash
sudo tcpdump -i INTERFAZ -nn -s 0 -A 'tcp port 80'
```

**Terminal B (equipo cliente):**

```bash
curl -v \
  --resolve "${DOMINIO}:80:${IP_SERVIDOR}" \
  "http://${DOMINIO}/mf0489-prueba-http" \
  -o /dev/null
```

La petición puede terminar en **301 o 308**, porque en la *Fase 3* se configuró la redirección HTTP → HTTPS.

> La prueba demuestra la visibilidad de la **petición de redirección HTTP**; no significa que las páginas de WordPress o la información protegida de la aplicación se estén entregando mediante HTTP.

### 17.3. Observar una petición HTTPS

**Terminal A (Ubuntu Server):**

```bash
sudo tcpdump -i INTERFAZ -nn -s 0 -A 'tcp port 443'
```

**Terminal B (equipo cliente):**

```bash
curl -v \
  --cacert "$CA_CERT" \
  --resolve "${DOMINIO}:443:${IP_SERVIDOR}" \
  "https://${DOMINIO}/mf0489-prueba-https" \
  -o /dev/null
```

El estado HTTP final puede ser 200, 301 o 404 según la aplicación; **lo importante es que la negociación TLS sea válida y la petición llegue a Nginx**.

En la captura deberían observarse conexiones y datos de TLS, pero **no la ruta HTTP ni su contenido en texto claro** en la interfaz externa.

Ten en cuenta que **HTTPS no oculta todo**: se siguen observando direcciones IP, puertos, tamaños, tiempos de conexión y ciertos metadatos del establecimiento TLS. El nombre SNI puede estar visible según el protocolo y las extensiones negociadas. Con TLS 1.3, además, el certificado del servidor se transporta cifrado durante el handshake, por lo que no debe suponerse que siempre será visible para Suricata.

> Una captura ASCII que no muestra una URL es una **observación útil**, pero no es una prueba matemática de cifrado por sí sola. Relaciona esa observación con la conexión TLS verificada mediante OpenSSL y `curl`.

### 17.4. Examinar los eventos de Suricata

Revisa el archivo de eventos:

```bash
sudo tail -n 30 /var/log/suricata/eve.json
```

Si tienes instalado `jq`, puedes filtrar los registros de flujo y TLS:

```bash
sudo jq -c \
  'select(.event_type == "flow" or .event_type == "tls") |
   {timestamp,event_type,src_ip,src_port,dest_ip,dest_port,tls}' \
  /var/log/suricata/eve.json | tail -n 20
```

**No todos los despliegues registran los mismos campos o tipos de eventos.** La presencia de eventos `tls` depende de la configuración de EVE y de que Suricata pueda identificar la sesión en la interfaz elegida. La ausencia de `event_type: tls` no demuestra por sí sola que el tráfico no esté cifrado o que Suricata esté inactivo.

Documenta con precisión **qué información viste realmente** y qué información no pudiste observar.

### 17.5. Tabla comparativa

| Elemento observado desde la interfaz externa | HTTP :80 | HTTPS :443 |
|---|---|---|
| IP origen y destino | | |
| Puerto de conexión | | |
| Método HTTP (`GET`, `HEAD`...) | | |
| Ruta solicitada (`/mf0489-prueba-...`) | | |
| Versión TLS / metadatos TLS | | |
| Contenido HTTP en texto claro | | |
| Eventos disponibles en Suricata | | |

---

### Evidencias

- identificación de interfaz
- Petición HTTP visible
- Petición HTTPS válida
- Capturas o fragmentos de salida
- Tabla comparativa
- Explicación de las limitaciones de visibilidad

---

# Actividad 18 - Detección HTTPS y correlación con Nginx

### 18.1. Generar una alerta de laboratorio en Suricata

Conserva las reglas existentes del MF0488. Si no tienes una regla que identifique el establecimiento de conexiones hacia el puerto 443, añade **una regla nueva propia** en tu `local.rules` activo.

La regla deberá:

- Detectar paquetes **TCP SYN** dirigidos al puerto **443** del sistema protegido.
- Utilizar `$HOME_NET` de forma coherente con la dirección de tu Ubuntu Server.
- Tener un `sid` local **no utilizado previamente**, y un mensaje reconocible de la Fase 4.
- Ser una regla de laboratorio destinada a registrar conexiones, **no una firma que afirme detectar ataques o identificar Nmap**.

Como pista puedes partir del siguiente esqueleto (deberás adaptar los campos):

```suricata
alert tcp any any -> $HOME_NET 443 (
    msg:"MF0489 - Conexion HTTPS observada";
    flags:S;
    sid:SID_LOCAL_NO_UTILIZADO;
    rev:1;
)
```

El valor `SID_LOCAL_NO_UTILIZADO` es un marcador: **sustitúyelo por un número válido**. No copies literalmente ese texto.

Verifica que el archivo está incluido en la configuración que utiliza Suricata y valida las reglas:

```bash
sudo suricata -T -c /etc/suricata/suricata.yaml
```

Después aplica el procedimiento de reinicio o recarga que corresponda a tu instalación y confirma que Suricata continúa activo.

**Importante:** un SYN hacia el puerto 443 es un evento de conexión, **no evidencia de un ataque**. Esta regla puede producir ruido en una red con actividad elevada; úsala con fines didácticos y documenta su alcance.

### 18.2. Mejorar la información del log de Nginx

Nginx ya registra peticiones web. Ahora deberás identificar, como mínimo:

- Fecha y hora con zona horaria.
- IP del cliente (según lo que realmente reciba Nginx).
- Método y ruta HTTP.
- Código de estado.
- Versión TLS negociada.
- Cipher suite.

Si el formato de logs actual no incluye TLS, investiga las variables **`$ssl_protocol`** y **`$ssl_cipher`**. Puedes crear un formato de acceso como este dentro del contexto `http` de Nginx (por ejemplo, en un archivo `default.conf` incluido por el `nginx.conf` oficial):

```nginx
log_format mf0489
  '$time_iso8601 client=$remote_addr:$remote_port '
  'request="$request" status=$status '
  'tls=$ssl_protocol cipher=$ssl_cipher';
```

Y asociarlo a tu `server` HTTPS, por ejemplo:

```nginx
access_log /var/log/nginx/access.log mf0489;
```

**Adapta la configuración al servidor que realmente tienes** y verifica que los registros llegan a Docker (en la imagen oficial de Nginx, habitualmente se encaminan a la salida del contenedor).

Valida y recarga:

```bash
docker compose exec reverse-proxy nginx -t
docker compose exec reverse-proxy nginx -s reload
```

### 18.3. Ejecutar una acción de prueba identificable

Deja visibles los registros en **dos terminales del servidor**:

**Terminal A — Suricata:**

```bash
sudo tail -f /var/log/suricata/fast.log
```

**Terminal B — Nginx:**

```bash
docker compose logs -f --tail=20 reverse-proxy
```

Desde el equipo cliente, realiza **una única petición identificable**, con una ruta que no coincida con pruebas anteriores:

```bash
curl -v \
  --cacert "$CA_CERT" \
  --resolve "${DOMINIO}:443:${IP_SERVIDOR}" \
  "https://${DOMINIO}/mf0489-correlacion-01" \
  -o /dev/null
```

La ruta puede devolver un **404**, lo cual no invalida la práctica: lo que queremos demostrar es que Nginx ha recibido una petición HTTPS y que la conexión correspondiente ha generado actividad observable.

Identifica la alerta en `fast.log`, si la regla se ha activado, y también en `eve.json`. Si necesitas filtrar por tu SID:

```bash
sudo jq -c \
  'select(.event_type == "alert" and .alert.signature_id == SID_LOCAL_NUMERICO)' \
  /var/log/suricata/eve.json
```

**Atención:** `SID_LOCAL_NUMERICO` también es un marcador; debes sustituirlo por un número en la expresión de `jq`.

### 18.4. Construir la correlación

No basta con adjuntar dos capturas distintas. Debes explicar **por qué consideras que corresponden a la misma prueba**.

Completa una tabla como esta, utilizando los datos que obtengas:

| Campo | Suricata | Nginx |
|---|---|---|
| Fecha y hora | | |
| Dirección IP del cliente, si es visible | | |
| Dirección/puerto de destino | | |
| Acción o evento | SYN hacia 443 / SID propio | `GET /mf0489-correlacion-01` |
| Resultado | Alerta de laboratorio | HTTP 200/301/404, según aplicación |
| Información TLS / HTTP observable | | |

Los registros pueden mostrar **horas expresadas en zonas distintas**. Normaliza las marcas temporales antes de comparar. Además, Docker y los mecanismos de NAT pueden modificar la IP o el puerto de origen observado por Nginx: documenta estas diferencias si aparecen; **no inventes una coincidencia**.

Explica también que la correlación **no significa que Suricata haya visto la ruta HTTPS**: el IDS relaciona conexiones y metadatos, mientras que Nginx ve la solicitud HTTP una vez descifrada.

### 18.5. Evento, alerta e incidente

Clasifica tu propia prueba:

- **Evento:** por ejemplo, una conexión TCP y una petición recibida.
- **Alerta:** la regla de Suricata se ha activado.
- **Incidente:** ¿se ha producido realmente uno?

Justifica por qué una alerta generada durante una comprobación autorizada del laboratorio **no debe clasificarse automáticamente como incidente de seguridad**.

---

### Evidencias

- Regla propia
- `suricata -T`
- Alerta real
- Petición visible en Nginx
- Tabla de correlación
- Conclusión evento/alerta/incidente.

> **No presentes un registro `flow` como si fuese un `alert`**. Investiga si Suricata observa la interfaz correcta, si `$HOME_NET` incluye la IP de destino, si la regla está cargada y si la petición realmente pasa por esa interfaz.

---

# Actividad 19 - Comprobación de las medidas de seguridad aplicadas

Después de las pruebas, verifica que **no has debilitado la infraestructura**.

### Comprobaciones

1. HTTP continúa redirigiendo a HTTPS.
2. HTTPS se verifica desde un cliente con `curl --cacert` **sin `-k`**.
3. El nombre DNS de la conexión coincide con el certificado.
4. Nginx utiliza TLS 1.2/1.3; TLS 1.1 no está habilitado.
5. La aplicación y su base de datos siguen funcionando y conservan los datos.
6. Los únicos puertos web publicados son los que exige la arquitectura; la BBDD no está expuesta innecesariamente.
7. Los certificados y la clave privada del servidor siguen instalados y con permisos apropiados; Nginx los monta **solo en lectura**.
8. `ca.key` permanece fuera de Ubuntu Server y no se ha publicado ningún secreto en GitHub.
9. Suricata continúa activo y las reglas utilizadas durante la fase están documentadas.

Comandos de apoyo:

```bash
docker compose ps
docker compose config --quiet
docker compose exec reverse-proxy nginx -t
sudo systemctl is-active suricata
sudo ls -l /etc/ssl/private/servidor.key
```

No publiques valores de variables de entorno ni el contenido de archivos de claves privadas. Si detectas contraseñas reales o claves expuestas, documenta el problema, sustituye los secretos y corrige la publicación antes de cerrar la fase.

### Evidencias

- Checklist completada
- Comprobaciones de continuidad
- Correcciones efectuadas

--- 

# [ExP 89] Fase 4 - Documentación

Actualiza GitHub para que otra persona pueda reproducir el **despliegue de Nginx y Docker Compose**, sin necesidad de disponer de tus claves privadas.

### Estructura orientativa

```text
docs/
└── mf0489/
    ├── arquitectura.md
    ├── pki.md
    ├── https-docker.md
    ├── auditoria-tls-monitorizacion.md
    └── evidencias/
        └── fase4/

nginx/
└── default.conf

suricata/
└── local.rules                 # O documentación de la regla activa
```

---

### Contenido de `auditoria-tls-monitorizacion.md`

1. Objetivo de la auditoría y arquitectura verificada.
2. Identificación de la máquina cliente, servidor e interfaz monitorizada.
3. Resultado de `curl --cacert` y `openssl s_client` contra el servicio real.
4. Versiones TLS admitidas y rechazadas; cipher suites observadas.
5. Comparación de visibilidad HTTP y HTTPS.
6. Regla de Suricata utilizada, motivo y evidencia de detección real.
7. Extractos de logs de Nginx, con fechas y estados HTTP.
8. Tabla y explicación de correlación entre dos fuentes.
9. Diferencias entre evento, alerta e incidente aplicadas a la prueba.
10. Checklist de seguridad, incidencias reales y soluciones.
11. Conclusión: **¿cómo monitorizas una aplicación sin poder leer directamente sus peticiones HTTPS desde la interfaz externa?**