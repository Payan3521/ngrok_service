# 🚇 ngrok Tunnel Service

> Túnel seguro ngrok contenerizado para exponer servicios locales a internet. Configuración por variables de entorno, lista para desarrollo y pruebas de webhooks.

---

## 1. Tecnologías

| Capa | Tecnología | Versión / Nota |
|------|-----------|----------------|
| Túnel | ngrok | `latest` (Docker Hub `ngrok/ngrok`) |
| Orquestación | Docker Compose | v3.8 |
| Configuración | `.env` + `.env.example` | Variables de entorno |
| Red | `host.docker.internal` | Acceso a puertos del host desde el contenedor |

---

## 2. Requisitos previos

- **Docker** ≥ 20.10 y **Docker Compose** ≥ 2.0 (o `docker compose` plugin)
- **Cuenta ngrok** (gratis) → obtén tu **Authtoken** en: https://dashboard.ngrok.com/get-started/your-authtoken
- Servicio local corriendo en el puerto que definas en `APP_PORT` (ej: `3000`, `8080`, `5000`, etc.)

> ⚠️ **Importante**: El servicio local debe estar accesible en `localhost:APP_PORT` en la **máquina host** (no dentro de otro contenedor). ngrok usa `host.docker.internal` para alcanzarlo.

---

## 3. Configuración rápida

```bash
# 1. Clona / entra al proyecto
cd /home/payan/orca/projects/ngrok_service

# 2. Copia la plantilla y edita tus valores
cp .env.example .env
# Edita .env con tu editor favorito
nano .env   # o code .env, vim .env, etc.

# 3. Variables obligatorias en .env:
#    NGROK_AUTHTOKEN=tu_token_aqui
#    APP_PORT=3000          # puerto de tu app local
#    NGROK_DOMAIN=          # opcional: dominio reservado (gratis en dashboard ngrok)

# 4. Levanta el túnel
docker compose up -d

# 5. Ver la URL pública asignada
docker compose logs -f ngrok_tunnel
# Busca líneas tipo: "started tunnel url=https://abc123.ngrok-free.app"
```

---

## 4. URLs importantes

| Recurso | URL |
|---------|-----|
| **Interfaz web ngrok (inspector)** | http://localhost:4040 |
| **API REST de túneles** | http://localhost:4040/api/tunnels |
| **Tu app pública** | La que muestra `docker compose logs` (ej: `https://abc123.ngrok-free.app`) |

---

## 5. Variables de entorno (`.env`)

| Variable | Obligatoria | Default | Descripción |
|----------|-------------|---------|-------------|
| `NGROK_AUTHTOKEN` | ✅ Sí | — | Token de tu cuenta ngrok |
| `APP_PORT` | ✅ Sí | `3000` | Puerto local de tu servicio a exponer |
| `NGROK_DOMAIN` | No | (aleatorio) | Dominio reservado (gratis en dashboard) o dominio propio (pago) |

### Dominios reservados gratis (recomendado)
1. Ve a https://dashboard.ngrok.com/domains
2. Clica **New Domain** → elige subdominio `*.ngrok-free.app`
3. Copia el dominio (ej: `mi-proyecto.ngrok-free.app`)
4. Ponlo en `.env`: `NGROK_DOMAIN=mi-proyecto.ngrok-free.app`

**Ventaja**: La URL **no cambia** entre reinicios → ideal para webhooks fijos.

---

## 6. Comandos de operación

```bash
# Iniciar en segundo plano
docker compose up -d

# Ver logs en tiempo real (para coger la URL pública)
docker compose logs -f ngrok_tunnel

# Ver solo la URL pública actual
curl -s http://localhost:4040/api/tunnels | jq -r '.tunnels[0].public_url'

# Reiniciar túnel (cambia URL si no usas NGROK_DOMAIN)
docker compose restart ngrok_tunnel

# Parar y eliminar contenedor
docker compose down

# Ver estado y healthcheck
docker compose ps
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"

# Entrar al contenedor (debug)
docker compose exec ngrok_tunnel sh
```

---

## 7. ⚠️ Aviso del navegador (Free Tier)

Al entrar a la URL pública **por primera vez**, ngrok muestra una página de advertencia:

> "You are about to visit: excavate-public-strangely.ngrok-free.dev ... This website is served for free through ngrok.com."

### Opciones para evitarla:

