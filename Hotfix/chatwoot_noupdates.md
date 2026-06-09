# Chatwoot — Mensajes no se actualizan en tiempo real

**Servidor:** `vmi2621981` (app.ontime.chat)  
**Instalación:** Local en `/home/chatwoot/chatwoot`  
**Stack:** Puma + Sidekiq + Redis + Nginx

---

## Síntoma

El navegador deja de recibir mensajes nuevos en tiempo real. Al recargar la página o reiniciar el app, vuelve a funcionar. No hay error visible en el frontend.

---

## Causa raíz identificada (Junio 2026)

Los procesos Redis visibles bajo el usuario `manakinlook` en `*:6379` son en realidad **contenedores Docker legítimos** del stack:

| Contenedor | Imagen | Para |
|---|---|---|
| `n8n-redis-1` | `redis:6-alpine` | n8n |
| `evo-redis-1` | `redis:latest` | Evolution API |

Ambos tienen `"6379/tcp": null` — no están publicados al host, solo son accesibles dentro de la red Docker. No hay conflicto real con el Redis de Chatwoot.

```bash
# Confirmar que el puerto 6379 solo lo tiene el Redis legítimo
ss -tlnp | grep 6379
# Esperado: solo 127.0.0.1:6379 y ::1:6379
```

La causa real es **conexiones WebSocket que se acumulan hasta saturar Puma**. Al reiniciar, las conexiones colgadas se limpian y el problema desaparece hasta la próxima acumulación.

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

### Paso 1 — Reiniciar Chatwoot

Chatwoot corre como servicios systemd:

```bash
systemctl restart chatwoot.target
# Reinicia chatwoot-web.1.service y chatwoot-worker.1.service de una sola vez
```

### Paso 2 — Verificar que el navegador reconecta

Abrir DevTools → Network → filtrar por **WS**. Debe aparecer una conexión a `/cable` con estado `101 Switching Protocols`.

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

## Watchdog — Reinicio automático

El endpoint `/` responde `200 OK` y es el health check correcto para Chatwoot. Instalarlo como cron:

```bash
cat > /etc/cron.d/chatwoot-watchdog << 'EOF'
*/5 * * * * root curl -sf -o /dev/null http://localhost:3000/ || systemctl restart chatwoot.target
EOF

chmod 644 /etc/cron.d/chatwoot-watchdog
```

Cada 5 minutos verifica que Puma responde. Si falla, reinicia ambos services (`chatwoot-web.1` y `chatwoot-worker.1`) automáticamente vía el target.

**Endpoints probados:**

| Endpoint | Resultado |
|---|---|
| `http://localhost:3000/` | `200 OK` ✅ — usar este |
| `http://localhost:3000/health` | `404` ❌ |
| `http://localhost:3000/auth/sign_in` | `500` ❌ |

---

## Configuración de referencia

| Variable | Valor |
|---|---|
| `FRONTEND_URL` | `https://app.ontime.chat` |
| `REDIS_URL` | `redis://localhost:6379` |
| `FORCE_SSL` | `false` |
| Puerto Puma | `3000` |
| Nginx config | `/etc/nginx/sites-enabled/nginx_chatwoot.conf` |
| Redis legítimo | `/usr/bin/redis-server 127.0.0.1:6379` (usuario `redis`) |
| Services | `chatwoot-web.1.service`, `chatwoot-worker.1.service`, `chatwoot.target` |
| Redis Docker (n8n) | `n8n-redis-1` — solo red interna Docker, no publicado al host |
| Redis Docker (Evo) | `evo-redis-1` — solo red interna Docker, no publicado al host |
