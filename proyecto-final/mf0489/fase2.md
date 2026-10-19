# [ExP 89] Fase 2 - Construcción de una Infraestructura de Clave Pública (PKI)

## Resultados de aprendizaje

RA4-C4 — Implementación de mecanismos de certificación y comunicaciones seguras. Aplicación práctica de RA3-C3 — Verificación de certificados digitales.

---

En esta fase comenzarás a incorporar los mecanismos criptográficos necesarios para ofrecer comunicaciones HTTPS. Para ello deberás construir una **infraestructura de clave pública (PKI) de laboratorio**, utilizando OpenSSL.

Al finalizar todas las actividades de la **Fase 2** tendrás:

- Una autoridad certificadora (CA) propia.
- Una clave privada y un certificado raíz de la CA.
- Una clave privada para tu servidor web.
- Una solicitud de firma de certificado (CSR).
- Un certificado digital emitido por tu CA.
- Una verificación satisfactoria de la cadena de confianza y de la identidad del servidor.

---

# Actividad 2 - Preparación de entornos

Se utilizarán **dos entornos separados**. Comprueba en ambos la disponibilidad de OpenSSL `openssl version`

### Entorno A — Administración de la CA

Puede ser tu equipo con Linux/WSL o una segunda máquina virtual. Aquí se crea y administra la CA; **no debe ser la VM que posteriormente se exportará como CTF**.

Estructura de trabajo recomendada:

```text
~/ca_root-pki/ca/
├── private/
│   └── ca.key              # Clave privada de la CA
├── solicitudes/
│   └── servidor.csr        # CSR recibida del servidor
├── emitidos/
│   └── servidor.crt        # Certificado firmado
├── ca.crt                  # Certificado público de la CA
└── servidor.ext            # Extensiones del certificado
```

### Entorno B — Ubuntu Server

Es la máquina virtual creada en MF0488. Generarás inicialmente la clave y la CSR en una carpeta de trabajo del usuario:

```text
~/mf0489/
├── servidor.key            # Clave privada TEMPORAL, protegida
├── servidor.csr            # Solicitud de firma
├── servidor.crt            # Certificado recibido de la CA
└── ca.crt                  # Certificado público recibido de la CA
```

Una vez emitido el certificado, instalarás los archivos necesarios en estas **rutas definitivas**:

```text
/etc/ssl/
├── private/
│   └── servidor.key        # Clave privada del servidor (600)
├── certs/
│   └── servidor.crt        # Certificado del servidor (644)
└── mf0489/
    └── ca.crt              # Certificado público de la CA (644)
```

La CSR se conservará como documentación del proceso, pero Nginx no la necesita para funcionar. Tras comprobar que la clave está instalada correctamente en `/etc/ssl/private/`, podrás eliminar su copia temporal de `~/mf0489/`.

### Nombre DNS del proyecto

Elige un nombre relacionado con tu proyecto. Como referencia utilizaremos `portal.proyecto.test`

El sufijo `.test` está reservado para entornos de prueba. Si utilizas la convención `.thm` por el futuro laboratorio de TryHackMe, indícalo en el informe; `.thm` no es un dominio formalmente reservado por los estándares.

**Importante:** utiliza **exactamente el mismo nombre** en la CSR, el SAN del certificado, el archivo `servidor.ext` y las verificaciones. Elegir un nombre no configura automáticamente su resolución DNS; eso se preparará cuando implementes HTTPS.

---

### Requisitos de seguridad

- `ca.key` permanece **exclusivamente en el Entorno A**; no se copia a Ubuntu Server ni se publica en GitHub.
- `servidor.key` permanece **exclusivamente en el Entorno B**; tampoco se publica en GitHub.
- Pueden transferirse las CSR y los certificados públicos (`.csr` y `.crt`).

### Evidencias
- Versión de OpenSSL
- Ubicación de ambos entornos
- Nombre DNS seleccionado

---

# Actividad 3 - Generación de la CA raíz | Entorno A

### 3.1. Preparar el directorio de la CA

```bash
umask 077
mkdir -p ~/ca_root-pki/ca/{private,solicitudes,emitidos}
cd ~/ca_root-pki/ca
```

### 3.2. Generar la clave privada de la CA

Genera `private/ca.key` utilizando estas condiciones:

- Algoritmo: **RSA**.
- Longitud: **4096 bits**.
- Protección de la clave mediante **AES-256** y una contraseña.
- Acceso permitido únicamente al propietario (**permisos 600**).

Construye el comando con `openssl genpkey`. Puedes consultar la ayuda:

```bash
openssl genpkey -help
```

Después ajusta los permisos y comprueba la clave:

```bash
chmod 600 private/ca.key
openssl pkey -in private/ca.key -check -noout
```

