# Actualización de Chatwoot self-hosted a v4.16.2- Ontime Cocinas - Mexico

**Servidor:** vmi2621981
**Fecha:** 27–28 de julio de 2026
**Instalación:** nativa (no Docker), gestionada con `cwctl`
**Versión previa:** v4.8.0 (con `cwctl` v3.5.0)
**Versión final:** v4.16.2

---

## 1. Diagnóstico inicial

Se verificó la versión de la herramienta de gestión y los procesos activos:

```bash
cwctl --version
# cwctl v3.5.0

ps -aux | grep -E "sidekiq|puma"
```

Se confirmó que Chatwoot corre de forma **nativa** (Puma + Sidekiq como procesos del sistema), no en contenedores Docker.

---

## 2. Backup de la base de datos

Como la instalación no usa Docker, el backup se hizo directamente contra PostgreSQL local:

```bash
sudo -u postgres pg_dumpall -c > ~/chatwoot_backup_$(date +%F).sql
```

Verificación del archivo generado:

```bash
ls -lh ~/chatwoot_backup_*.sql
tail -n 5 /root/chatwoot_backup_2026-07-28.sql
```

Resultado: backup de **178 MB**, completo (termina con `PostgreSQL database cluster dump complete`).

---

## 3. Primer intento de actualización

```bash
sudo cwctl --upgrade
```

`cwctl` detectó que Chatwoot v4.0+ requiere soporte de **pgvector** en PostgreSQL:

```
Chatwoot v4.0 and above requires pgvector support in PostgreSQL.
Does your postgres support pgvector and want to proceed with the upgrade? [y/N]: y
```

El proceso se abortó porque detectó **cambios de código locales** (personalizaciones) en:

- `app/services/facebook/send_on_facebook_service.rb`
- `config/installation_config.yml`
- `public/brand-assets/logo.svg`
- `public/brand-assets/logo_dark.svg`

```
Custom code changes detected. Aborting update.
Please proceed to update manually.
```

---

## 4. Verificación de pgvector

Se confirmó que la extensión ya estaba instalada en la base:

```bash
sudo -u postgres psql -d chatwoot_production -c "SELECT * FROM pg_extension WHERE extname = 'vector';"
```

Resultado: `vector` versión `0.6.0` instalada correctamente. No se necesitó ninguna acción adicional.

---

## 5. Revisión de cambios locales (git)

Se corrigió primero un error de permisos de Git:

```bash
git config --global --add safe.directory /home/chatwoot/chatwoot
```

Luego se inspeccionaron los cambios detectados:

```bash
git status
git diff app/services/facebook/send_on_facebook_service.rb
git diff config/installation_config.yml
git diff public/brand-assets/logo.svg public/brand-assets/logo_dark.svg
```

**Resumen de las personalizaciones encontradas:**

| Archivo | Cambio |
|---|---|
| `send_on_facebook_service.rb` | `messaging_type` cambiado de `MESSAGE_TAG` / `ACCOUNT_UPDATE` a `RESPONSE` |
| `installation_config.yml` | `INSTALLATION_NAME` cambiado de `'Chatwoot'` a `'OnTime Chat'` |
| `logo.svg` / `logo_dark.svg` | Logo de marca personalizado |

También se detectaron archivos sueltos sin versionar (backups manuales, `.bkp`, `.save`, `.bak`) que no representaban riesgo para la actualización.

---

## 6. Guardar cambios locales con `git stash`

Para poder actualizar sin perder las personalizaciones:

```bash
git stash push -m "custom branding + facebook fix antes de v4.16.2" -- \
  app/services/facebook/send_on_facebook_service.rb \
  config/installation_config.yml \
  public/brand-assets/logo.svg \
  public/brand-assets/logo_dark.svg
```

---

## 7. Segundo intento de actualización — error de repositorios APT

Al reintentar `sudo cwctl --upgrade`, el script intentó actualizar Node.js a v24.x, pero `apt update` falló por repositorios rotos ajenos a Chatwoot:

- Clave GPG faltante de Yarn (`NO_PUBKEY 62D54FD4003F6525`)
- Repositorio de RabbitMQ/Erlang devolviendo `404` y `502` (paquete no instalado, repos huérfanos)

### 7.1 Corrección de la clave GPG de Yarn

```bash
curl -fsSL https://dl.yarnpkg.com/debian/pubkey.gpg | sudo gpg --dearmor -o /usr/share/keyrings/yarnkey.gpg

echo "deb [signed-by=/usr/share/keyrings/yarnkey.gpg] https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
```

### 7.2 Confirmar que RabbitMQ no está instalado

```bash
sudo systemctl status rabbitmq-server 2>/dev/null
dpkg -l | grep -i rabbitmq
```

