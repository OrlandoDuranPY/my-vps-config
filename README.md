# VPS Debian 13 — Documentación

Configuración paso a paso de la VPS. El plan general está en [PLAN.md](PLAN.md).
Cada fase registra qué se hizo, los comandos usados y cómo se verificó.

| Fase | Estado |
|------|--------|
| 0. Preparación (llave SSH, proveedor, Cloudflare) | Validada |
| 1. Acceso SSH seguro | Validada |
| 2. Sistema base | Validada |
| 3. Firewall y fail2ban | Validada |
| 4. Stack web | Validada |
| 5. Bases de datos | Pendiente |
| 6. Estructura por app | Pendiente |
| 7. Deploy (GitHub Actions) | Pendiente |
| 8. Backups | Pendiente |
| 9. Verificación final | Pendiente |

---

## Fase 0 — Preparación

Todo se hace **fuera de la VPS**: en tu computadora, en el panel del proveedor y en Cloudflare.

### 0.1 Llave SSH dedicada (en tu computadora)

Se usa una llave exclusiva para la VPS, distinta de la de GitHub. Si una se filtra, la otra sigue segura.

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/vps_ed25519 -C "usuario@dispositivo"
```

- `-C` es solo un **comentario** que se guarda al final de la llave pública. No afecta la seguridad
  ni el login; sirve para reconocer cada llave en `~/.ssh/authorized_keys` del servidor.
- **Una llave por dispositivo**, cada una con su comentario (`usuario@laptop`, `usuario@pc-trabajo`...).
  Nunca copiar la llave privada entre equipos: si pierdes uno, borras solo su línea en `authorized_keys`
  y los demás siguen funcionando.

- Pon una **passphrase**: si roban el archivo, sin ella no sirve.
- `-a 100`: más rondas de derivación, lo que hace más lento un ataque de fuerza bruta a la passphrase.
- Resultado: `~/.ssh/vps_ed25519` (privada, nunca se comparte) y `~/.ssh/vps_ed25519.pub` (pública, va al servidor).

Para no escribir la passphrase en cada conexión, cárgala en el agente de la sesión:

```bash
ssh-add ~/.ssh/vps_ed25519
```

### 0.2 Datos de la VPS

Anota (en un gestor de contraseñas, no en este repo):
- IP pública IPv4 (y la IPv6, si tiene)
- Cómo entras hoy: root con contraseña o con una llave del proveedor
- Usuario admin que se va a crear en la Fase 1

### 0.3 Panel del proveedor (Vultr)

No se usan snapshots (tienen costo extra). Por eso la consola web es el único plan de rescate.

**Consola web (obligatorio probarla)**
1. `Products → Compute` → clic en el servidor.
2. En `Overview` están la IP, el usuario `root` y su contraseña (ícono del ojo para mostrarla).
3. Arriba a la derecha, ícono de monitor **View Console**. Se abre una terminal en el navegador.
4. Entrar como `root` con esa contraseña. La consola no permite pegar texto de forma fiable:
   si la contraseña es larga, cámbiala después de entrar con `passwd` por una que puedas escribir.
5. Comprobar la versión:
   ```bash
   cat /etc/debian_version
   ```
   Debe mostrar `13.x`.

**Firewall de Vultr (gratuito, filtra antes de llegar a la VPS)**
1. `Network → Firewall Groups → Add Firewall Group`, nombre: `web-ssh`.
2. Reglas **IPv4**:
   - `SSH` → puerto `22` → origen `Anywhere (0.0.0.0/0)`
   - `HTTPS` → puerto `443` → origen `Cloudflare`
3. Reglas **IPv6**: las mismas dos (en IPv6 también existe la opción `Cloudflare`).
4. Sección `Attached Instances` → `Attach to Instance` → elegir el servidor.

Todo lo que no esté en las reglas queda bloqueado. Si la opción `Cloudflare` no aparece, usar `Anywhere`
en el 443: el firewall del servidor (Fase 3) también filtra por IPs de Cloudflare.

### 0.4 Cloudflare

> Los nombres de los menús pueden variar un poco según la versión del panel.

**DNS** (`DNS → Records`)
- [ ] `A` para `@` → IP de la VPS, **Proxied (nube naranja)**.
- [ ] `AAAA` para `@` → IPv6 de la VPS, proxied (solo si la VPS tiene IPv6).
- [ ] `CNAME` para `www` → `@`, proxied.
- [ ] Ningún registro **gris (DNS only)** apuntando a la IP de la VPS: revelaría el servidor real.
      Para SSH se usa la IP directamente, no un subdominio.

**SSL/TLS**
- [ ] `SSL/TLS → Overview → Configure`: elegir **Full (Strict)** manualmente y **Save**.
      No usar "Automatic SSL/TLS": ajusta el modo solo y podría bajarlo a Full o Flexible si un día no ve el origen.
- [ ] `SSL/TLS → Origin Server → Create Certificate`:
  - Clave: RSA 2048 o ECC (cualquiera sirve)
  - Hostnames: `tudominio.com` y `*.tudominio.com`
  - Validez: 15 años
  - Copia el **Origin Certificate** y la **Private Key** a tu gestor de contraseñas.
    **La private key solo se muestra una vez.** Se subirán a la VPS en la Fase 4.
  - Nota: el comodín cubre un solo nivel (`api.tudominio.com` sí, `a.b.tudominio.com` no).
- [ ] `SSL/TLS → Origin Server → pestaña Authenticated Origin Pulls` → activar **Global** (certificado compartido de Cloudflare). Zone-level y Per-hostname se dejan vacíos.
      Cloudflare presentará un certificado de cliente al conectar con nginx, y nginx rechazará
      cualquier conexión sin él.
- [ ] `SSL/TLS → Edge Certificates`:
  - **Always Use HTTPS**: On
  - **Minimum TLS Version**: 1.2
  - **TLS 1.3**: On
  - **Automatic HTTPS Rewrites**: On
  - **Certificate Transparency Monitoring**: On (avisa por correo si alguien emite un certificado para tu dominio)
  - No tocar: Advanced Certificate Manager (de pago), Disable Universal SSL, Opportunistic Encryption (dejar como está)
  - **HSTS**: todavía no. Se activa en la Fase 9, cuando todo funcione por HTTPS
    (activarlo antes puede dejar el sitio inaccesible por meses en los navegadores).

**Seguridad**
- [ ] Managed rules: en el plan Free la sección aparece vacía con "Upgrade plan". El Free Managed Ruleset se aplica automáticamente y no se configura; no hay nada que hacer.
- [ ] `Security → Settings`: **Bot Fight Mode** On.
      Ojo: puede bloquear webhooks y clientes de API legítimos. Si alguno falla, revisar aquí primero.
- [ ] `Security → Security rules → Create rule → Custom rule` → regla **"Bloquear rutas sensibles"**, acción **Block**, expresión:
  ```
  (http.request.uri.path contains "/.env") or
  (http.request.uri.path contains "/.git") or
  (http.request.uri.path contains "/wp-admin") or
  (http.request.uri.path contains "/wp-login.php") or
  (http.request.uri.path contains "/xmlrpc.php") or
  (http.request.uri.path contains "/phpmyadmin")
  ```
- [ ] `Security → Security rules → Create rule → Rate limiting rule` → regla **"Login"**:
  - Expresión: `(http.request.uri.path contains "/login")`
  - Mismo IP, más de **10 peticiones en 10 segundos** → **Block** por 10 segundos
    (el plan gratuito permite 1 regla con esos períodos).

**Correo**
- [ ] No se instalará servidor de correo en la VPS. Para enviar correos desde las apps usar un proveedor
      transaccional (Resend, Postmark, Amazon SES). Los registros `MX`/`SPF`/`DKIM` apuntan a ese proveedor,
      **nunca a la IP de la VPS**.

### Verificación de la Fase 0

- [ ] `ls ~/.ssh/vps_ed25519*` muestra las dos llaves.
- [ ] Consola web/VNC probada con éxito.
- [ ] `dig +short tudominio.com` devuelve IPs de **Cloudflare**, no la de la VPS.
- [ ] Certificado Origin CA y su private key guardados en el gestor de contraseñas.
- [ ] Authenticated Origin Pulls en On y SSL en Full (strict).

> Mientras nginx no esté configurado (Fase 4), el dominio mostrará un error 52x de Cloudflare. Es normal.

---

## Fase 1 — Acceso SSH seguro

Objetivo: entrar con un usuario propio y llave SSH, y bloquear root y contraseñas.

Valores usados en esta sección (reemplazar por los reales):
- `USUARIO_ADMIN`: usuario administrador (evitar nombres obvios como `admin` o `ubuntu`)
- `IP_DE_LA_VPS`: IP pública IPv4

> **Regla de oro:** mantener siempre una sesión de root abierta hasta comprobar que el nuevo acceso funciona
> en otra terminal. Si algo falla, la consola web del proveedor es el plan de rescate.

### 1.1 Actualizar el sistema (como root)

```bash
apt update && apt full-upgrade -y
```

Si se actualizó el kernel, reiniciar y volver a entrar:

```bash
reboot
```

### 1.2 Crear el usuario administrador (como root)

```bash
apt install -y sudo
adduser USUARIO_ADMIN
```

`adduser` pide una contraseña: será la de `sudo`, **no** se usará para SSH. Guardarla en el gestor de contraseñas.
Los demás campos (nombre, teléfono...) se pueden dejar vacíos con Enter.

Darle permisos de administrador y crear el grupo que podrá entrar por SSH:

```bash
usermod -aG sudo USUARIO_ADMIN
groupadd sshusers
usermod -aG sshusers USUARIO_ADMIN
```

Verificar:

```bash
id USUARIO_ADMIN
```

Debe incluir los grupos `sudo` y `sshusers`.

### 1.3 Copiar la llave pública (en tu computadora)

```bash
ssh-copy-id -i ~/.ssh/vps_ed25519.pub USUARIO_ADMIN@IP_DE_LA_VPS
```

Pide la contraseña de `USUARIO_ADMIN` una única vez (todavía está permitido entrar con contraseña).
Con varios dispositivos, repetir este paso desde cada uno con su propia llave.

### 1.4 Probar el acceso con llave (en una terminal nueva)

```bash
ssh -i ~/.ssh/vps_ed25519 USUARIO_ADMIN@IP_DE_LA_VPS
```

Debe entrar **sin pedir la contraseña del usuario** (solo la passphrase de la llave, si no está en el agente).
Ya dentro, comprobar sudo:

```bash
sudo whoami
```

Debe responder `root`. **No seguir si alguno de los dos falla.**

### 1.5 Endurecer SSH (como root o con sudo)

Revisar si el proveedor dejó configuraciones propias:

```bash
ls /etc/ssh/sshd_config.d/
```

Crear `/etc/ssh/sshd_config.d/00-hardening.conf`. El prefijo `00-` es importante: OpenSSH usa el **primer**
valor que encuentra, así que este archivo gana sobre otros como `50-cloud-init.conf`.

```bash
sudo nano /etc/ssh/sshd_config.d/00-hardening.conf
```

Contenido:

```
# Solo llave pública, sin root, solo el grupo sshusers
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no
AuthenticationMethods publickey
AllowGroups sshusers

