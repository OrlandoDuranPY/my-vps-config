# Plan: VPS Debian 13 segura para PHP/Laravel, Node (Vue/Nuxt/React/Next) y Python

Este documento es solo la **planeación**. Cada fase se va a ejecutar y documentar por separado
(en el README) cuando lleguemos a ella.

## Decisiones tomadas

- **SO:** Debian 13 (trixie), VPS de ≤ 2 GB de RAM.
- **Runtime:** todo nativo con systemd. Nada de Docker.
- **Cloudflare:** DNS con proxy activo (nube naranja), SSL **Full (strict)** con certificado
  **Origin CA** y **Authenticated Origin Pulls** (mTLS). En el firewall, el 443 solo acepta IPs de Cloudflare.
- **Deploy:** GitHub Actions compila y sube el código por SSH (rsync) con un usuario por app, sin root.
- **Acceso de administración:** SSH solo con llave, usuario admin con sudo, sin login de root.

## Arquitectura objetivo

```
Visitante ──HTTPS──> Cloudflare (WAF, DDoS, caché, TLS público)
                         │  443, solo desde IPs de Cloudflare + certificado cliente (mTLS)
                         ▼
                   nginx (VPS) ── vhost por dominio/subdominio
                    ├── php-fpm: un pool por app Laravel (corre con el usuario Linux de esa app)
                    ├── Node (Nuxt/Next SSR): servicio systemd en 127.0.0.1:<puerto>
                    ├── Python (FastAPI/Django): gunicorn/uvicorn en 127.0.0.1:<puerto>
                    └── estáticos (Vue/React SPA): servidos directo por nginx
                         │
                   PostgreSQL / MariaDB  → solo 127.0.0.1, un usuario y una BD por app

Tú (admin) ──SSH:22, solo llave──> VPS     (fail2ban vigila los intentos)
GitHub Actions ──SSH:22, llave de deploy──> usuario de la app
```

Puertos expuestos a Internet: **22** (SSH) y **443** (solo Cloudflare). El 80 queda cerrado:
con Full (strict) Cloudflare siempre conecta al origen por 443, y la redirección HTTP→HTTPS la hace Cloudflare.

---

## Fase 0 — Preparación (antes de tocar el servidor)

**En tu computadora**
1. Crear una llave SSH ed25519 dedicada para la VPS (con passphrase).
2. Anotar la IP de la VPS y el acceso root/contraseña que dio el proveedor.

**En el proveedor de la VPS**
3. Snapshot inicial (punto de retorno si algo sale mal).
4. Si el proveedor tiene firewall de red, dejar solo 22 y 443 (defensa en profundidad).
5. Tener a mano la consola web/VNC del proveedor: es la salida de emergencia si te bloqueas por SSH.

**En Cloudflare**
6. DNS: registros `A`/`AAAA` del dominio y subdominios **con proxy (naranja)**. Ningún registro gris
   apuntando a la IP de la VPS (revelaría el origen). Para SSH usa la IP directa, no un subdominio.
7. SSL/TLS → modo **Full (strict)**.
8. SSL/TLS → Origin Server → crear certificado **Origin CA** para `dominio.com` y `*.dominio.com`
   (vale 15 años). Guardar el `.pem` y la `.key` de forma segura; se suben a la VPS en la Fase 3.
9. Activar **Authenticated Origin Pulls** (antes de que nginx lo exija, o los sitios darán 400).
10. Edge Certificates: Always Use HTTPS, TLS mínimo 1.2, TLS 1.3 activo.
    HSTS se activa al final, cuando todo funcione por HTTPS.
11. Seguridad: Managed Ruleset gratuito activo, Bot Fight Mode (ojo con APIs y webhooks),
    una regla de rate limiting para rutas de login, y una regla que bloquee ruido típico
    (`/.env`, `/.git`, `/wp-admin`, `/xmlrpc.php`).
12. Correo: no montar servidor de correo en la VPS. Usar un proveedor transaccional (Resend, Postmark, SES).

**Resultado:** llave SSH lista, snapshot hecho, Cloudflare configurado y certificado Origin CA guardado.

---

## Fase 1 — Acceso SSH seguro y usuario administrador

Objetivo: entrar a la VPS desde tu computadora con tu propia llave, con un usuario normal con sudo,
y cerrar el acceso de root y por contraseña.

1. Primer ingreso como root (con lo que dio el proveedor) y actualización completa del sistema.
2. Crear el usuario admin (ej. `orlando`), agregarlo a `sudo` y a un grupo `sshusers`,
   ponerle contraseña (solo se usa para sudo, nunca para SSH).
3. Copiar tu llave pública a `~/.ssh/authorized_keys` del admin.
4. **Probar el login con la llave en una terminal nueva, sin cerrar la sesión de root.**
5. Endurecer SSH con un archivo en `/etc/ssh/sshd_config.d/` (con prefijo `00-` para que gane sobre
   `50-cloud-init.conf`, que en algunos VPS activa contraseñas):
   - sin root, sin contraseñas, solo llave pública
   - solo usuarios del grupo `sshusers`
   - `MaxAuthTries 3`, `LoginGraceTime` corto, keepalive
   - sin X11 ni agent forwarding; túneles locales permitidos (para conectar DBeaver/TablePlus a las BD por SSH)
