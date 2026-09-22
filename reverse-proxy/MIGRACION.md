# Migración: sacar el nginx de adentro de `taller`

Una sola vez. `taller` ya tiene un certificado real sirviendo tráfico —
la idea de este orden es que la única ventana de downtime real sea los
segundos entre "apago el nginx viejo" y "prendo el nuevo", con todo lo
demás preparado de antes.

## 1. Preparación (sin downtime — se puede hacer con días de anticipación)

En `taller` (VPS), con los cambios de este PR ya traídos (`git pull`):

```bash
./deploy.sh
```

Esto recrea `frontend` con la red `edge` sumada (ver `docker-compose.yml`)
— **no toca** el `workshop-edge`/`workshop-certbot` viejos, que ya no están
en el archivo pero siguen corriendo como contenedores sueltos ("huérfanos"),
sirviendo tráfico real sin enterarse de nada.

Si usás observabilidad, lo mismo con Grafana:

```bash
docker compose -f docker-compose.observability.yml up -d
```

Confirmá que `frontend` y `grafana` están en la red `edge` además de
`workshop-net` (`docker inspect workshop-frontend --format '{{.NetworkSettings.Networks}}'`).

En el VPS, este proxy:

```bash
scp -r reverse-proxy user@IP:/opt/reverse-proxy
ssh user@IP
docker network inspect edge >/dev/null 2>&1 || docker network create edge
```

Copiar el certificado que ya existe (no se vuelve a pedir):

```bash
# Confirmar el nombre real del volumen viejo — depende del nombre de
# carpeta del repo de taller en el VPS (default de compose: <carpeta>_certbot-etc).
docker volume ls | grep certbot

# Con el nombre confirmado (ejemplo: workshop_certbot-etc):
docker volume create reverse-proxy_certbot-certs
docker run --rm \
  -v workshop_certbot-etc:/src:ro \
  -v reverse-proxy_certbot-certs:/dest \
  alpine cp -a /src/. /dest/
```

Validar que el `.conf` de taller matchea el certificado copiado, sin
publicar puertos todavía (80/443 los sigue teniendo el `edge` viejo):

```bash
cd /opt/reverse-proxy
docker run --rm \
  -v "$(pwd)/conf.d:/etc/nginx/conf.d:ro" \
  -v reverse-proxy_certbot-certs:/etc/letsencrypt:ro \
  nginx:1.27-alpine nginx -t
```

Si dice `syntax is ok` / `test is successful`, está todo listo para el
corte.

## 2. Corte (la única ventana real, segundos)

```bash
# En taller:
docker stop workshop-edge workshop-certbot
docker rm workshop-edge workshop-certbot

# En reverse-proxy:
cd /opt/reverse-proxy
docker compose up -d
```

Verificar enseguida:

```bash
curl -I https://admin.tallerlallave.com
curl -I https://grafana.tallerlallave.com
```

200 con certificado válido en los dos. Si algo falla, `docker compose logs
nginx` en `reverse-proxy/` — el certificado copiado y la config ya se
validaron en el paso 1, así que un fallo acá sería de red (`edge` mal
armada) o de un typo en el `.conf`, no del certificado en sí.

## 3. Limpieza (después de confirmar que todo anda, sin apuro)

```bash
# En taller — los volúmenes viejos del certbot ya no los usa nadie:
docker volume rm workshop_certbot-etc workshop_certbot-webroot

# Si taller tenía una línea de cron para renovar el certificado viejo,
# sacarla — reverse-proxy/README.md tiene la nueva (renueva los dos
# certificados, taller y Acompañada, con una sola línea).
crontab -e
```

## 4. Sumar Acompañada

Esta parte no es "migración" — es sumar un sitio nuevo, cero riesgo para lo
que ya funciona. Seguir "Sumar un dominio nuevo" en `reverse-proxy/README.md`
con `francia.tetraquarklover.space`/`api.francia.tetraquarklover.space`
(el `conf.d/francia.conf` de acá ya tiene el `:80` armado, solo falta pedir
el certificado y agregar los `:443`).