# Limitar intentos y sesiones colgadas
MaxAuthTries 3
MaxSessions 5
LoginGraceTime 20
ClientAliveInterval 300
ClientAliveCountMax 2

# Sin reenvíos innecesarios. "local" permite túneles -L para conectar
# clientes de base de datos (DBeaver, TablePlus) por SSH
X11Forwarding no
AllowAgentForwarding no
PermitTunnel no
AllowTcpForwarding local
```

Validar la sintaxis (no debe imprimir nada) y aplicar:

```bash
sudo sshd -t
sudo systemctl reload ssh
```

### 1.6 Comprobar (desde tu computadora, sin cerrar la sesión anterior)

1. El acceso con llave sigue funcionando:
   ```bash
   ssh -i ~/.ssh/vps_ed25519 USUARIO_ADMIN@IP_DE_LA_VPS
   ```
2. Root está bloqueado (debe responder `Permission denied (publickey)`):
   ```bash
   ssh root@IP_DE_LA_VPS
   ```
3. Las contraseñas están bloqueadas (debe responder `Permission denied (publickey)`):
   ```bash
   ssh -o PubkeyAuthentication=no USUARIO_ADMIN@IP_DE_LA_VPS
   ```
4. La configuración activa es la esperada (en la VPS):
   ```bash
   sudo sshd -T | grep -E '^(permitrootlogin|passwordauthentication|allowgroups|maxauthtries)'
   ```
   Resultado esperado: `permitrootlogin no`, `passwordauthentication no`, `allowgroups sshusers`, `maxauthtries 3`.

Si todo coincide, ya se pueden cerrar las sesiones de root.

### 1.7 Alias de conexión (en tu computadora)

Agregar al final de `~/.ssh/config`:

```
Host vps
    HostName IP_DE_LA_VPS
    User USUARIO_ADMIN
    IdentityFile ~/.ssh/vps_ed25519
    IdentitiesOnly yes
    AddKeysToAgent yes