6. Validar la config (`sshd -t`), recargar el servicio y volver a probar desde otra terminal.
7. En tu computadora: entrada en `~/.ssh/config` (alias `vps`, usuario, IP, llave) para entrar con `ssh vps`.

Cambiar el puerto 22 es opcional: solo reduce ruido en los logs; la seguridad real la dan las llaves y fail2ban.

**Verificación:** `ssh vps` funciona; `ssh root@IP` y el login con contraseña son rechazados.

---

## Fase 2 — Sistema base

1. Paquetes base: `curl`, `git`, `unzip`, `rsync`, `htop`, `ca-certificates`, `gnupg`.
2. **Actualizaciones automáticas de seguridad** (`unattended-upgrades`) + `needrestart` para reiniciar
   servicios tras los parches. Decidir si se permite reinicio automático de madrugada.
3. **Swap de 2 GB** con `swappiness` bajo (imprescindible con 2 GB de RAM).
4. **sysctl** endurecido: anti-spoofing, sin ICMP redirects ni source routing, SYN cookies,
   `kptr_restrict`, `dmesg_restrict`.
5. Journal persistente con límite de tamaño (Debian 13 no trae rsyslog; los logs están en `journalctl`).
6. Zona horaria (recomendado: UTC en el servidor) y sincronización de hora activa.

**Verificación:** `swapon --show`, `systemctl status unattended-upgrades`, `sysctl` muestra los valores.

---

## Fase 3 — Firewall y protección contra fuerza bruta

1. **nftables** (nativo de Debian 13), política por defecto `drop`:
   - permitir conexiones establecidas, loopback, ICMPv6 y ping limitado
   - permitir 22/tcp
   - permitir 443/tcp **solo desde los rangos IP de Cloudflare**
   - todo lo demás bloqueado (incluye 80, 5432 y 3306)
2. Script + timer de systemd que descarga semanalmente los rangos de Cloudflare
   (`cloudflare.com/ips-v4` y `ips-v6`), valida que la lista sea sana antes de aplicarla
   (para no dejar el 443 cerrado), y regenera la regla del firewall y la config `real_ip` de nginx.
3. **fail2ban** con backend `systemd`, jail `sshd` (modo agresivo), bans incrementales
   y jail `recidive` para reincidentes. Sin jails de nginx: el tráfico web llega por Cloudflare
   y el filtrado web se hace allí.

**Verificación:** `nft list ruleset`, `fail2ban-client status sshd`, y desde fuera
`curl -k https://IP_DE_LA_VPS` no conecta (timeout).

---

## Fase 4 — Stack web

1. **nginx** (versión de Debian):
   - quitar el sitio por defecto; un `default_server` en 443 que rechaza el handshake si el dominio no está configurado
   - `server_tokens off`, TLS 1.2/1.3, timeouts cortos, límite de tamaño de body
   - IP real del visitante desde `CF-Connecting-IP` (solo confiando en IPs de Cloudflare)
   - rate limit por IP como segunda capa
   - certificado Origin CA en `/etc/ssl/cloudflare/` (permisos 600) y CA de Authenticated Origin Pulls
     con `ssl_verify_client on`: solo Cloudflare puede hablar con nginx
   - cabeceras de seguridad básicas (`nosniff`, `Referrer-Policy`, `X-Frame-Options`)
2. **PHP 8.4** (versión de Debian 13) con php-fpm y extensiones para Laravel
   (mbstring, xml, curl, zip, bcmath, intl, gd, pgsql, mysql, sqlite3, opcache).
   `php.ini` endurecido (`expose_php off`, sin `display_errors`, cookies seguras, opcache) + **Composer**.
3. **Node.js LTS** desde el repositorio de NodeSource (el de Debian es antiguo) + corepack para pnpm/yarn.
4. **Python 3.13** (Debian) con `venv`, `pip` y herramientas de compilación.

**Verificación:** `nginx -t`, `php -v`, `node -v`, `python3 -V`; `ss -tlnp` solo muestra 22 y 443 hacia afuera.

---

## Fase 5 — Bases de datos

1. **PostgreSQL 17** (Debian): escucha solo en localhost, autenticación `scram-sha-256`, tuning para 2 GB.
2. **MariaDB 11.8** como "MySQL": Debian 13 no incluye MySQL de Oracle. MariaDB es compatible y Laravel
   la soporta con el driver `mariadb`. Si algún proyecto necesita MySQL de Oracle específicamente, se evalúa
   el repositorio oficial de MySQL. Config: `bind-address 127.0.0.1`, `skip-name-resolve`,
   `local-infile=0`, tuning para 2 GB.
