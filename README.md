# Outline self-hosted (sin MinIO)

Stack Docker Compose para migrar base de conocimiento de Notion/Loom a Outline.

**Servicios:** Outline (app + API), PostgreSQL (datos), Redis (caché/sesiones).
**Almacenamiento:** local en volumen Docker (`outline-data`) — sin MinIO/S3. Videos y archivos van al disco del servidor.
**Auth:** OIDC genérico (Google Workspace queda listo para activar después).

## Requisitos

- Docker + Docker Compose en el servidor.
- Puerto 3000 libre (o cambiar `OUTLINE_PORT` en `.env`).

## Arranque

```bash
# .env ya viene con secrets generados. Editar URL + OIDC antes de levantar.
docker compose up -d

# Ver logs (primer arranque corre migraciones de BD)
docker compose logs -f outline
```

Cuando el log muestre que escucha en el puerto 3000, abrir `URL` en el navegador.

## Configuración mínima antes de producción

Editar `.env`:

1. **`URL`** → dominio o IP real del servidor (ej. `http://192.168.1.50:3000` o `https://wiki.empresa.com`).
   - Si va detrás de proxy HTTPS, poner `FORCE_HTTPS=true`.
2. **OIDC** → datos del proveedor de identidad:
   - `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET`
   - `OIDC_AUTH_URI`, `OIDC_TOKEN_URI`, `OIDC_USERINFO_URI`
   - En el proveedor, registrar la **Redirect URI**: `<URL>/auth/oidc.callback`
   - `OIDC_USERNAME_CLAIM`: claim del username (`preferred_username`, `email`, etc.)

Reaplicar cambios: `docker compose up -d`.

## Videos pesados

`FILE_STORAGE_UPLOAD_MAX_SIZE=5368709120` (5 GB) en `.env`. Subir/bajar el límite en bytes según necesidad. El almacenamiento local usa el disco del host vía volumen `outline-data` — vigilar espacio.

## Activar Google Workspace después

1. Crear credenciales OAuth en Google Cloud Console.
   - Redirect URI: `<URL>/auth/google.callback`
2. En `.env`, descomentar y rellenar `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET`.
3. En `docker-compose.yml`, descomentar las dos líneas `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` del servicio `outline`.
4. `docker compose up -d`.

## Operación

```bash
docker compose ps            # estado
docker compose logs -f       # logs
docker compose down          # detener (conserva volúmenes/datos)
docker compose pull && docker compose up -d   # actualizar imagen
```

## Backup

Datos persisten en volúmenes Docker: `postgres-data`, `redis-data`, `outline-data`.

```bash
# Dump de la BD
docker compose exec postgres pg_dump -U outline outline > backup.sql

# Backup de archivos/videos
docker run --rm -v outline-setup_outline-data:/data -v ${PWD}:/backup alpine tar czf /backup/outline-files.tgz -C /data .
```
