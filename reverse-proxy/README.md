# reverse-proxy

nginx único del VPS: el único servicio con `80`/`443` publicados al host,
reparte tráfico por dominio a quien esté conectado a la red `edge`. Es un
proyecto aparte — ni `taller` ni Acompañada (ni ningún proyecto futuro) lo
poseen, y ninguno de ellos sabe que el otro existe. Este repo es el único
lugar donde vive el mapa completo dominio → contenedor.

Probado localmente antes de entregarlo: routing por SNI/Host a los dos
proyectos, certificado servido correctamente tras "migrarlo" (copiado, no
re-emitido), catch-all HTTP (444) y catch-all HTTPS (`ssl_reject_handshake`)
para cualquier dominio no reconocido, y que un backend real (puerto
incorrecto a propósito) da 502 en vez de tumbar todo el proxy.

## Instalar en el VPS (primera vez)

```bash
scp -r reverse-proxy user@IP:/opt/reverse-proxy
ssh user@IP "docker network create edge"
```

Cualquier proyecto que este proxy vaya a servir (`taller`, Acompañada) tiene
que sumarse a `edge` en su propio `docker-compose.yml`, con `container_name`
fijo — ver `MIGRACION.md` para el caso de `taller` (que hoy tiene su propio
nginx) y el README de Acompañada para el suyo.

## Sumar un dominio nuevo

1. DNS: registro A apuntando a la IP del VPS.
2. `conf.d/<proyecto>.conf`: copiar la estructura de `conf.d/francia.conf`
   (server block de `:80` con el challenge de certbot — **sin** `:443`
   todavía, agregarlo ahora con un certificado inexistente hace fallar
   `nginx -t` del archivo entero).
3. `docker compose exec nginx nginx -t && docker compose exec nginx nginx -s reload`.
4. Certificado:
   ```bash
   docker compose run --rm certbot certonly --webroot -w /var/www/certbot \
     -d tu-dominio.com --email vos@ejemplo.com --agree-tos --no-eff-email
   ```
5. Agregar el/los `server { listen 443 ssl; ... }` al mismo `.conf` (ver
   `conf.d/taller.conf` como ejemplo con dos dominios sobre el mismo
   certificado), reemplazar el `return 404;` del bloque `:80` por
   `return 301 https://$host$request_uri;`, `nginx -t && nginx -s reload`.

## Renovación

```bash
crontab -e
```

```
0 3,15 * * * cd /opt/reverse-proxy && docker compose run --rm certbot renew --quiet && docker compose exec nginx nginx -s reload
```

Renueva TODOS los certificados que administra este proxy (taller y
Acompañada por igual) — no hace falta una línea de cron por dominio.

## Sacar un proyecto

Borrar su `conf.d/<proyecto>.conf`, `nginx -t && nginx -s reload`. El
certificado, dejarlo vencer solo o `docker compose run --rm certbot delete
--cert-name tu-dominio.com`. El proyecto en sí no se entera — solo deja de
recibir tráfico por ese dominio.
