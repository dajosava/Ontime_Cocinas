# Chatwoot — Mensajes no se actualizan en tiempo real

**Servidor:** `vmi2621981` (app.ontime.chat)  
**Instalación:** Local en `/home/chatwoot/chatwoot`  
**Stack:** Puma + Sidekiq + Redis + Nginx

---

## Síntoma

El navegador deja de recibir mensajes nuevos en tiempo real. Al recargar la página o reiniciar el app, vuelve a funcionar. No hay error visible en el frontend.

---

## Causa raíz identificada (Junio 2026)

**Redis huérfanos levantados por el usuario `manakinlook`** corriendo en `*:6379` (todas las interfaces), compitiendo con el Redis legítimo en `127.0.0.1:6379`.

Chatwoot usa ActionCable sobre WebSocket con Redis como pub/sub backend. Cuando hay múltiples instancias de Redis en el mismo puerto, las conexiones de ActionCable pueden resolverse al Redis huérfano, que no tiene el estado de los canales, y los mensajes dejan de propagarse al cliente.

---

## Diagnóstico paso a paso

### 1. Verificar cuántas instancias de Redis hay corriendo

```bash
ps -aux | grep redis
```

**Normal (solo debe aparecer esto):**
```
redis    XXXXXX  ...  /usr/bin/redis-server 127.0.0.1:6379
```

**Anormal (problema confirmado si aparece esto):**
```
manakin+ XXXXXX  ...  redis-server *:6379
manakin+ XXXXXX  ...  redis-server *:6379
```

### 2. Verificar que Redis responde

```bash
redis-cli ping
# Esperado: PONG
```

### 3. Verificar conexiones WebSocket activas

```bash
ss -tn | grep :3000 | wc -l
```

Un número muy alto (>500) puede indicar conexiones colgadas.

### 4. Revisar estado de puma y sidekiq

```bash
ps -aux | grep -E "sidekiq|puma"
```

### 5. Revisar logs de nginx

```bash
tail -50 /var/log/nginx/chatwoot_error_443.log
tail -50 /var/log/nginx/chatwoot_access_443.log
```

---

## Fix

### Paso 1 — Matar los Redis huérfanos

```bash
# Obtener los PIDs de los Redis huérfanos
ps -aux | grep redis

# Matar los que pertenecen a manakinlook (NO el de /usr/bin/redis-server)
kill <PID1> <PID2>

# Verificar que solo quede el legítimo
ps -aux | grep redis
```

> **Nota:** Los huérfanos se vuelven a levantar solos. Ver sección "Solución permanente" abajo.

### Paso 2 — Reiniciar Chatwoot

```bash
systemctl restart chatwoot.service

# Si no tiene service, como usuario chatwoot:
sudo -u chatwoot bash -c "cd /home/chatwoot/chatwoot && bundle exec rails restart"
```

### Paso 3 — Verificar que el navegador reconecta

Abrir DevTools → Network → filtrar por **WS**. Debe aparecer una conexión a `/cable` con estado `101 Switching Protocols`.

---

## Solución permanente — Evitar que vuelvan los Redis huérfanos

Identificar qué los levanta:

```bash
# Ver el proceso padre de los Redis huérfanos
ps -p <PID_HORFANO> -o pid,ppid,user,cmd

# Revisar crontab del usuario
crontab -u manakinlook -l

# Revisar servicios de systemd del usuario
systemctl list-units --all | grep -i manakin

# Revisar su home
ls -la /home/manakinlook/
cat /home/manakinlook/.bashrc
```

Una vez identificado el script o servicio que los levanta, deshabilitarlo o eliminar la línea que llama a `redis-server`.

---

## Config de nginx (referencia)

La config en `/etc/nginx/sites-enabled/nginx_chatwoot.conf` ya tiene los headers correctos para WebSocket. No tocar a menos que se reinstale nginx.

Puntos clave que deben estar presentes:

```nginx
# En location /cable:
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_read_timeout 3600s;

# En el upstream:
keepalive 3200;
```

---

## Watchdog (opcional)

Si los reinicios manuales son frecuentes, agregar monitoreo automático:

```bash
# /etc/cron.d/chatwoot-watchdog
*/5 * * * * root curl -sf http://localhost:3000/auth/sign_in -o /dev/null || systemctl restart chatwoot.service
```

---

## Configuración de referencia

| Variable         | Valor                          |
|------------------|-------------------------------|
| `FRONTEND_URL`   | `https://app.ontime.chat`     |
| `REDIS_URL`      | `redis://localhost:6379`      |
| `FORCE_SSL`      | `false`                       |
| Puerto Puma      | `3000`                        |
| Nginx config     | `/etc/nginx/sites-enabled/nginx_chatwoot.conf` |
| Redis legítimo   | `/usr/bin/redis-server 127.0.0.1:6379` (usuario `redis`) |
