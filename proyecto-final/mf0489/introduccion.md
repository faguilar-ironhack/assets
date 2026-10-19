# [ExP 89] Introducción

## Evolución de la infraestructura

```text
ESTADO INICIAL
Ubuntu Server
├── Suricata
├── SSH
└── Docker
    ├── Aplicación web (HTTP)
    └── Base de datos
             │
             ▼
           MF0489
             │
    ┌────────┴─────────┐
    │                  │
    ▼                  ▼
CA de laboratorio    Nginx
    │                  │
    └── Certificado ───┤
                       ▼
                    HTTPS
                       │
                       ▼
                Aplicación web
                       │
                       ▼
                  Base de datos
```

## ¿Qué se trabaja?

| Competencias             | Evidencias obligatoria                                                             |
| ------------------------ | ---------------------------------------------------------------------------------- |
| Evaluación criptográfica | Justificación de HTTP frente a HTTPS y selección de TLS                            |
| Certificados digitales   | Interpretación de Subject, Issuer, fechas, clave pública, SAN y algoritmos         |
| Verificación             | Comprobación de certificados mediante OpenSSL                                      |
| Autoridad certificadora  | CA propia creada con su clave privada protegida                                    |
| Emisión de certificados  | Clave del servidor, CSR y certificado firmado                                      |
| Cadena de confianza      | Verificación satisfactoria con la CA del laboratorio                               |
| HTTPS                    | Nginx configurado para publicar la aplicación mediante TLS                         |
| Validación del servicio  | Conexión HTTPS comprobada verificando el certificado, sin depender de `curl -k`    |
| Integración              | Docker Compose actualizado y funcionando                                           |
| Monitorización           | Comprobación de registros web y visibilidad de Suricata después de habilitar HTTPS |
| GitHub                   | Configuraciones y documentación reproducibles, sin claves privadas publicadas      |