Sin resultados → seguro deshabilitar esos repositorios sin afectar ningún servicio activo.

### 7.3 Deshabilitar repositorios de RabbitMQ/Erlang

```bash
sudo mv /etc/apt/sources.list.d/erlang.list /etc/apt/sources.list.d/erlang.list.disabled
sudo mv /etc/apt/sources.list.d/rabbitmq-erlang.list /etc/apt/sources.list.d/rabbitmq-erlang.list.disabled
sudo mv /etc/apt/sources.list.d/rabbitmq.list /etc/apt/sources.list.d/rabbitmq.list.disabled
```

Un archivo (`rabbitmq-erlang.list`) no se renombró correctamente en el primer intento; se detectó con:

```bash
grep -rl "packagecloud.io/rabbitmq" /etc/apt/sources.list /etc/apt/sources.list.d/
```

y se eliminó directamente:

```bash
sudo rm /etc/apt/sources.list.d/rabbitmq-erlang.list
```

### 7.4 Confirmación

```bash
sudo apt update
```

Resultado: sin errores, `77 packages can be upgraded`.

---

## 8. Actualización exitosa

```bash
sudo cwctl --upgrade
```

El proceso completó correctamente:

- ✅ Redis ya en versión compatible (8.0.2), sin cambios necesarios
- ✅ Node.js actualizado a v24.x
- ✅ Build de assets con Vite completado (~1m 45s)
- ✅ Todas las migraciones de base de datos aplicadas sin errores (decenas de migraciones desde v4.8.0 hasta v4.16.2: nuevas tablas, índices, columnas para features como Calls, Captain AI, Data Imports, etc.)
- ✅ "Updating full deployment" — reinicio de servicios

---

## 9. Verificación post-actualización

```bash
sudo systemctl status chatwoot-web chatwoot-worker
ps -aux | grep -E "sidekiq|puma"
curl -I http://localhost:3000
```

Confirmado: Chatwoot v4.16.2 corriendo correctamente, todos los servicios activos.

---

## 10. Pendiente: recuperar personalizaciones

Paso final, aún por ejecutar, para restaurar el logo, el nombre de instalación y el fix de Facebook guardados en el paso 6:

```bash
git stash list
git stash pop
```

Si aparece conflicto en `send_on_facebook_service.rb`, revisar si el fix (`messaging_type: 'RESPONSE'`) sigue siendo necesario en la nueva versión, ya que Meta deprecó `MESSAGE_TAG`/`ACCOUNT_UPDATE` hace tiempo y es posible que el proyecto ya lo haya corregido oficialmente en versiones recientes.

---

## Resumen de comandos clave (orden completo)

```bash
# 1. Backup
sudo -u postgres pg_dumpall -c > ~/chatwoot_backup_$(date +%F).sql

# 2. Permisos de git
git config --global --add safe.directory /home/chatwoot/chatwoot

# 3. Guardar personalizaciones
git stash push -m "custom branding + facebook fix antes de v4.16.2" -- \
  app/services/facebook/send_on_facebook_service.rb \
  config/installation_config.yml \
  public/brand-assets/logo.svg \
  public/brand-assets/logo_dark.svg

# 4. Arreglar repos APT
curl -fsSL https://dl.yarnpkg.com/debian/pubkey.gpg | sudo gpg --dearmor -o /usr/share/keyrings/yarnkey.gpg
echo "deb [signed-by=/usr/share/keyrings/yarnkey.gpg] https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
sudo rm /etc/apt/sources.list.d/rabbitmq-erlang.list
sudo mv /etc/apt/sources.list.d/erlang.list /etc/apt/sources.list.d/erlang.list.disabled
sudo mv /etc/apt/sources.list.d/rabbitmq.list /etc/apt/sources.list.d/rabbitmq.list.disabled
sudo apt update

# 5. Actualizar Chatwoot
sudo cwctl --upgrade

# 6. Verificar
sudo systemctl status chatwoot-web chatwoot-worker
curl -I http://localhost:3000

# 7. Recuperar personalizaciones (pendiente)
git stash pop
```

---

## Notas para futuras actualizaciones

- Los repositorios de RabbitMQ/Erlang quedaron deshabilitados (`.disabled`), no eliminados — reactivar si algún día se instala RabbitMQ.
- Mantener los backups de base de datos al menos unos días después de cada actualización.
- Los cambios locales en `send_on_facebook_service.rb`, `installation_config.yml` y los logos volverán a bloquear futuras actualizaciones automáticas de `cwctl` — repetir el flujo de `git stash` / `git stash pop` en cada upgrade, o considerar mover estas personalizaciones a un mecanismo más sostenible (variables de entorno donde sea posible, o un fork/patch documentado).
