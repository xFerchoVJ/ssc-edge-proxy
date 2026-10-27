# ssc-edge-proxy

nginx compartido del VPS. Es el único servicio que publica el puerto 80; enruta
por subdominio a cada sistema a través de la red Docker externa `edge`.
El TLS lo termina redes antes del VPS, así que aquí todo es HTTP.

```
redes (HTTPS) → VPS:80 → ssc-edge-proxy (nginx)
                          ├─ empresas.sscqroo.gob.mx       → empresas-app:80
                          └─ comercioseguro.sscqroo.gob.mx → comercioseguro-web:3000
```

## Estructura

| Ruta | Qué hace |
|---|---|
| `docker-compose.yml` | Servicio `nginx`, red `edge` externa |
| `nginx/conf.d/00-default.conf` | Resolver DNS de Docker, mapa WebSocket, `default_server` que cierra hosts desconocidos (444) |
| `nginx/conf.d/<app>.conf` | Un archivo por sistema |
| `nginx/snippets/proxy.conf` | Headers de proxy + WebSocket (ActionCable), compartido |
| `brand/` | Logos de la página de mantenimiento (`/maintenance-assets/`) |

Cada app conserva en **su propio repo** `nginx/maintenance/` (`maintenance.html` +
el flag `on` que crea/borra su deploy). El edge lo monta en
`/etc/nginx/maintenance/<app>`.

## Contrato con cada app

1. Su servicio web se une a la red externa `edge` con un **alias único**
   (`empresas-app`, `comercioseguro-web`). No uses el nombre de servicio
   (`app`, `web`): dos proyectos pueden llamarlo igual y colisionar en `edge`.
2. No publica puertos al host.
3. Su workflow de deploy asegura la red: `docker network inspect edge || docker network create edge`.

## Agregar un sistema nuevo

1. Pedir a redes el subdominio apuntando al VPS (que conserven el header `Host`).
2. En la app: unir el servicio web a `edge` con alias `<nombre>-web`, quitar `ports`, agregar `nginx/maintenance/` y el flag `/nginx/maintenance/on` al `.gitignore`.
3. Aquí: copiar `nginx/conf.d/comercioseguro.conf` a `<nombre>.conf` y cambiar `server_name`, `set $upstream` (alias:puerto) y las dos rutas `/etc/nginx/maintenance/<nombre>`.
4. En `docker-compose.yml` agregar el volumen `${<NOMBRE>_DIR:-/opt/<repo>}/nginx/maintenance:/etc/nginx/maintenance/<nombre>:ro` y la variable en `.env.example`.
5. Push a `main`: el workflow valida (`nginx -t`) y recarga.

## Deploy

Push a `main` → runner self-hosted en `/opt/ssc-edge-proxy`: `git pull`, asegura `edge`,
`docker compose up -d`, `nginx -t`, `nginx -s reload`. Si la config es inválida
falla antes del reload y nginx sigue con la anterior.

## Puesta en marcha en el VPS (una vez)

1. Crear el repo en GitHub y registrar un runner self-hosted (`self-hosted`, `Linux`, `X64`) para este repo.
2. `docker network create edge`
3. Clonar en `/opt/ssc-edge-proxy`.
4. Si empresas aún corre con su nginx propio: desplegar empresas (su `app` se une a `edge`), luego `docker rm -f ssc-empresas-privadas-nginx-1` (hay unos segundos sin servicio) y `docker compose up -d` aquí.
5. Desplegar Cancún en `/opt/comercio-seguro-ssc-cancun` (primer `up -d --build` a mano).
6. Verificar: `curl -H 'Host: empresas.sscqroo.gob.mx' localhost/up` y lo mismo con `comercioseguro...` → 200.

## Probar en local (Mac)

```bash
docker network create edge

# 1. Cada app, con su .env.production local
cd ../ssc-empresas-privadas
docker compose --env-file .env.production -f docker-compose.production.yml up -d --build
cd ../comercio-seguro-ssc-cancun
docker compose --env-file .env.production -f docker-compose.production.yml up -d --build

# 2. El edge
cd ../ssc-edge-proxy
cp .env.example .env        # EDGE_PORT=8080 y rutas relativas a los repos
docker compose up -d
docker compose exec nginx nginx -t
```

Abrir `http://empresas.localhost:8080` y `http://comercioseguro.localhost:8080`
(los navegadores resuelven `*.localhost` a 127.0.0.1 sin tocar `/etc/hosts`).

Pruebas:

| Prueba | Esperado |
|---|---|
| `curl -i -H 'Host: otro.com' localhost:8080` | conexión cerrada (444) |
| `touch ../comercio-seguro-ssc-cancun/nginx/maintenance/on` | Cancún muestra mantenimiento con logos; empresas sigue normal. `rm` lo quita |
| `docker stop scu-cancun-web-1` | Cancún 502, empresas sigue arriba. Al levantarlo vuelve sin reiniciar nginx |
| Deploy simulado de una app | correr a mano los pasos de su `deploy.yml` |

## Troubleshooting

- `502 Bad Gateway`: la app no está en `edge` o no tiene el alias. `docker network inspect edge`.
- `nginx: [emerg] host not found in upstream`: alguien quitó el `set $upstream` y usó `proxy_pass http://alias` directo.
- Mantenimiento no aparece: revisar que la carpeta montada exista (`EMPRESAS_DIR` / `COMERCIOSEGURO_DIR`) y que el flag se llame `on`.