```

Si ya existe una entrada para esa IP (de un servidor anterior), reemplazarla: un usuario o llave
viejos darán `Permission denied`.

Desde ahora basta con:

```bash
ssh vps
```

`AddKeysToAgent yes` guarda la llave en el agente SSH al usarla: la passphrase se pide **una vez por sesión**
del sistema (hasta cerrar sesión o reiniciar), no en cada conexión. En Fedora/GNOME el agente ya viene activo.
No quitar la passphrase de la llave para evitar escribirla: si roban el archivo, sería usable directamente.

`IdentitiesOnly yes` evita que el cliente pruebe todas tus llaves (con varias llaves cargadas, el servidor
cortaría por `MaxAuthTries 3` antes de llegar a la correcta).

### 1.8 Eliminar usuarios por defecto del proveedor

Algunas imágenes (por ejemplo Vultr) crean un usuario `linuxuser` (uid 1000), a veces con sudo.
Con `AllowGroups sshusers` ya no puede entrar por SSH, pero es una cuenta con privilegios que no se usa.

Revisar qué es y si tiene sudo:

```bash
getent passwd 1000
id linuxuser
```

Antes de borrarlo, confirmar que no tiene llaves ni procesos ni archivos propios:

```bash
sudo ls -la /home/linuxuser
sudo cat /home/linuxuser/.ssh/authorized_keys
ps -u linuxuser
```

Una vez comprobado que `USUARIO_ADMIN` funciona (paso 1.6), eliminarlo junto con su home (irreversible):

```bash
sudo deluser --remove-home linuxuser
```

Comprobar que no quedaron permisos de sudo para él (solo debe aparecer `README`, y el `grep` no debe imprimir nada):

```bash
sudo ls /etc/sudoers.d/
sudo grep -r linuxuser /etc/sudoers.d/
```

### 1.9 Agregar más dispositivos

Con las contraseñas bloqueadas, `ssh-copy-id` ya no sirve desde un equipo nuevo. Cada dispositivo genera
**su propia llave**, y su llave **pública** se agrega desde una sesión que ya funciona.
La llave privada nunca sale del dispositivo donde se creó.

**1. Generar la llave en el dispositivo nuevo**

Linux / macOS:

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/vps_ed25519 -C "usuario@dispositivo"
```

Android con Termux (instalar OpenSSH primero):

```bash
pkg update && pkg install openssh
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/vps_ed25519 -C "usuario@tablet"
```

**2. Copiar la llave pública** (una sola línea que empieza con `ssh-ed25519`):

```bash
cat ~/.ssh/vps_ed25519.pub
```

Pasarla al equipo que ya tiene acceso por cualquier medio (nota en el gestor de contraseñas, mensaje a uno mismo).
Es pública: no pasa nada si se ve. **Nunca** enviar el archivo sin `.pub`.

**3. Autorizarla en el servidor** (desde una sesión que ya funciona, como `USUARIO_ADMIN`):

```bash
nano ~/.ssh/authorized_keys
```

Pegar la llave en una línea nueva, al final. Una llave por línea. Verificar (muestra tipo y comentario de cada llave):

```bash
awk '{print $1, $NF}' ~/.ssh/authorized_keys
```

Cada línea debe terminar con un comentario distinto, que identifica a cada dispositivo.

**4. Configurar el alias y probar** (en el dispositivo nuevo): agregar a `~/.ssh/config` la misma entrada del
paso 1.7 y conectar:

```bash
ssh vps
```

**Quitar un dispositivo** (perdido, robado o reemplazado): borrar su línea de `~/.ssh/authorized_keys`.
Los demás siguen funcionando.

### Verificación de la Fase 1

- [ ] `ssh vps` entra sin contraseña de usuario.
- [ ] `sudo whoami` → `root`.
- [ ] `ssh root@IP_DE_LA_VPS` → `Permission denied (publickey)`.
- [ ] Login con contraseña → `Permission denied (publickey)`.
- [ ] `sshd -T` muestra los valores esperados.
- [ ] Sin usuarios por defecto del proveedor (`getent passwd 1000` no devuelve nada).

---

## Fase 2 — Sistema base

Objetivo: dejar el sistema con herramientas básicas, hora correcta, swap, kernel endurecido,
logs con límite y parches de seguridad automáticos.

Todos los comandos se ejecutan en la VPS como `USUARIO_ADMIN` (con `sudo`).

> En Debian, herramientas de administración como `sysctl`, `swapon` o `sshd` están en `/usr/sbin`, que no
> está en el `PATH` de un usuario normal. Si sale `command not found`, anteponer `sudo`.

### 2.1 Paquetes base

```bash
sudo apt update
sudo apt install -y curl git unzip rsync htop ca-certificates gnupg
```

### 2.2 Hora del sistema

El servidor trabaja en **UTC** (logs y tareas programadas sin cambios de horario). Revisar:

```bash
timedatectl
```

Debe mostrar `Time zone: Etc/UTC` (o `UTC`), `System clock synchronized: yes` y `NTP service: active`.
Si la zona es otra:

```bash
sudo timedatectl set-timezone UTC
```

