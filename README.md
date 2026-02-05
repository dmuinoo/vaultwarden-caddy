# vaultwarden-caddy
Se trata de un dockercompose para montar de forma automatica VaultWarden con Caddy como proxy https
=======
# Vaultwarden + Caddy (Reverse Proxy TLS) — Stack README

Este stack despliega **Vaultwarden** (servidor compatible Bitwarden) detrás de **Caddy** como **reverse proxy TLS**. Está pensado para uso en VM o servidor, con persistencia y un enfoque “production-friendly” (certificados automáticos, headers, compresión, límites razonables y backups).

## Este stack es para montar automaticamente un sistema de gestion de llaves, Vaultwarden (Es un fork opensource de Bitwarden):

Vaultwarden es una implementación ligera del servidor de Bitwarden escrita en Rust.

    * Características arquitectónicas:
    * Servidor centralizado (auto-hosted)
    * API REST compatible con clientes Bitwarden
    * Base de datos (SQLite/PostgreSQL)
    * Acceso vía web + apps móviles + extensiones
    * Sincronización en tiempo real

🔐 Seguridad

    * Cifrado end-to-end (modelo Bitwarden)
    * Zero-knowledge encryption
    * 2FA integrado (TOTP, WebAuthn)
    * Control de acceso multiusuario
    * Logs y auditoría
    * Compartición segura de secretos
    * Políticas organizativas

👥 Multiusuario y colaboración

 Ventaja clara aquí:
    * Organizaciones
    * Colecciones compartidas
    * Gestión de permisos
    * Equipos
    * Sincronización automática
   👉 Diseñado para equipos.

🌐 Acceso multiplataforma

    * Clientes oficiales Bitwarden:
    * Web
    * iOS / Android
    * Windows / macOS / Linux
    * Extensiones de navegador
    * Todo sincronizado.

🚀 Despliegue y operación

  Requiere:
     * Docker / servidor
     * Backup de base de datos
     * HTTPS
     * Gestión de certificados
     * Hardening

👉 Es básicamente un servicio de contraseñas en red.

> Nota: Vaultwarden puede usar **SQLite** o **PostgreSQL**. Para alta disponibilidad (HA) real y múltiples instancias, **PostgreSQL** es lo recomendado.

---

## 1) Componentes

### Vaultwarden
- API web + UI (web vault)
- Opcional: WebSockets para notificaciones en tiempo real
- Persistencia: `vw-data` (adjuntos, configuración, base de datos si usas SQLite)

### Caddy
- Reverse proxy TLS (Let’s Encrypt) o TLS interno (si es lab)
- Redirección HTTP→HTTPS
- Termina TLS y reenvía a Vaultwarden
- Puede añadir headers de seguridad, gzip/zstd, límites de subida, etc.

---

## 2) Puertos / Endpoints

- **80/TCP** → Caddy (HTTP) → redirect a HTTPS
- **443/TCP** → Caddy (HTTPS) → Vaultwarden
- **(interno)** Vaultwarden:
  - `8080` (HTTP app)
  - `3012` (WebSocket notifications) si lo habilitas

Rutas típicas:
- `https://<TU_DOMINIO>/` → Web Vault + API
- `wss://<TU_DOMINIO>/notifications/hub` → WebSocket (si aplica)

---

## 3) Variables importantes (Vaultwarden)

Recomendadas:
- `DOMAIN=https://<TU_DOMINIO>`  
- `WEBSOCKET_ENABLED=true` (si quieres notificaciones)
- `SIGNUPS_ALLOWED=false` (en producción suele ser “false”)
- `ADMIN_TOKEN=<token-largo>` (panel admin en `/admin`)

Opcionales (correo, 2FA, etc.):
- SMTP: `SMTP_HOST`, `SMTP_FROM`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_PORT`, etc.

---

## 4) Persistencia (volúmenes)

Debes persistir:
- Vaultwarden:
  - `/data` → adjuntos, config y DB si SQLite
- Caddy:
  - `/data` y `/config` → certificados, estado y configuración

Backups mínimos recomendados:
- Si **SQLite**: backup del fichero DB (con cuidado) + `/data/attachments`
- Si **PostgreSQL**: `pg_dump` + `/data/attachments`

---

## 5) Seguridad mínima (recomendado)

- **Deshabilitar registros** (SIGNUPS_ALLOWED=false) salvo que sea un entorno controlado.
- Activar WebSocket solo si lo necesitas.
- Forzar HTTPS con Caddy.
- Restringir `/admin` (ideal: IP allowlist o BasicAuth delante con Caddy).
- Rotar `ADMIN_TOKEN` si sospechas exposición.
- Usar un dominio real y certificados válidos.

---

## 6) Operación

### Arranque / parada
- Si usas Docker Compose:
  - `docker compose up -d`
  - `docker compose logs -f`
  - `docker compose down`

### Verificación rápida
- `curl -I https://<TU_DOMINIO>`
- Comprueba que hay `200/302` y que el certificado es correcto.
- Si WebSocket: prueba login desde cliente y revisa notificaciones.