OpenSSL te solicitará la contraseña de la clave privada.

> **Seguridad:** no incluyas la contraseña en comandos versionados, scripts, capturas ni documentos del repositorio. Los permisos `600` permiten leer y escribir únicamente al propietario; **no al grupo**.

### 3.3. Crear el certificado raíz de la CA

Crea un certificado X.509 **autofirmado** con los siguientes requisitos:

| Propiedad | Requisito |
|---|---|
| Algoritmo de firma | SHA-256 |
| Validez | Aproximadamente 10 años (3650 días) |
| Clave de firma | `private/ca.key` |
| Archivo de salida | `ca.crt` |
| País | `ES` |
| Organización | Nombre relacionado con tu proyecto |
| Common Name | Nombre identificativo de tu CA |
| `basicConstraints` | Crítico, `CA:TRUE,pathlen:0` |
| `keyUsage` | Crítico, `keyCertSign,cRLSign` |
| `subjectKeyIdentifier` | `hash` |

Para crear el certificado deberás completar el siguiente comando *(los guiones bajos son espacios para completar y no deben ejecutarse literalmente)*

```bash
openssl req ______ ______ \
  ______ \
  -days ______ \
  -key ____________________ \
  -out ____________________ \
  -subj "______________________________" \
  -addext "basicConstraints=critical,CA:TRUE,pathlen:0" \
  -addext "keyUsage=critical,___________________" \
  -addext "subjectKeyIdentifier=hash"
```

`CA:TRUE` identifica el certificado como certificado de CA; `pathlen:0` impide que esta CA emita certificados de otras CA subordinadas en una cadena válida. No impide que firme certificados de servidores.

### 3.4. Inspeccionar y verificar la CA

Investiga cómo mostrar todos los campos de `ca.crt` con `openssl x509` y localiza:

- `Subject` e `Issuer` (explica por qué coinciden).
- Fechas de validez.
- Algoritmo y tamaño de la clave pública.
- `Basic Constraints: CA:TRUE`.
- `Key Usage: Certificate Sign, CRL Sign`.

Verifica el certificado raíz utilizando expresamente esa CA como ancla de confianza:

```bash
openssl verify -CAfile ca.crt ca.crt
```

---

### Resultado esperado

`ca.crt: OK`. Este resultado **no implica** que un navegador o cualquier otro ordenador ya confíe en tu CA.


### Evidencias

- Creación de la clave (sin mostrar su contenido)
- Permisos `600`
- Campos del certificado raíz y verificación `OK`

---

# Actividad 4 - Generación de la clave privada y CSR del servidor | Entorno B — Ubuntu Server

### 4.1. Preparar el directorio temporal de trabajo

```bash
umask 077
mkdir -p ~/mf0489
cd ~/mf0489
```

No crees `/etc/certificados`. Aún se están **generando** los ficheros; después se **instalarán** en `/etc/ssl/`.

### 4.2. Generar la clave privada del servidor

Crea `servidor.key` utilizando:

- RSA de **3072 bits**.
- Archivo temporal de salida: `~/mf0489/servidor.key`.
- Permisos: **600**.

Investiga y construye el comando correspondiente con `openssl genpkey`.

La clave del servidor no tendrá contraseña en esta práctica porque Nginx deberá poder iniciar el servicio automáticamente. Su protección dependerá de los permisos, del acceso a la VM y del almacenamiento seguro.

### 4.3. Generar la CSR

Define tu nombre DNS (adapta el ejemplo):

```bash
DOMINIO="portal.proyecto.test"
```

Genera `~/mf0489/servidor.csr` completando:

```bash
openssl req -new \
  -sha256 \
  -key ____________________ \
  -out ____________________ \
  -subj "/CN=${DOMINIO}" \
  -addext "subjectAltName=DNS:${DOMINIO}"
```

El campo **Subject Alternative Name (SAN)** debe incluir el nombre que utilizarán los clientes para conectarse. Si se prevé validar el servidor por su dirección IP, se necesitaría además un SAN del tipo `IP:...`.

### 4.4. Verificar la CSR

Completa el comando:

```bash
openssl req \
  -in ____________________ \
  -text -noout -verify
```

Comprueba que:

1. La firma de la CSR es válida.
2. El `CN` y el `SAN` contienen el nombre elegido.
3. La CSR incluye la clave pública del servidor.

### 4.5. Enviar la CSR a la CA

Transfiere **solo** `servidor.csr` al directorio `solicitudes/` del Entorno A. Puedes utilizar `scp`, SFTP u otro método controlado.

Por ejemplo, si el equipo de administración admite SSH:

