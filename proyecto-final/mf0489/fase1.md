# [ExP 89] Fase 1 - Análisis de requisitos de seguridad y selección de mecanismos criptográficos

El objetivo de esta primera fase es **actualizar el diagrama de arquitectura existente para incorporar los componentes necesarios para ofrecer HTTPS**, sin modificar todavía los servicios.

---

## Actividad 1 — Actualización del diagrama de arquitectura

Cada alumno o equipo deberá recuperar el diagrama de arquitectura realizado en el MF0488 y actualizarlo incorporando los siguientes elementos:

- **Nginx Reverse Proxy:** será el punto de entrada HTTPS a la aplicación web.
- **Certificado digital TLS:** se utilizará para autenticar el servidor y establecer comunicaciones cifradas.
- **Autoridad certificadora (CA):** entidad de laboratorio que emitirá el certificado del servidor. Deberá representarse fuera de la infraestructura que posteriormente se exportará como máquina vulnerable.
- **Comunicación HTTPS:** entre el cliente y Nginx, utilizando el puerto 443.
- **Comunicación interna:** entre Nginx, la aplicación web y la base de datos.
- **Redes Docker:** indicar qué componentes comparten cada red.
- **Suricata:** mantener su ubicación e indicar qué tráfico podrá observar.


### Entrega

Se deberá incorporar al repositorio GitHub:

1. **Diagrama de arquitectura actualizado**, con los nuevos componentes y conexiones claramente identificados.
2. **Breve explicación técnica** de dónde se implementará TLS, qué servicio gestionará los certificados y qué comunicaciones permanecerán dentro de Docker.