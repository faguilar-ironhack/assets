# [ExP 89] Fase 5 - Documentación, reproducibilidad y entrega final

## Resultados de aprendizaje

Consolidación de RA1-C1, RA3-C3 y RA4-C4 mediante la defensa técnica de la implementación.

---

Durante las fases anteriores has diseñado, construido y auditado una infraestructura que utiliza **certificados digitales, HTTPS, Nginx Reverse Proxy, Docker Compose y Suricata**. La finalidad es que otra persona pueda entender qué se ha desplegado, comprobar que funciona, recuperar el servicio ante un problema y realizar los pasos necesarios para ponerlo en marcha **sin que tengas que publicar contraseñas o claves privadas**.

---

# Último check

Realiza una última comprobación de extremo a extremo.

- Actualiza el diagrama definitivo
- Localiza el despliegue que realmente estás utilizando
- Comprueba los certificados instalados
- Comprueba que los archivos de configuración son coherentes
- Revisa el procedimiento de recuperación
- Realiza una comprobación de reproducibilidad
- Comprueba que las claves privadas están protegidas
- Revisa el `.gitignore`
- Revisa qué has publicado
- Prueba ls confianza HTTPS
- Prueba de redirección HTTP → HTTPS
- Prueba funcional de la aplicación
- Prueba de configuración y monitorización

---

# Estructura orientativa a MF4089

```text
proyecto/
├── README.md
├── docker-compose.yml
├── .env.example
├── .gitignore
├── nginx/
│   └── default.conf
├── scripts/
│   └── ...                       # Comprobaciones o instalación sin secretos
├── suricata/
│   └── local.rules               # Si se utiliza y puede publicarse
├── pki/
│   ├── ca.crt                    # Público
│   ├── servidor.crt              # Público
│   ├── servidor.csr              # Público
│   └── servidor.ext              # Extensiones de emisión
└── docs/
    └── mf0489/
        ├── fase1-arquitectura.md
        ├── fase2-pki.md
        ├── fase3-https-docker.md
        ├── fase4-auditoria-tls-monitorizacion.md
        ├── fase5-entrega-final.md
        ├── guia-despliegue.md
        ├── arquitectura-final.png
        └── evidencias/
            ├── fase2/
            ├── fase3/
            ├── fase4/
            └── fase5/
```

**No deben aparecer** `ca.key`, `servidor.key`, `.env` real, copias de seguridad privadas ni exportaciones de la VM en este repositorio.

---

# .Gitignore orientativo a MF0489

Como referencia, deberá excluir los archivos sensibles que existan en tu proyecto.

```gitignore
.env
**/.env
**/*.key
**/private/
*.p12
*.pfx
backups/
*.sql
*.sql.gz
```

El archivo `.env.example` **sí debe publicarse** y solo contendrá valores de ejemplo, nunca credenciales reales.

> **Importante:** `.gitignore` evita nuevas incorporaciones accidentales, pero **no retira archivos que ya hayan sido versionados**.