```bash
scp ~/mf0489/servidor.csr usuario_ca@IP_CA:~/ca_root-pki/ca/solicitudes/
```

Adapta el usuario, la IP y la ruta a tu entorno. **Nunca envíes `servidor.key` a la CA**.

---

### Evidencias

- Generación de la clave del servidor
- Permisos de la clave
- CSR creada
- Nombre DNS y SAN identificados
- CSR validada
- Explicación de por qué no se transfiere la clave privada

---

# Actividad 5 - Firma y emisión del certificado Emitir el certificado del servidor

Vuelve al **Entorno A — Administración de la CA** y sitúate en `~/ca_root-pki/ca/`.

### 5.1. Revisar la solicitud recibida

Antes de emitir el certificado, verifica la firma de la CSR y comprueba el nombre solicitado:

```bash
openssl req -in solicitudes/servidor.csr -text -noout -verify
```

**Una CA no debe firmar automáticamente cualquier solicitud:** debe comprobar que el nombre solicitado está autorizado.

### 5.2. Definir las extensiones del certificado

Crea `servidor.ext` en el directorio de la CA:

```ini
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=DNS:portal.proyecto.test
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
```

**Sustituye `portal.proyecto.test` por el nombre DNS de tu servidor.** El nombre del SAN del certificado firmado debe coincidir con el autorizado en la CSR. El archivo `servidor.ext` define las extensiones del certificado final: no basta con incluirlas únicamente en la CSR.

Analiza qué indican `CA:FALSE`, `serverAuth`, `subjectKeyIdentifier` y `authorityKeyIdentifier`.

### 5.3. Firmar la CSR

Utiliza tu CA para emitir un certificado válido durante 365 días:

```bash
openssl x509 -req \
  -in solicitudes/servidor.csr \
  -CA ca.crt \
  -CAkey private/ca.key \
  -CAcreateserial \
  -out emitidos/servidor.crt \
  -days 365 \
  -sha256 \
  -extfile servidor.ext
```

OpenSSL solicitará la contraseña de `private/ca.key`. La opción `-CAcreateserial` crea un fichero de numeración para la emisión; no es una clave privada.

### 5.4. Inspeccionar el certificado emitido

Utiliza `openssl x509` y comprueba:

- `Subject`: servidor autorizado.
- `Issuer`: tu CA.
- `Subject Alternative Name`: DNS elegido.
- `Basic Constraints: CA:FALSE`.
- `Extended Key Usage`: autenticación de servidor.
- Fechas de validez y algoritmo de firma.

### 5.5. Devolver los certificados públicos

Transfiere al **Entorno B**:

```text
emitidos/servidor.crt  →  ~/mf0489/servidor.crt
ca.crt                 →  ~/mf0489/ca.crt
```

No debes transferir `private/ca.key` ni su contraseña.

---

### Evidencias

- CSR analizada antes de firmar
- Archivo de extensiones utilizado
- Emisión satisfactoria del certificado
- Subject e Issuer correctos
- SAN incluido
- `CA:FALSE`
- Certificado transferido al servidor

---

# Actividad 6 - Instalación y verificación de la PKI

Trabaja en el **Entorno B — Ubuntu Server**.

### 6.1. Instalar los archivos en sus rutas definitivas

Los archivos deben estar previamente en `~/mf0489/`. Ejecuta:

```bash
sudo mkdir -p /etc/ssl/mf0489

sudo install -o root -g root -m 600 \
  ~/mf0489/servidor.key \
  /etc/ssl/private/servidor.key

sudo install -o root -g root -m 644 \
  ~/mf0489/servidor.crt \
  /etc/ssl/certs/servidor.crt

sudo install -o root -g root -m 644 \
  ~/mf0489/ca.crt \
  /etc/ssl/mf0489/ca.crt
```

Revisa los permisos y propietarios:

```bash
sudo ls -l /etc/ssl/private/servidor.key
ls -l /etc/ssl/certs/servidor.crt
ls -l /etc/ssl/mf0489/ca.crt
```

La clave privada debe ser accesible únicamente al usuario autorizado. El certificado de la CA **se almacena aquí para verificaciones**, pero no se incorpora automáticamente al almacén global de confianza.

### 6.2. Verificación sin confiar todavía en tu CA

Antes de incorporar la CA al almacén de confianza del sistema, prueba:

```bash
openssl verify \
  -purpose sslserver \
  -verify_hostname portal.proyecto.test \
  /etc/ssl/certs/servidor.crt
```

Sustituye el nombre DNS por el tuyo. **Si la CA no está ya instalada entre las autoridades confiables**, se espera un fallo de construcción de la cadena, como `unable to get local issuer certificate`.

Explica por qué este error no significa necesariamente que el certificado esté mal firmado.