### 2.3 Swap de 2 GB

Con 2 GB de RAM, la swap evita que el sistema mate procesos (PHP, Node, bases de datos) en picos de memoria.
Revisar si ya existe:

```bash
sudo swapon --show
free -h
```

`swapon` está en `/usr/sbin`, fuera del `PATH` de un usuario normal en Debian: siempre con `sudo`.

Si ya existe swap (algunos proveedores, como Vultr, la crean al instalar) **no crear otra**: basta con
que haya ~2 GB y comprobar que arranca sola y solo root la lee:

```bash
grep swap /etc/fstab
ls -l /swapfile
```

Esperado: una línea con `/swapfile` en `fstab` y permisos `-rw------- root root`.

Si `sudo swapon --show` no imprime nada, crearla:

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

- `chmod 600`: la swap puede contener datos de memoria (contraseñas, sesiones); solo root la lee.
- La línea en `/etc/fstab` la activa en cada arranque.

### 2.4 Endurecer el kernel (sysctl)

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

Contenido:

```
# Red: anti-spoofing, sin redirects ni source routing, SYN cookies
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv4.conf.all.log_martians = 1
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.tcp_syncookies = 1

# Kernel: ocultar direcciones de memoria y dmesg a usuarios sin privilegios, limitar ptrace
kernel.kptr_restrict = 2
kernel.dmesg_restrict = 1
kernel.yama.ptrace_scope = 1

# Memoria: usar la swap solo como respaldo
vm.swappiness = 10
```

Aplicar (sin reiniciar):

```bash
sudo sysctl --system
```

Revisar que no haya errores en la salida y comprobar un par de valores:

```bash
sudo sysctl vm.swappiness net.ipv4.tcp_syncookies kernel.kptr_restrict
```

Esperado: `vm.swappiness = 10`, `net.ipv4.tcp_syncookies = 1`, `kernel.kptr_restrict = 2`.

### 2.5 Límite de tamaño de los logs

Debian 13 no instala rsyslog: los logs viven en el journal (`journalctl`). Limitarlos para no llenar el disco:

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo nano /etc/systemd/journald.conf.d/size.conf
```

Contenido:

```
[Journal]
SystemMaxUse=200M
```

Aplicar y comprobar:

```bash
sudo systemctl restart systemd-journald
sudo journalctl --disk-usage
```

Un usuario normal solo ve sus propios logs: para revisar los del sistema usar siempre `sudo journalctl ...`.

### 2.6 Parches de seguridad automáticos

```bash
sudo apt install -y unattended-upgrades apt-listchanges needrestart
```

- `unattended-upgrades`: instala solo las actualizaciones de **seguridad** de Debian, todos los días.
- `needrestart`: tras actualizar, reinicia los servicios que usaban librerías viejas.

Crear la configuración local:

```bash
sudo nano /etc/apt/apt.conf.d/52local-unattended
```

Contenido:

```
// Buscar e instalar parches de seguridad cada día
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::AutocleanInterval "7";

Unattended-Upgrade::Remove-Unused-Dependencies "true";

// Reinicio cuando un parche lo requiera (kernel, librerías base).
// "true" = automático a la hora indicada (UTC). "false" = manual.
Unattended-Upgrade::Automatic-Reboot "REINICIO_AUTOMATICO";
Unattended-Upgrade::Automatic-Reboot-Time "10:00";
```

Reemplazar `REINICIO_AUTOMATICO` por `true` o `false`.

La hora está en **UTC**: convertir desde la hora local de menor tráfico (ejemplo: 04:00 en UTC−6 = `10:00`).

- **true**: el servidor se reinicia solo a esa hora cuando hace falta (~1 minuto sin servicio).
- **false**: reiniciar manualmente. Para saber si hay un reinicio pendiente:
  ```bash
  ls /var/run/reboot-required
  ```
  Si el archivo existe, reiniciar con `sudo reboot` en un horario conveniente.

Que `needrestart` reinicie servicios sin preguntar (si no, se queda esperando una respuesta que nadie da):

```bash
echo "\$nrconf{restart} = 'a';" | sudo tee /etc/needrestart/conf.d/auto.conf
```

Probar en modo simulación (no instala nada; revisar que no haya errores):

```bash
sudo unattended-upgrade --dry-run --debug 2>&1 | tail -20
```

Comprobar que los temporizadores están activos:

```bash
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer
```

Salir con `q` si la lista queda en pantalla. El reinicio automático solo ocurre si existe `/var/run/reboot-required`
(lo crea un parche que lo necesita), no a diario.

### Verificación de la Fase 2

- [ ] `timedatectl` → UTC y reloj sincronizado.
- [ ] `sudo swapon --show` → una swap de ~2 GB (la del proveedor o `/swapfile`).
- [ ] `sudo sysctl vm.swappiness` → `10`.
- [ ] `sudo journalctl --disk-usage` → por debajo de 200M.
- [ ] `unattended-upgrade --dry-run` sin errores y temporizadores `apt-daily*` activos.

---

## Fase 3 — Firewall y protección contra fuerza bruta

Objetivo: dejar abiertos solo el **22** (SSH) y el **443** (únicamente desde Cloudflare), y bloquear
automáticamente a quien insista en adivinar el acceso SSH.

Este firewall es la **segunda capa**: el firewall de Vultr (Fase 0) ya filtra antes de llegar a la VPS.

> **Riesgo:** una regla mal escrita puede dejarte sin SSH. Por eso las reglas se prueban primero con un
> **temporizador de reversión** que las borra solo a los 5 minutos, y solo se hacen permanentes cuando
> se comprueba el acceso desde otra terminal. Plan de rescate: consola web del proveedor.

Todos los comandos son en la VPS como `USUARIO_ADMIN`.

### 3.1 Instalar nftables y quitar UFW

nftables es el firewall nativo de Debian 13.

```bash
sudo apt install -y nftables
```

Todavía no activar el servicio.

Algunas imágenes (Vultr, entre otras) traen **UFW activo**, solo con el 22 abierto. Dos firewalls a la vez
se estorban: cada paquete debe pasar por ambos y UFW descartaría el 443 de Cloudflare. Revisar:

```bash
sudo ufw status verbose
```

Si dice `Status: active`, desactivarlo, comprobar que `ssh vps` sigue entrando desde otra terminal y
desinstalarlo para que nftables sea el único firewall del servidor:

```bash
sudo ufw disable
```

```bash
sudo apt purge -y ufw
```

### 3.2 Script que descarga los rangos IP de Cloudflare

Las IPs de Cloudflare cambian de vez en cuando, así que se descargan con un script en lugar de copiarlas a mano.
El script valida las listas antes de usarlas, para no dejar el 443 cerrado por una descarga rota.

```bash
sudo nano /usr/local/sbin/cloudflare-ips
```

Contenido:

```bash
#!/bin/bash
# Descarga los rangos IP de Cloudflare y genera /etc/nftables.d/cloudflare.nft
set -euo pipefail