---

## 7) HA: Vaultwarden “real” = múltiples instancias + DB compartida

**Vaultwarden en HA (2 o más réplicas)** requiere:
- DB **PostgreSQL** (no SQLite local)
- Un mecanismo de “shared storage” o estrategia para adjuntos (NFS/S3/replicación) si hay múltiples pods/containers
- Load balancing delante (Caddy puede, pero normalmente usas LB / reverse proxy adicional)

---

# 8) Por qué se recomienda WAL (archiving) con 2 nodos vs “streaming” (síncrono) que suele pedirse con 3 nodos

Aquí hablamos de **replicación PostgreSQL**. Dos conceptos:

## A) Streaming replication (asíncrona o síncrona)
- **Asíncrona**: el primario confirma commits sin esperar al standby.
  - ✅ Alta disponibilidad del primario
  - ⚠️ Riesgo de perder los últimos commits si el primario muere antes de que el standby los reciba (RPO > 0)

- **Síncrona**: el primario **espera** confirmación del standby para confirmar commits.
  - ✅ Casi cero pérdida (RPO ~ 0)
  - ❌ Con **solo 2 nodos** (1 primario + 1 standby) es frágil: si el standby cae o se corta el enlace, el primario puede quedarse **bloqueado** (impacto fuerte en disponibilidad).

### ¿Por qué se habla de 3 nodos?
Porque con **3 nodos** (1 primario + 2 standby) puedes configurar confirmación “quórum/ANY 1”:
- el primario confirma si **al menos uno** de los dos standby confirma
- esto mantiene RPO muy bajo *y* reduce el riesgo de bloqueo por caída de un standby

En práctica:
- **Síncrono con 2 nodos** = disponibilidad peor (te puede parar escrituras).
- **Síncrono con 3 nodos** = puedes lograr *durabilidad* con menos penalización de disponibilidad.

## B) WAL archiving (archivado de WAL) — recomendado incluso con 2 nodos
WAL archiving no “compite” con streaming: **lo complementa**.

Qué te aporta:
- **PITR (Point-In-Time Recovery)**: restaurar a un momento exacto
- Backups consistentes (base backup + WAL)
- Recuperación aunque el standby esté atrasado o roto
- Un “seguro” independiente del estado del standby

Con **2 nodos**, la combinación más común y robusta es:
- Streaming **asíncrono** (para tener un standby usable)
- + **WAL archiving** (para garantizar recuperación y minimizar pérdida, con mejor operabilidad que forzar síncrono)

### Idea clave (2 nodos):
- Si fuerzas streaming **síncrono**, mejoras RPO pero empeoras mucho la disponibilidad.
- Si usas streaming **asíncrono** + **WAL archiving**, mantienes disponibilidad y mejoras tu capacidad real de recuperación (PITR), aceptando que el RPO no es estrictamente cero.

---

## Recomendación práctica según número de nodos (PostgreSQL)

### 1 nodo (lab)
- Sin replicación
- Backups periódicos (pg_dump) y/o basebackup

### 2 nodos (mínimo serio, sin “quórum”)
- Streaming **asíncrono**
- **WAL archiving** a almacenamiento fiable (disco externo, NFS, S3, etc.)
- Failover manual o semi-automático (cuidado con split-brain)

### 3 nodos (serio con HA + menor pérdida)
- Streaming **síncrono** con “ANY 1” (quórum de confirmación)
- WAL archiving igualmente (PITR y recuperación completa)
- Failover automatizable con menor riesgo (Patroni + etcd/consul, por ejemplo)

---

## 9) Checklist de backups (mínimo)

- DB:
  - `pg_dump` diario (o más frecuente)
  - WAL archiving si buscas PITR
- Vaultwarden:
  - `/data/attachments`
  - claves/configuración (si aplica)
- Caddy:
  - `/data` y `/config` (certs/estado) si quieres restauración rápida

---

## 10) Troubleshooting rápido

- 502/504 en Caddy:
  - Vaultwarden caído o puerto incorrecto
  - DNS/DOMAIN mal configurado
- Login OK pero no llegan notificaciones:
  - habilitar `WEBSOCKET_ENABLED=true`
  - proxy de `/notifications/hub` como WebSocket
- `/admin` expuesto:
  - proteger con allowlist o BasicAuth delante del proxy
e
