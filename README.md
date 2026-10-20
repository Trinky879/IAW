# Práctica 1: Servidores, Proxies y Certificados - Nimbus Docs
**Alumno:** Adrián Fornell

## I. Diseño y Requisitos

### 1. Dominios ficticios elegidos
* **Marca Pública (`docs.afornell-cloud.es`):** Elegido para alojar la documentación técnica abierta a clientes sobre servicios cloud de la marca Afornell Cloud.
* **Marca Interna (`intranet.fornell-tech.internal`):** Elegido para la documentación privada de ingeniería de la marca Fornell Tech. Utiliza el TLD `.internal` para indicar que es de uso exclusivamente interno.
* **Puerto interno (`8016`) y fichero sensible (`credenciales-afornell.cfg`):** Nombres propios basados en las iniciales de Adrián Fornell (A=1, F=6) para cumplir con la regla de originalidad.

### 2. Elección y justificación del MPM de Apache
Se ha elegido **`mpm_event`**. Es un modelo híbrido (multiproceso y multihilo) asíncrono donde un hilo dedicado gestiona las conexiones en espera (*Keep-Alive*) sin bloquear hilos trabajadores. Resulta óptimo para un portal de documentación con alta concurrencia y menor consumo de RAM frente a `prefork` o `worker`.

---

## II. Fase 1 - Backend Apache Multi-Marca

### Comportamiento ante dominios desconocidos
Cuando Apache recibe una petición cuyo encabezado `Host` no coincide con ningún `ServerName` ni `ServerAlias`, selecciona por defecto el **primer Virtual Host cargado en orden alfabético**. Para controlarlo, se ha creado `00-afornell-catchall.conf` (`ServerName bloqueo.afornell.local`), que se carga primero gracias al prefijo `00-` y devuelve un `403 Forbidden` controlado (`Require all denied`).

---

## III. Fase 2 - Terminación TLS en Nginx

* **Protocolos y Cipher Suite:** Se restringe `ssl_protocols` a `TLSv1.2` y `TLSv1.3`. El cipher suite (`ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305`) garantiza *Forward Secrecy* (`ECDHE`), autenticación con nuestro certificado de curva elíptica (`ECDSA`) y cifrado autenticado (`GCM` y `ChaCha20`).
* **Código de redirección HTTP a HTTPS:** Se usa **`301 Moved Permanently`** para indicar a los navegadores que el cambio a HTTPS es definitivo y pueda almacenarse en caché.

### Salida real de `openssl s_client`
```text
subject=C = ES, ST = Andalucia, L = Jerez, O = NimbusDocs-AFornell, CN = docs.afornell-cloud.es
issuer=C = ES, ST = Andalucia, L = Jerez, O = NimbusDocs-AFornell, CN = docs.afornell-cloud.es
Server Temp Key: X25519, 253 bits
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
```

---

## IV. Fase 2 - Seguridad y Límite de Peticiones

### 1. Tercera cabecera de seguridad elegida
Además de `Strict-Transport-Security` y `X-Content-Type-Options`, se ha añadido **`X-Frame-Options: DENY`**, que impide incrustar el portal documental en etiquetas `<iframe>` externas y protege contra ataques de *Clickjacking*.

### 2. Cálculo y justificación de `rate` y `burst` (Rate Limiting)
* **`rate=3r/s`:** Permite el flujo sostenido máximo de un usuario legítimo leyendo documentación (3 peticiones/segundo).
* **`burst=6 nodelay`:** Permite absorber sin retrasos el pico puntual legítimo de hasta 6 peticiones/segundo al recargar varias pestañas, pero bloquea inmediatamente con código HTTP `429` los scripts de *scraping* (>20 peticiones/segundo).

### Demostración del bloqueo por ráfaga
```text
Peticion 1: HTTP 200
Peticion 2: HTTP 200
Peticion 3: HTTP 200
Peticion 4: HTTP 200
Peticion 5: HTTP 200
Peticion 6: HTTP 200
Peticion 7: HTTP 200
Peticion 8: HTTP 429
Peticion 9: HTTP 429
Peticion 10: HTTP 429
Peticion 11: HTTP 429
Peticion 12: HTTP 429
Peticion 13: HTTP 429
Peticion 14: HTTP 429
Peticion 15: HTTP 429
```

---

## V. Fase 3 - Diagnóstico Provocado

| Síntoma observado | Hipótesis inicial | Comando de diagnóstico usado | Causa real y solución |
| :--- | :--- | :--- | :--- |
| **Fallo 1 (Apache):** Al acceder a `intranet.fornell-tech.internal`, Apache devuelve `403 Forbidden` en vez de la web interna. | El sitio no está habilitado o fallan los permisos en `/var/www/fornell_internal`. | `docker exec afornell-apache apache2ctl -S` | **Causa:** En `20-afornell-interna.conf` se puso `<VirtualHost *:80>` en lugar de `<VirtualHost *:8016>`, por lo que caía en el catch-all. **Solución:** Cambiar a `*:8016` y recargar Apache. |
| **Fallo 2 (Nginx):** Ambas marcas devuelven siempre `403 Forbidden` por HTTPS aunque los contenedores están activos. | Fallo de red entre Nginx y Apache. | `docker exec afornell-apache tail /var/log/apache2/afornell_catchall_access.log` | **Causa:** Faltaba `proxy_set_header Host $host;` en Nginx, enviando `Host: backend_afornell` a Apache. **Solución:** Añadir `proxy_set_header Host $host;` y recargar Nginx. |