OUT=/etc/nftables.d/cloudflare.nft
TMP=$(mktemp)
trap 'rm -f "$TMP"' EXIT

v4=$(curl -fsS --max-time 20 https://www.cloudflare.com/ips-v4)
v6=$(curl -fsS --max-time 20 https://www.cloudflare.com/ips-v6)

# Validar antes de tocar nada: solo CIDRs y una cantidad mínima de rangos
check() { # $1=lista $2=regex $3=mínimo de líneas
  [ "$(printf '%s\n' "$1" | grep -cvE "$2")" -eq 0 ] && [ "$(printf '%s\n' "$1" | wc -l)" -ge "$3" ]
}
check "$v4" '^([0-9]{1,3}\.){3}[0-9]{1,3}/[0-9]{1,2}$' 5 || { echo "Lista IPv4 inválida" >&2; exit 1; }
check "$v6" '^[0-9a-fA-F:]+/[0-9]{1,3}$' 3 || { echo "Lista IPv6 inválida" >&2; exit 1; }

{
  echo "define CF_V4 = { $(printf '%s\n' "$v4" | paste -sd, - | sed 's/,/, /g') }"
  echo "define CF_V6 = { $(printf '%s\n' "$v6" | paste -sd, - | sed 's/,/, /g') }"
} > "$TMP"

mkdir -p /etc/nftables.d
[ -f "$OUT" ] && cp "$OUT" "$OUT.bak"
install -m 644 "$TMP" "$OUT"

# Si el firewall ya está activo: validar y recargar; si falla, restaurar la lista anterior
if systemctl is-active --quiet nftables; then
  if nft -c -f /etc/nftables.conf; then
    systemctl reload nftables
  else
    [ -f "$OUT.bak" ] && mv "$OUT.bak" "$OUT"
    echo "Configuración inválida, se restauró la lista anterior" >&2
    exit 1
  fi
fi
```

Darle permisos, ejecutarlo y revisar el resultado:

```bash
sudo chmod 750 /usr/local/sbin/cloudflare-ips
sudo /usr/local/sbin/cloudflare-ips
cat /etc/nftables.d/cloudflare.nft
```

Deben aparecer dos líneas: `define CF_V4 = { ... }` y `define CF_V6 = { ... }` con los rangos.

### 3.3 Reglas del firewall

Guardar el archivo original por si hace falta volver atrás:

```bash
sudo cp /etc/nftables.conf /etc/nftables.conf.orig
sudo nano /etc/nftables.conf
```

Reemplazar **todo** el contenido por:

```
#!/usr/sbin/nft -f
# Política por defecto: descartar todo lo entrante.
# SSH abierto (protegido por llave + fail2ban); HTTPS solo desde Cloudflare.
# El puerto 80 no se abre: con SSL Full (strict) Cloudflare siempre conecta por 443.

include "/etc/nftables.d/cloudflare.nft"

# Declarar + borrar + recrear la tabla es atómico y no toca las tablas de fail2ban
# (por eso no se usa "flush ruleset").
table inet filter
delete table inet filter

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        ct state established,related accept
        ct state invalid drop
        iif lo accept

        # IPv6 necesita ICMPv6 (descubrimiento de vecinos); ping IPv4 limitado
        meta l4proto ipv6-icmp accept
        ip protocol icmp limit rate 10/second accept

        # DHCP: sin esto, la VPS pierde la red cuando vence su dirección
        udp sport 67 udp dport 68 accept
        udp sport 547 udp dport 546 accept

        tcp dport 22 accept

        ip  saddr $CF_V4 tcp dport 443 accept
        ip6 saddr $CF_V6 tcp dport 443 accept
    }

    chain forward {
        type filter hook forward priority filter; policy drop;
    }
}
```

Comprobar la sintaxis (no debe imprimir nada):

```bash
sudo nft -c -f /etc/nftables.conf
```

### 3.4 Probar con reversión automática

1. Programar que, en 5 minutos, se borren todas las reglas (red de seguridad):
   ```bash
   sudo systemd-run --unit=fw-rollback --on-active=5min /usr/sbin/nft flush ruleset
   ```
2. Aplicar las reglas (la sesión actual no se corta: las conexiones ya establecidas siguen permitidas):
   ```bash
   sudo nft -f /etc/nftables.conf
   ```
3. **Sin cerrar la sesión actual**, abrir una terminal nueva en tu computadora y entrar:
   ```bash
   ssh vps
   ```
4. Si entra, cancelar la reversión:
   ```bash
   sudo systemctl stop fw-rollback.timer
   ```
   Si no entra, no hacer nada: a los 5 minutos las reglas se borran solas y el acceso vuelve.

### 3.5 Hacer el firewall permanente

```bash
sudo systemctl enable --now nftables
sudo nft list ruleset | head -40
```

Debe verse la tabla `inet filter` con `policy drop` y las reglas de los puertos 22 y 443.

### 3.6 Actualizar las IPs de Cloudflare cada semana

```bash
sudo nano /etc/systemd/system/cloudflare-ips.service
```

```
[Unit]
Description=Actualiza rangos IP de Cloudflare para el firewall
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/cloudflare-ips
```

```bash
sudo nano /etc/systemd/system/cloudflare-ips.timer
```

```
[Unit]
Description=Actualización semanal de IPs de Cloudflare