### 6.3. Verificación correcta con la CA del laboratorio

```bash
openssl verify \
  -CAfile /etc/ssl/mf0489/ca.crt \
  -purpose sslserver \
  -verify_hostname portal.proyecto.test \
  /etc/ssl/certs/servidor.crt
```

**Resultado esperado:**

```text
/etc/ssl/certs/servidor.crt: OK
```

Este resultado confirma la cadena de confianza respecto a la CA proporcionada, el propósito de servidor y la coincidencia del nombre.

### 6.4. Provocar un fallo de identidad

Ejecuta la misma validación, pero cambia el nombre a `servidor-equivocado.test`:

```bash
openssl verify \
  -CAfile /etc/ssl/mf0489/ca.crt \
  -purpose sslserver \
  -verify_hostname servidor-equivocado.test \
  /etc/ssl/certs/servidor.crt
```

**Resultado esperado:** error `hostname mismatch` (el texto puede variar según la versión de OpenSSL).

Explica por qué un certificado puede estar firmado correctamente y, aun así, no identificar al servidor al que queremos conectarnos.

### 6.5. Verificar las fechas de validez

Investiga cómo obtener `notBefore`, `notAfter` y comprobar si el certificado ya ha caducado utilizando `openssl x509`.

### 6.6. Comprobar la correspondencia de la clave privada y el certificado

Compara los resúmenes SHA-256 de las claves públicas:

**Clave pública derivada de la clave privada instalada:**

```bash
sudo openssl pkey \
  -in /etc/ssl/private/servidor.key \
  -pubout -outform DER \
  | openssl dgst -sha256
```

**Clave pública contenida en el certificado:**

```bash
openssl x509 \
  -in /etc/ssl/certs/servidor.crt \
  -pubkey -noout \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256
```

**Los resultados deben coincidir.** No muestres ni publiques el contenido de la clave privada.

### 6.7. Completar la tabla de comprobaciones

| Comprobación | Resultado real | Interpretación |
|---|---|---|
| La CA raíz se verifica con su propio certificado | | |
| Verificación sin confiar en la CA | | |
| Certificado verificado con `-CAfile` | | |
| SAN coincide con el DNS elegido | | |
| Verificación con DNS incorrecto | | |
| Certificado vigente | | |
| Clave privada y certificado corresponden | | |

### 6.8. Conservar los archivos necesarios

Cuando hayas comprobado que `/etc/ssl/private/servidor.key` existe, tiene permisos correctos y corresponde al certificado, **elimina la copia temporal** de la clave en `~/mf0489/`:

```bash
rm ~/mf0489/servidor.key
```

Conserva el archivo `servidor.csr` como evidencia de la solicitud. Los archivos que se utilizarán en la siguiente fase serán:

```text
/etc/ssl/private/servidor.key
/etc/ssl/certs/servidor.crt
/etc/ssl/mf0489/ca.crt
```

---

### Evidencias

- Validación de confianza satisfactoria
- Validación de nombre incorrecto fallida
- Fechas del certificado
- Comparación de claves públicas
- Tabla de resultados

---

# [ExP 89] Fase 2 - Documentación

Actualiza tu repositorio GitHub con **documentación, configuraciones reproducibles y evidencias**, pero sin publicar material sensible.

### Estructura orientativa

```text
docs/
└── mf0489/
    ├── arquitectura.md
    ├── pki.md
    └── evidencias/
        └── fase2/

pki/
├── ca.crt
├── servidor.csr
├── servidor.crt
└── servidor.ext
```

Los archivos públicos del repositorio son copias para documentación. **Nginx utilizará los archivos instalados en `/etc/ssl/`**, no los que aparezcan dentro del repositorio.

### Protección de secretos

El archivo `.gitignore` deberá excluir como mínimo:

```gitignore
**/*.key
**/private/
*.p12
*.pfx
.env
```

Ten cuidado con los archivos `.pem`: pueden contener certificados públicos **o claves privadas**. Inspecciónalos antes de subirlos.

Si una clave privada se ha publicado, no basta con añadirla a `.gitignore`: deberás sustituir la clave afectada, retirar la información expuesta del historial accesible y documentar la incidencia.

### Contenido del `pki.md`

1. Arquitectura PKI y separación de los dos entornos.
2. Nombre DNS elegido.
3. Algoritmos, tamaños de claves y justificación.
4. Procedimiento de creación y verificación de la CA.
5. Creación y validación de la CSR.
6. Configuración de `servidor.ext` y emisión del certificado.
7. Instalación y permisos en Ubuntu Server.
8. Pruebas de confianza correcta e incorrecta.
9. Comprobación de clave y certificado.
10. Problemas encontrados y soluciones aplicadas.