3. Regla: **un usuario y una base de datos por app**, con contraseña aleatoria y privilegios solo sobre su BD.
   Nunca usar `root`/`postgres` desde las apps.
4. Acceso desde tu computadora (DBeaver, TablePlus) por **túnel SSH**, nunca abriendo 5432/3306.

**Verificación:** `ss -tlnp` muestra 5432 y 3306 solo en `127.0.0.1`.

---

## Fase 6 — Estructura por aplicación (aislamiento)

Cada sitio o app tiene:
- **Su propio usuario Linux** (ej. `blog`, `api`). Si una app es comprometida, no puede leer el código
  ni el `.env` de las otras.
- Directorio `/var/www/<app>/` con `releases/` (versiones), `shared/` (`.env`, `storage/`, uploads)
  y `current` (symlink a la versión activa → deploys atómicos y rollback inmediato).
- `.env` con permisos `600`, propiedad del usuario de la app.
- Su vhost de nginx.
- Según el tipo:
  - **Laravel:** pool php-fpm propio (socket propio, `pm = ondemand` para ahorrar RAM) + servicios systemd
    de usuario para `queue:work` y `schedule:work`.
  - **Nuxt/Next (SSR):** servicio systemd de usuario escuchando en `127.0.0.1:<puerto>`, con límite de memoria.
  - **Python:** igual que Node, con gunicorn/uvicorn dentro de un `venv`.
  - **Vue/React SPA o sitios estáticos:** nginx sirve los archivos directamente.
- Una llave SSH de deploy restringida (`restrict`) solo para ese usuario.

La idea es tener un script que cree todo esto con un solo comando por app.

---

## Fase 7 — Deploy con GitHub Actions

1. El build se hace **en GitHub Actions, no en la VPS** (compilar Next/Nuxt con 2 GB suele quedarse sin memoria).
2. Flujo: build → `rsync` a `releases/<commit>` → enlazar `shared/` → migraciones (Laravel) →
   cambiar `current` de forma atómica → recargar php-fpm / reiniciar el servicio → conservar las últimas 5 versiones.
3. Secretos en GitHub: llave privada de deploy, host y la huella del servidor (`known_hosts`) para evitar MITM.
4. El usuario de deploy no tiene sudo, salvo una regla exacta para recargar php-fpm.

---

## Fase 8 — Backups

1. Dump diario de cada base de datos (`pg_dump` / `mariadb-dump`) con retención local de 7 días (timer de systemd).
2. Copia externa cifrada con **restic** hacia **Cloudflare R2** (u otro S3): dumps + `/var/www/*/shared` + `/etc`.
   Retención: 7 diarios, 4 semanales, 6 mensuales.
3. Snapshots periódicos del proveedor como capa extra.
4. **Probar una restauración** al terminar y luego cada trimestre. Un backup no probado no es backup.

---

## Fase 9 — Verificación final y mantenimiento

**Checklist de seguridad**
- [ ] Root y contraseñas bloqueados por SSH
- [ ] Solo 22 y 443 abiertos; 443 solo para Cloudflare
- [ ] La IP de la VPS no responde HTTPS directamente
- [ ] BD solo en localhost
- [ ] fail2ban activo y baneando
- [ ] Parches automáticos funcionando
- [ ] Auditoría con `lynis audit system` y revisión de sus sugerencias
- [ ] HSTS activado en Cloudflare

**Rutina**
- Semanal: `systemctl --failed`, `journalctl -p err -b`, `fail2ban-client status`.
- Mensual: `apt full-upgrade`, revisar espacio en disco y uso de RAM.
- Trimestral: probar restauración de backups, revisar usuarios y llaves SSH.
- Monitoreo de disponibilidad externo (Cloudflare Health Checks o UptimeRobot).

---

## Riesgos y límites (2 GB de RAM)

Estimación aproximada: sistema ~200 MB, PostgreSQL ~300 MB, MariaDB ~350 MB, nginx ~20 MB,
cada pool PHP hasta ~300 MB bajo carga, cada app Node SSR 100–250 MB.
→ Es realista tener **2–3 apps** a la vez. Si un solo motor de BD alcanza para tus proyectos,
quitar el otro libera unos 300 MB.

Otros riesgos:
- **Fuga de IP de origen:** si la IP estuvo expuesta antes (DNS histórico), el firewall igual bloquea,
  pero conviene tenerlo en cuenta.
- **Certificado global de Authenticated Origin Pulls:** es compartido por todos los clientes de Cloudflare;
  combinado con el filtro de IPs es suficiente para empezar. Más adelante se puede usar un certificado propio por zona.
- **Bloquearte por SSH:** por eso la Fase 1 exige probar el acceso en otra terminal antes de cerrar la sesión de root.

## Orden de ejecución

0 → 1 → 2 → 3 → 4 → 5 → 6 (primera app) → 7 → 8 → 9.
Cada fase se documentará en el README con los comandos reales usados y su verificación.