| Método | Funciona en Free Tier | Notas |
|--------|----------------------|-------|
| **Dominio reservado `*.ngrok-free.app`** | ✅ Sí | **Recomendado**. Configura `NGROK_DOMAIN` en `.env`. El aviso sale **una sola vez** por navegador/dispositivo. |
| **Header `ngrok-skip-browser-warning`** | ⚠️ Parcial | Debe enviarlo el **cliente** (curl, Postman, tu frontend). No se puede forzar desde el servidor. |
| **User-Agent personalizado** | ⚠️ Parcial | Ya incluido en el compose (`--host-header=rewrite` + UA custom). Reduce pero no elimina siempre. |
| **Plan pago (custom domain + edge)** | ✅ Completo | Elimina el aviso totalmente. |

### Para desarrollo de webhooks / APIs (sin navegador):
```bash
# curl salta el aviso automáticamente
curl https://tu-dominio.ngrok-free.app/tu-endpoint

# Postman / Insomnia / tu app: funciona directo
# El aviso SOLO afecta a navegadores web interactivos
```

---

## 8. Estructura del proyecto

```
ngrok_service/
├── docker-compose.yml      # Orquestación del túnel
├── .env                    # Tus valores reales (NO commitear)
├── .env.example            # Plantilla para el equipo (SÍ commitear)
├── .gitignore              # Ignora .env, logs, archivos de IDE
└── README.md               # Esta documentación
```

---

## 9. Casos de uso típicos

| Caso | Configuración recomendada |
|------|---------------------------|
| **Probar webhook de Stripe/GitHub/Telegram** | `NGROK_DOMAIN=mi-webhook.ngrok-free.app` → URL fija para registrar en el proveedor |
| **Compartir demo rápida con cliente** | Sin `NGROK_DOMAIN` → URL aleatoria, dura mientras el contenedor esté arriba |
| **Desarrollo móvil (React Native, Flutter)** | `APP_PORT=8081` (Metro) o `19000` (Expo) + `NGROK_DOMAIN` fijo |
| **Testear OAuth callbacks** | Dominio reservado + `http://localhost:4040` para inspeccionar requests |

---

## 10. Troubleshooting

### "Tunnel not found" / URL 404
```bash
# 1. Verifica que tu app local responde en el puerto
curl http://localhost:${APP_PORT}/health   # o tu endpoint

# 2. Verifica logs de ngrok
docker compose logs ngrok_tunnel | tail -30

# 3. Revisa el inspector web
open http://localhost:4040
```

### "ngrok agent failed to start" / Error de authtoken
```bash
# Token inválido o expirado
# 1. Ve a https://dashboard.ngrok.com/get-started/your-authtoken
# 2. Copia el NUEVO token
# 3. Actualiza .env y reinicia
docker compose down && docker compose up -d
```

### Puerto `APP_PORT` ya en uso / "address already in use"
```bash
# Cambia APP_PORT en .env a otro libre
# O mata el proceso que lo usa:
lsof -ti:${APP_PORT} | xargs kill -9
```

### Healthcheck falla
```bash
# El healthcheck consulta la API interna de ngrok (puerto 4040)
# Si falla, ngrok no está listo. Espera ~10s tras 'up -d'
docker compose logs ngrok_tunnel
```

### Acceder desde otro contenedor Docker (no host)
> `host.docker.internal` **solo funciona desde contenedor → host**.  
> Si tu app corre en **otro contenedor**, usa la red Docker compartida:

```yaml
# En docker-compose.yml de TU APP (no de ngrok):
services:
  mi_app:
    # ...
    networks: [red_compartida]

  ngrok_tunnel:
    # ...
    networks: [red_compartida]
    command:
      - "http"
      - "mi_app:${PUERTO_INTERNO}"   # nombre del service, no host.docker.internal
    # ...

networks:
  red_compartida:
    external: true
```

---

## 11. Seguridad y buenas prácticas

- **NUNCA** commitees `.env` (contiene `NGROK_AUTHTOKEN`). Está en `.gitignore`.
- Usa **dominios reservados gratis** (`*.ngrok-free.app`) para URLs estables sin pagar.
- El **inspector web** (`:4040`) expone headers, bodies y queries → **no lo expongas a internet** (solo `localhost`).
- Para producción real: usa **ngrok edges** con dominios propios + WAF + OAuth (planes pagos).
- Rotación de token: Si el token se filtra, revócalo en dashboard y genera uno nuevo.

---

## 12. Referencias

- Docs oficiales ngrok Docker: https://ngrok.com/docs/guides/docker/
- Dashboard ngrok (token, dominios, edges): https://dashboard.ngrok.com/
- API Inspector local: http://localhost:4040
- Precios y límites free tier: https://ngrok.com/pricing