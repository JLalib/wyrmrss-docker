# 🐳 WyrmRSS Docker - Lector RSS Autohospedado

[![GitHub](https://img.shields.io/badge/GitHub-wyrmrss--docker-blue?logo=github)](https://github.com/kryoseu/wyrmrss)
[![Docker](https://img.shields.io/badge/Docker-ghcr.io%2Fkryoseu%2Fwyrm--server%3Alatest-blue?logo=docker)](https://ghcr.io/kryoseu/wyrm-server)
[![License](https://img.shields.io/badge/License-GPL--3.0-orange)](https://www.gnu.org/licenses/gpl-3.0.html)

## 📋 Descripción general

**WyrmRSS** es un lector de feeds RSS autohospedado escrito en **Rust**, con arquitectura separada de **backend** (servidor API), **frontend** (interfaz web) y base de datos **PostgreSQL**. Permite suscribirse a cualquier feed RSS, ver vídeos de **YouTube** sin salir del lector y organizar los canales en **carpetas** plegables. Incluye importación/exportación **OPML** para migrar suscripciones fácilmente. Se despliega mediante **Docker Compose** en tu propio servidor.

## ✨ Características principales

- 🎬 **Canales de YouTube**: Suscríbete a un canal y mira los vídeos en el propio lector, sin salir de él.
- 📁 **Carpetas**: Agrupa los feeds en secciones plegables de la barra lateral, con contador de no leídos.
- 📦 **OPML**: Importa y exporta tus suscripciones, con las carpetas incluidas.
- 📖 **Marcado de lectura**: Elige si los artículos se marcan al abrirlos, manualmente o nunca, con retraso opcional.
- 🧹 **Limpieza automática**: Borra los artículos leídos o no leídos después de los días que configures. Los guardados para leer más tarde se conservan.
- 🔐 **API protegible**: Con `WYRM_API_KEY` limitas el acceso a la API; sin esa variable, la API queda abierta.

## 📋 Requisitos del sistema

- **Docker** y **docker-compose** (o `docker compose` v2)
- Puertos libres: **3000** (interfaz web) y **3001** (servidor API)
- Un volumen de **PostgreSQL** para los datos (incluido en el compose oficial)

## 🐳 Instalación

### Paso 1: Crea el fichero `.env`

```bash
# Contraseña de PostgreSQL. El compose oficial trae "wyrm" como valor por defecto; cámbialo
POSTGRES_PASSWORD=cambia-esta-contrasena

# Clave de la API. Opcional, pero recomendable si el puerto es accesible desde fuera
# WYRM_API_KEY=una-clave-larga-y-aleatoria
```

> **Importante**: El compose oficial usa `wyrm` como usuario y contraseña de la base de datos. Antes de arrancar, cambia la contraseña por una propia. Lo que está en el ejemplo es público.

### Paso 2: Crea el `compose.yml`

```yaml
services:
  wyrm-db:
    image: postgres:18
    container_name: wyrm-db
    environment:
      POSTGRES_USER: wyrm
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: wyrm
    volumes:
      - wyrm_data:/var/lib/postgresql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U wyrm -d wyrm -h 127.0.0.1"]
      interval: 5s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped

  wyrm-server:
    image: ghcr.io/kryoseu/wyrm-server:latest
    container_name: wyrm-rss
    environment:
      # Todas las variables son opcionales; estos son los valores por defecto
      WYRM_PORT: 3001
      WYRM_DATABASE_CONNECTION: "postgres://wyrm:${POSTGRES_PASSWORD}@wyrm-db/wyrm"
      WYRM_DATABASE_POOL_SIZE: 30
      # WYRM_API_KEY: "${WYRM_API_KEY}"
    ports:
      - "3001:3001"
    healthcheck:
      test: ["CMD", "bash", "-c", "exec 3<>/dev/tcp/127.0.0.1/3001"]
      interval: 5s
      timeout: 5s
      retries: 10
    volumes:
      # Carpeta con el fichero de configuración wyrm.toml opcional
      - ./config:/app/config
    depends_on:
      wyrm-db:
        condition: service_healthy
    restart: unless-stopped

  wyrm-ui:
    image: ghcr.io/kryoseu/wyrm-ui:latest
    container_name: wyrm-ui
    environment:
      WYRM_WEB_PORT: 3000
      WYRM_BACKEND_URL: "http://wyrm-server:3001"
      # WYRM_API_KEY: "${WYRM_API_KEY}" # debe coincidir con la del servidor
    ports:
      - "3000:3000"
    depends_on:
      wyrm-server:
        condition: service_healthy
    restart: unless-stopped

volumes:
  wyrm_data:
```

> **Sobre los volúmenes**: La base de datos usa un volumen Docker (`wyrm_data`) porque así lo hace el compose oficial. La configuración del servidor sí va en una carpeta del host (`./config`), como prefieres en el blog.

### Paso 3: Arranca los servicios

```bash
docker compose up -d

# Revisa los logs si algún servicio no pasa el healthcheck
docker compose logs -f wyrm-server
```

### Paso 4: Accede a la interfaz

Abre **http://localhost:3000** (o `http://IP_DEL_SERVIDOR:3000`). Si configuraste `WYRM_API_KEY`, la interfaz te pedirá la clave al entrar.

- **Interfaz web**: http://localhost:3000
- **API**: http://localhost:3001

## ⚙️ Configuración

1. **Variables de entorno** (`.env`): `POSTGRES_PASSWORD` (obligatorio), `WYRM_API_KEY` (opcional, recomendado para exposición externa).
2. **Fichero `wyrm.toml`** (en `./config/`): Permite poner los ajustes de arranque en lugar de variables de entorno. Las variables tienen prioridad sobre el fichero, según el README oficial.
3. **Ajustes en tiempo de ejecución** (desde la UI, página de ajustes, sin reiniciar):
   - Tamaño de página
   - Intervalo de sondeo (por defecto 900 segundos)
   - Timeouts y reintentos
   - Política de marcado de lectura
4. **Limpieza automática**: Opción de borrar posts leídos o no leídos después de X días (**irreversible**). Actívala solo cuando sepas que la copia de seguridad está en orden.

## 🚀 Primeros pasos

1. Importa tus suscripciones desde un fichero **OPML**, si ya tienes uno.
2. Crea **carpetas** para agrupar los feeds por tema.
3. Ajusta el **intervalo de sondeo** y los **días de limpieza** desde la página de ajustes.
4. Haz una **copia de seguridad** del volumen de PostgreSQL (`wyrm_data`) y de la carpeta `config`.

## 💡 Casos de uso

- 📰 **Noticias y blogs**: Todos los feeds en un lector que no depende de terceros.
- 🎬 **Canales de YouTube**: Seguir vídeos sin cuenta ni anuncios de la plataforma.
- 🔖 **Lectura para más tarde**: Guardar artículos y limpiar lo leído automáticamente.

## 🔒 Acceso remoto seguro

Para exponer WyrmRSS de forma segura a Internet:

1. **Reverse proxy** (Nginx Proxy Manager, Traefik, Caddy) con **TLS/SSL** (Let's Encrypt).
2. **Autenticación adicional**: Authelia, Authentik o Basic Auth en el proxy.
3. **Define `WYRM_API_KEY`** en `.env` y en el compose (tanto en `wyrm-server` como en `wyrm-ui`) para proteger la API.
4. **No expongas el puerto 3001** (API) directamente; solo el 3000 (UI) a través del proxy.
5. **Red Docker interna**: Los contenedores se comunican por nombre de servicio (`wyrm-server`, `wyrm-db`); no mapees puertos de base de datos al host.

## 🛠️ Gestión y mantenimiento

| Acción | Comando |
|--------|---------|
| Ver estado | `docker compose ps` |
| Ver logs (servidor) | `docker compose logs -f wyrm-server` |
| Actualizar imágenes | `docker compose pull && docker compose up -d` |
| Detener stack | `docker compose down` |
| Backup BD (volumen) | `docker run --rm -v wyrm_data:/data -v $(pwd):/backup alpine tar czf /backup/wyrm_data_$(date +%F).tar.gz -C /data .` |
| Restaurar BD | `docker run --rm -v wyrm_data:/data -v $(pwd):/backup alpine tar xzf /backup/wyrm_data_YYYY-MM-DD.tar.gz -C /data` |

## 📝 Licencia

**GNU General Public License v3.0** – Consulta el archivo [LICENSE](https://github.com/kryoseu/wyrmrss/blob/main/LICENSE) en el repositorio oficial.

---

> 📖 **Guía completa y vídeo tutorial**: [Cómo instalar WyrmRSS en Docker - Lector RSS autohospedado](https://genbyte.blogspot.com/2026/11/como-instalar-wyrmrss-en-docker-lector.html)