[Timer]
OnCalendar=weekly
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

Activar, ejecutar una vez a mano y revisar:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cloudflare-ips.timer
sudo systemctl start cloudflare-ips.service
systemctl status cloudflare-ips.service --no-pager
systemctl list-timers cloudflare-ips.timer
```

El servicio debe terminar con `status=0/SUCCESS` y el temporizador mostrar una fecha en `NEXT`.

### 3.7 fail2ban

Bloquea temporalmente las IPs que fallan repetidamente al entrar por SSH.

```bash
sudo apt install -y fail2ban python3-systemd
sudo nano /etc/fail2ban/jail.local
```

Contenido:

```
[DEFAULT]
# Debian 13 no trae rsyslog (no existe /var/log/auth.log): leer del journal
backend = systemd
banaction = nftables-multiport
banaction_allports = nftables-allports

# 1 hora de bloqueo; cada reincidencia lo duplica, hasta 1 semana
bantime = 1h
bantime.increment = true
bantime.maxtime = 1w
findtime = 10m
maxretry = 5

[sshd]
enabled = true
mode = aggressive
```

No hay jails de nginx a propósito: todo el tráfico web llega a través de Cloudflare, así que banear la IP
en el origen no corta nada. El filtrado web se hace en el WAF de Cloudflare.

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

Debe mostrar `Journal matches: _SYSTEMD_UNIT=ssh.service ...`. Es normal que ya aparezca alguna IP en
`Banned IP list`: fail2ban lee el journal anterior y bloquea a los bots que llevaban días probando SSH.
Si el bloqueo es de un bot real, ya queda comprobado que el jail funciona.

Probar el bloqueo sin arriesgarse: banear una IP de documentación (no existe en Internet) y quitarla:

```bash
sudo fail2ban-client set sshd banip 203.0.113.5
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip 203.0.113.5
```

**No probar fallando contraseñas desde tu propia IP**: te bloquearías a ti mismo.

### 3.8 Verificación desde fuera (en tu computadora)

**Ejecutar en tu computadora, no en la VPS.** El 443 no debe responder a quien no sea Cloudflare (debe
terminar en timeout):

```bash
curl -k -m 10 https://IP_DE_LA_VPS
```

Resultado esperado: `Connection timed out after 10002 milliseconds`. Si el mismo comando se ejecuta **dentro de la
VPS**, falla al instante con `Could not connect to server`: la conexión sale por la interfaz local, el firewall
la acepta (`iif lo accept`) y no hay nada escuchando en el 443. No es una fuga, solo es la prueba hecha desde el
sitio equivocado. Como contraste, un puerto cualquiera (`curl -m 10 http://IP_DE_LA_VPS:8080`) también debe dar timeout.

Y SSH debe seguir funcionando:

```bash
ssh vps
```

### 3.9 Prueba de reinicio

Comprobar que el firewall y SSH vuelven solos tras reiniciar (no dejar esta prueba para cuando haya sitios):

```bash
sudo reboot
```

Esperar ~1 minuto y entrar con `ssh vps`. Después:

```bash
sudo nft list ruleset | head -20
sudo fail2ban-client status sshd
```

Si no entra tras 3-4 minutos, usar la consola web del proveedor y revisar `sudo systemctl status nftables`.

### Verificación de la Fase 3

- [ ] `sudo nft list ruleset` → tabla `inet filter` con `policy drop` (y sin tablas de UFW).
- [ ] `cat /etc/nftables.d/cloudflare.nft` → rangos IPv4 e IPv6.
- [ ] `systemctl list-timers cloudflare-ips.timer` → próxima ejecución programada.
- [ ] `sudo fail2ban-client status sshd` → jail activo.
- [ ] `curl -k -m 10 https://IP_DE_LA_VPS` desde fuera → timeout.
- [ ] `ssh vps` funciona, también después de reiniciar.

---

## Fase 4 — Stack web

Objetivo: nginx recibiendo solo tráfico de Cloudflare (con el certificado Origin CA y autenticación mutua TLS),
y los lenguajes instalados: PHP 8.4 con Composer, Node.js LTS y Python 3.13.

Versiones (las de Debian 13 salvo Node): nginx 1.26, PHP 8.4, Composer 2.8, Python 3.13, Node.js 24 LTS (NodeSource).

Valores genéricos de esta sección: `tudominio.com` es tu dominio.

### 4.1 Instalar nginx y quitar el sitio por defecto

```bash
sudo apt install -y nginx
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Sin sitios configurados, nginx no debe escuchar en ningún puerto todavía:

```bash
sudo ss -tlnp | grep nginx
```

No debe imprimir nada.

### 4.2 Certificados de Cloudflare en el servidor

Tres archivos en `/etc/ssl/cloudflare/`:

| Archivo | Qué es | Permisos |
|---|---|---|
| `origin.pem` | Certificado Origin CA (el de 15 años creado en la Fase 0) | 644 |
| `origin.key` | Su private key | **600**, solo root |
| `origin-pull-ca.pem` | CA de Cloudflare para validar que quien conecta es Cloudflare (Authenticated Origin Pulls) | 644 |

```bash
sudo install -d -m 755 /etc/ssl/cloudflare
sudo nano /etc/ssl/cloudflare/origin.pem
sudo nano /etc/ssl/cloudflare/origin.key
```

Pegar en cada archivo el contenido guardado en el gestor de contraseñas (certificado completo con sus líneas
`BEGIN/END CERTIFICATE`; la key con `BEGIN/END PRIVATE KEY`). **Nunca pegar la private key en chats ni repositorios.**

```bash
sudo chown root:root /etc/ssl/cloudflare/origin.pem /etc/ssl/cloudflare/origin.key
sudo chmod 644 /etc/ssl/cloudflare/origin.pem
sudo chmod 600 /etc/ssl/cloudflare/origin.key
sudo curl -fsS -o /etc/ssl/cloudflare/origin-pull-ca.pem \
  https://developers.cloudflare.com/ssl/static/authenticated_origin_pull_ca.pem
