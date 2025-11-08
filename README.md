# Gestión automática de certificados SSL para atenapp.com.ar

Este entorno usa **Nginx + Certbot en contenedores Docker** para servir y renovar certificados SSL de Let's Encrypt.

---

## 🔧 Estructura de contenedores

| Contenedor | Rol | Puertos | Comentario |
|-------------|------|----------|-------------|
| `nginx-proxy` | Servidor web principal | 80 / 443 | Expone HTTPS al público y monta los certificados desde `/etc/letsencrypt` |
| `certbot` | Cliente Let's Encrypt | - | Gestiona la obtención y renovación de certificados |
| `backend-backend` | Servidor PHP (CodeIgniter 4) | 80 interno | Recibe las peticiones del proxy |
| `backend-db-1` | Base de datos MySQL | 3307 | Solo interno |

---

## 📁 Volúmenes relevantes

Asegurarse de que ambos contenedores compartan los mismos volúmenes:

```yaml
volumes:
  - /etc/letsencrypt:/etc/letsencrypt:rw
  - /var/www/certbot:/var/www/certbot:rw
```

---

## 🔒 Certificados activos

Listar todos los certificados:

```bash
docker exec certbot certbot certificates
```

Ejemplo de salida:
```
Certificate Name: atenapp.com.ar
Domains: atenapp.com.ar www.atenapp.com.ar
Expiry Date: 2026-01-01 16:35:16+00:00 (VALID: 53 days)
Certificate Path: /etc/letsencrypt/live/atenapp.com.ar/fullchain.pem
Private Key Path: /etc/letsencrypt/live/atenapp.com.ar/privkey.pem
```

---

## 🔁 Renovación automática

El cron configurado en el host renueva todos los días a las **03:00 AM**:

```cron
0 3 * * * docker exec certbot certbot renew --quiet --deploy-hook "nginx -s reload" -n && docker exec nginx-proxy nginx -s reload
```

### Ver cron activo
```bash
crontab -l
```

### Logs
- `/var/log/syslog` → ejecución del cron
- `/var/log/letsencrypt/letsencrypt.log` → registros de Certbot

---

## ⚙️ Renovación manual

Forzar una renovación de prueba (sin escribir cambios reales):

```bash
docker exec certbot certbot renew --dry-run
```

Renovación real y recarga inmediata de Nginx:

```bash
docker exec certbot certbot renew --quiet
docker exec nginx-proxy nginx -s reload
```

---

## 🔍 Verificación de certificados activos

Comprobar fechas del certificado actualmente servido:

```bash
openssl s_client -connect atenapp.com.ar:443 -servername atenapp.com.ar </dev/null 2>/dev/null | openssl x509 -noout -dates
```

Resultado esperado:
```
notBefore=Oct  3 16:35:17 2025 GMT
notAfter=Jan  1 16:35:16 2026 GMT
```

Verificar todos los certificados locales:

```bash
docker exec certbot openssl x509 -in /etc/letsencrypt/live/atenapp.com.ar/fullchain.pem -noout -dates
```

---

## 🌐 Configuración Nginx relevante

### Bloque HTTP (puerto 80)
```nginx
location ^~ /.well-known/acme-challenge/ {
    alias /var/www/certbot/.well-known/acme-challenge/;
    default_type "text/plain";
    try_files $uri =404;
}

location / {
    return 301 https://$host$request_uri;
}
```

### Bloque HTTPS (puerto 443)
```nginx
ssl_certificate     /etc/letsencrypt/live/atenapp.com.ar/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/atenapp.com.ar/privkey.pem;
```

---

## 🧪 Diagnóstico rápido

### 1️⃣ Ver certificado servido externamente
```bash
echo | openssl s_client -connect atenapp.com.ar:443 -servername atenapp.com.ar 2>/dev/null | openssl x509 -noout -serial -dates
```

### 2️⃣ Verificar IPv4/IPv6
```bash
curl -4I https://atenapp.com.ar
curl -6I https://atenapp.com.ar
```

---

## ✅ Buenas prácticas

- No forzar `--force-renewal` salvo que haya errores en los archivos.
- Mantener el cron activo y revisar los logs cada tanto.
- Verificar DNS `A` y `AAAA` (si hay IPv6) apuntando a la misma IP.
- Asegurarse de que `/etc/letsencrypt` tenga permisos correctos y esté montado en ambos contenedores.

---

**Última actualización:** Noviembre 2025  
**Autor:** Gonzalo Guerra / AtenApp Infrastructure
