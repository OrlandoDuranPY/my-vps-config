# VPS Debian 13 — Documentación

Configuración paso a paso de la VPS. El plan general está en [PLAN.md](PLAN.md).
Cada fase registra qué se hizo, los comandos usados y cómo se verificó.

| Fase | Estado |
|------|--------|
| 0. Preparación (llave SSH, proveedor, Cloudflare) | Validada |
| 1. Acceso SSH seguro | Validada |
| 2. Sistema base | Pendiente |
| 3. Firewall y fail2ban | Pendiente |
| 4. Stack web | Pendiente |
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