```

**Problema frecuente:** algunos gestores de contraseñas guardan el certificado y la key **en una sola línea**,
sin saltos de línea, y OpenSSL no los lee (`Could not find certificate` / `No supported data to decode`).
Un PEM válido tiene `BEGIN` en su línea, el cuerpo en líneas de 64 caracteres y `END` en su línea.
Revisar el número de líneas (un certificado ronda las 25; una key, unas 28; **1 línea es el síntoma**):

```bash
sudo wc -l /etc/ssl/cloudflare/origin.pem /etc/ssl/cloudflare/origin.key
```

Si están en una línea, reconstruir el formato sin volver a pegar. Certificado:

```bash
sudo sh -c '{ echo "-----BEGIN CERTIFICATE-----"; tr -d "\n " < /etc/ssl/cloudflare/origin.pem | sed "s/-----BEGINCERTIFICATE-----//;s/-----ENDCERTIFICATE-----//" | fold -w 64; echo; echo "-----END CERTIFICATE-----"; } > /etc/ssl/cloudflare/origin.pem.new && mv /etc/ssl/cloudflare/origin.pem.new /etc/ssl/cloudflare/origin.pem'
```

Private key. Se crea con `umask 077` (nadie más puede leerla) y solo reemplaza al original si OpenSSL
la acepta; si falla, el original queda intacto y hay que borrar `origin.key.new`:

```bash
sudo sh -c 'umask 077; { echo "-----BEGIN PRIVATE KEY-----"; tr -d "\n " < /etc/ssl/cloudflare/origin.key | sed "s/-----BEGINPRIVATEKEY-----//;s/-----ENDPRIVATEKEY-----//" | fold -w 64; echo; echo "-----END PRIVATE KEY-----"; } > /etc/ssl/cloudflare/origin.key.new && openssl pkey -in /etc/ssl/cloudflare/origin.key.new -noout && mv /etc/ssl/cloudflare/origin.key.new /etc/ssl/cloudflare/origin.key'
```

Comprobar que el certificado cubre tu dominio y que la key le corresponde (debe imprimir `COINCIDEN`):

```bash
sudo openssl x509 -in /etc/ssl/cloudflare/origin.pem -noout -subject -enddate -ext subjectAltName
diff <(sudo openssl x509 -in /etc/ssl/cloudflare/origin.pem -noout -pubkey) <(sudo openssl pkey -in /etc/ssl/cloudflare/origin.key -pubout) && echo COINCIDEN
sudo openssl x509 -in /etc/ssl/cloudflare/origin-pull-ca.pem -noout -subject -enddate
```

La CA de Authenticated Origin Pulls tiene fecha de vencimiento (en 2029 al escribir esto): antes de esa fecha
hay que volver a descargarla, o nginx rechazará a Cloudflare. Conviene dejar un recordatorio en el calendario.

### 4.3 Ajustes globales de nginx

Ver qué trae Debian en `nginx.conf` para no duplicar directivas:

```bash
grep -nE 'ssl_|server_tokens' /etc/nginx/nginx.conf
```

En Debian 13 ya vienen `server_tokens off`, `ssl_protocols TLSv1.2 TLSv1.3` y `ssl_prefer_server_ciphers off`:
**no se repiten** en el archivo siguiente (nginx rechaza directivas duplicadas con `directive is duplicate`).
Si en otra versión `server_tokens` está comentado o `ssl_protocols` incluye TLS 1.0/1.1, corregirlo en `nginx.conf`.

Crear los ajustes globales:

```bash
sudo nano /etc/nginx/conf.d/00-hardening.conf
```

```
# Límites contra clientes lentos o cuerpos enormes
client_max_body_size 20m;
client_body_timeout 15s;
client_header_timeout 15s;
send_timeout 15s;

ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;

# Límite de peticiones por IP real del visitante (segunda capa tras Cloudflare).
# Cada sitio lo activa con: limit_req zone=perip burst=40 nodelay;
limit_req_zone $binary_remote_addr zone=perip:10m rate=20r/s;
limit_req_status 429;

# WebSockets para apps Node/Python
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
```

```bash
sudo nginx -t
```

### 4.4 Fragmentos reutilizables y sitio por defecto

Los fragmentos (`snippets`) evitan repetir lo mismo en cada sitio.

```bash
sudo nano /etc/nginx/snippets/cloudflare-origin.conf
```

```
# Certificado Origin CA + exigir que quien conecta sea Cloudflare (mTLS)
ssl_certificate         /etc/ssl/cloudflare/origin.pem;
ssl_certificate_key     /etc/ssl/cloudflare/origin.key;
ssl_client_certificate  /etc/ssl/cloudflare/origin-pull-ca.pem;
ssl_verify_client on;
```

```bash
sudo nano /etc/nginx/snippets/security-headers.conf
```

```
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-Frame-Options "SAMEORIGIN" always;
```

HSTS no va aquí: se activa en Cloudflare en la Fase 9.

Sitio por defecto: rechaza el handshake TLS de cualquier dominio no configurado (escaneos por IP, dominios ajenos
apuntando a tu servidor):

```bash
sudo nano /etc/nginx/sites-available/00-default.conf
```

```
server {
    listen 443 ssl default_server;
    listen [::]:443 ssl default_server;
    ssl_reject_handshake on;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/00-default.conf /etc/nginx/sites-enabled/00-default.conf
sudo nginx -t
```

### 4.5 IP real del visitante

Detrás de Cloudflare, nginx ve la IP de Cloudflare. La IP real viene en la cabecera `CF-Connecting-IP`, y solo
se puede confiar en ella si la petición viene de un rango de Cloudflare. Se amplía el script de la Fase 3 para
generar esa configuración y mantenerla al día.

```bash
sudo nano /usr/local/sbin/cloudflare-ips
```

Agregar **al final** del archivo:

```bash

# nginx: confiar en CF-Connecting-IP solo si la petición viene de un rango de Cloudflare
if [ -d /etc/nginx/conf.d ]; then
  {
    echo "# Generado por /usr/local/sbin/cloudflare-ips, no editar a mano"
    printf '%s\n' "$v4" "$v6" | sed 's/.*/set_real_ip_from &;/'
    echo "real_ip_header CF-Connecting-IP;"
  } > /etc/nginx/conf.d/cloudflare-realip.conf
  if nginx -t 2>/dev/null; then
    systemctl reload nginx
  else
    echo "nginx: configuración inválida, no se recargó" >&2
  fi
fi
```

Ejecutarlo y comprobar:

```bash
sudo /usr/local/sbin/cloudflare-ips
head -3 /etc/nginx/conf.d/cloudflare-realip.conf
sudo nginx -t
```

### 4.6 Sitio de prueba

Un sitio mínimo para comprobar todo el camino Cloudflare → nginx. Se elimina en la Fase 6, cuando haya sitios reales.

```bash
sudo mkdir -p /var/www/test
echo 'ok' | sudo tee /var/www/test/index.html
sudo nano /etc/nginx/sites-available/test.conf
```

```
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name tudominio.com www.tudominio.com;

    include snippets/cloudflare-origin.conf;
    include snippets/security-headers.conf;
    limit_req zone=perip burst=40 nodelay;

    root /var/www/test;
    index index.html;

    location / { try_files $uri $uri/ =404; }
    location ~ /\. { deny all; }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/test.conf /etc/nginx/sites-enabled/test.conf
sudo nginx -t
sudo systemctl reload nginx
```

**Desde tu computadora**, la petición pasa por Cloudflare:

```bash
curl -I https://tudominio.com
```

Debe responder `HTTP/2 200` con `server: cloudflare`.

**En la VPS**, comprobar que sin ser Cloudflare no se puede hablar con nginx (sin certificado de cliente):

```bash
curl -sk --resolve tudominio.com:443:127.0.0.1 https://tudominio.com -o /dev/null -w '%{http_code}\n'
```

Debe imprimir `400` (nginx rechaza: "No required SSL certificate was sent"). Y un dominio no configurado no completa el handshake:

```bash
curl -sk --resolve otro.invalid:443:127.0.0.1 https://otro.invalid -o /dev/null -w '%{http_code}\n'
```

Debe imprimir `000` (handshake rechazado).

La IP real del visitante debe aparecer en el log, no la de Cloudflare:

```bash
sudo tail -n 3 /var/log/nginx/access.log
```

### 4.7 PHP y Composer

```bash
sudo apt install -y php-fpm php-cli php-mbstring php-xml php-curl php-zip php-bcmath php-intl php-gd \
  php-pgsql php-mysql php-sqlite3 php8.4-opcache composer
```

Endurecer PHP-FPM:

```bash
sudo nano /etc/php/8.4/fpm/conf.d/99-hardening.ini
```

```
expose_php = Off
display_errors = Off
log_errors = On
cgi.fix_pathinfo = 0

memory_limit = 256M
max_execution_time = 60
upload_max_filesize = 20M
post_max_size = 20M

session.cookie_httponly = 1
session.cookie_secure = 1
session.use_strict_mode = 1

opcache.enable = 1
opcache.memory_consumption = 96
opcache.max_accelerated_files = 20000
; El deploy recarga php-fpm, así que PHP no revisa si los archivos cambiaron en cada petición.
; Si se edita código a mano en el servidor: sudo systemctl reload php8.4-fpm
opcache.validate_timestamps = 0
```

```bash
sudo php-fpm8.4 -t
sudo systemctl reload php8.4-fpm
php -v
composer --version
php -m | grep -E 'pdo_pgsql|pdo_mysql|mbstring|intl|gd|zip|bcmath|Zend OPcache'
```

El pool `www` que trae Debian se reemplaza por un pool por aplicación en la Fase 6.

### 4.8 Node.js LTS

Se usa el repositorio oficial de NodeSource (el Node de Debian es una versión vieja). La clave se descarga y
se verifica con `signed-by`, sin ejecutar scripts remotos como root. En el servidor solo se **ejecuta** Node:
las apps se compilan en GitHub Actions (Fase 7), por eso no se instala pnpm ni yarn aquí.

```bash
sudo install -d -m 755 /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_24.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt update
apt-cache policy nodejs | head -4
```

`Candidate` debe ser `24.x` y venir de `deb.nodesource.com`. Instalar:

```bash
sudo apt install -y nodejs
node -v
npm -v
```

### 4.9 Python

```bash
sudo apt install -y python3-venv python3-pip
python3 -V
python3 -m venv /tmp/prueba-venv && /tmp/prueba-venv/bin/pip -V && rm -rf /tmp/prueba-venv
```

Debian impide instalar paquetes con `pip` fuera de un entorno virtual (PEP 668): cada app usa su propio `venv`.
No se instalan compiladores (`build-essential`): si un paquete Python necesita compilar, se prefiere su versión
binaria (por ejemplo `psycopg2-binary`) y solo si hace falta se instalan temporalmente.

### Verificación de la Fase 4

- [ ] `curl -I https://tudominio.com` desde tu computadora → `HTTP/2 200`.
- [ ] En la VPS, sin certificado de cliente → `400`; dominio desconocido → `000`.
- [ ] `sudo tail -n 3 /var/log/nginx/access.log` → tu IP real, no la de Cloudflare.
- [ ] `sudo ss -tlnp` → hacia afuera solo 22 y 443 (nginx en 443; PHP-FPM por socket).
- [ ] `php -v` → 8.4, `composer --version`, `node -v` → v24, `python3 -V` → 3.13.
- [ ] `sudo nginx -t` y `sudo php-fpm8.4 -t` sin errores.

---

## Problemas comunes

### `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!`

Pasa al reinstalar la VPS o al recibir una IP usada antes: el servidor tiene llaves de host nuevas y
`~/.ssh/known_hosts` guarda las viejas. **No borrar la huella sin verificarla**: también es lo que se vería
en un ataque man-in-the-middle.

1. En la consola web del proveedor, ver la huella real del servidor:
   ```bash
   ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
   ```
2. Comparar con la huella que muestra el aviso (`SHA256:...`). Si **no** coincide, no conectarse.
3. Si coincide, borrar la entrada vieja en tu computadora y reconectar aceptando la nueva:
   ```bash
   ssh-keygen -R IP_DE_LA_VPS
   ```
